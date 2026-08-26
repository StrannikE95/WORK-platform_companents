# RabbitMQ 4.3.5 — установка и конфигурирование

Связанный документ (глоссарий, Khepri/Raft, quorum queues, почему так): `RabbitMQ.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного кластера на несколько ЦОДов **не делаем**: Khepri (метаданные) и quorum queues — распределённая база на Raft, чувствительная к партициям; межЦОДовый ping для sync-кворума неприемлем.

Версии: **RabbitMQ 4.3.5**, Erlang **27.x** (в официальном образе уже есть; свой Erlang младше 27.0 — нода не стартует). Образ: `rabbitmq:4.3.5-management`. На Kubernetes — **Cluster Operator v2.22.5**; в `spec.image` пинить **4.3.5-management** (дефолт оператора на дату соседнего файла — **4.3.4**).  
Документация: https://www.rabbitmq.com/docs/ · оператор: https://www.rabbitmq.com/kubernetes/operator/operator-overview

Kafka остаётся шиной событий. RabbitMQ — очереди задач / AMQP, **не** замена Kafka и **не** озеро клиентских данных.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Один кластер RabbitMQ живёт **внутри одного ЦОДа**. Между ЦОДами — отдельные кластеры + **Shovel или Federation** (асинхронно) либо тёплый standby. Это вариант B из `RabbitMQ.md`, не «по ноде в каждый ЦОД в одном Raft».
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Dev — изолированная сеть; пароль в примере не секрет.
4. Нагрузки нет — поэтому **нет** цифры CPU/RAM/диска «хватит для прода». Есть минимум, чтобы процесс жил (кворум из 3 нод **внутри** площадки), и рычаги роста.
5. Берём community RabbitMQ 4.3.5 (MPL-2.0), не Tanzu. Erlang **28** официально только для **новых** кластеров; rolling 27→28 на живом Khepri имеет известную проблему — на образе 4.3.5 остаёмся на **27.x**.
6. В контуре **есть** роль AMQP/очередей задач. Если все потоки уже на Kafka — этот брокер можно не ставить.
7. Для 2 ЦОДов: активный кластер в ЦОД-1, в ЦОД-2 — независимый кластер + Shovel/Federation **или** тёплый standby. Для 3 ЦОДов: то же + третья площадка с таким же независимым экземпляром (или только standby). Третий ЦОД **не** добавляет третий writer в тот же Raft.

Критические пробелы (закрыть до боя, не «потом»): нужен ли брокер рядом с Kafka; профиль очередей (сообщ./с, размер, prefetch); срок хранения backlog; формальный RPO при обрыве Shovel; PKI. Официальная таблица RTT для **кластеризации** — в `RabbitMQ.md`; она объясняет, почему stretch не берём, а не даёт «зелёный свет» на ваш канал.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** публиковать и потреблять, отладить exchange/queue/binding, контракт confirm/ack. **Не** цель: отказ площадки и тысячи quorum queues.

### Предпосылки

- Docker Engine (стенд разработчика) **или** любой однонодовый Kubernetes.
- Порты 5672 (AMQP) и 15672 (management UI) свободны на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name rmq-dev \
  -e RABBITMQ_DEFAULT_USER=app \
  -e RABBITMQ_DEFAULT_PASS=dev \
  -p 127.0.0.1:5672:5672 \
  -p 127.0.0.1:15672:15672 \
  rabbitmq:4.3.5-management
```

Привязка к `127.0.0.1` обязательна: `-p 5672:5672` без адреса часто слушает все интерфейсы. Порты **4369** (epmd) и **25672** (Erlang distribution) **не** публиковать даже на Dev.

Проверка:

```bash
docker exec -it rmq-dev rabbitmq-diagnostics status
docker exec -it rmq-dev rabbitmqctl version
```

В выводе должна быть **4.3.5**. Management UI — `http://127.0.0.1:15672`, пользователь `app`, не `guest`/`guest`.

Клиент AMQP — на `localhost:5672`, vhost `/`, пользователь `app`. Publisher confirms и manual ack в **коде** лучше включить сразу: стенд иначе врёт про прод.

### Установка (Kubernetes Dev, если стенд уже в K8s)

```bash
# Cluster Operator v2.22.5 — манифест с GitHub Releases проекта rabbitmq/cluster-operator
# (файл cluster-operator.yml тега v2.22.5, не latest)
kubectl apply -f <манифест-v2.22.5>
kubectl -n rabbitmq-system rollout status deployment/rabbitmq-cluster-operator
```

Минимальный кластер (не прод-манифест):

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rmq-dev
spec:
  replicas: 1
  image: rabbitmq:4.3.5-management
  persistence:
    storage: 10Gi
```

Сервис AMQP: `rmq-dev:5672`. Образ пинить на 4.3.5 явно, не дефолт оператора 4.3.4.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Одна нода | да | Некому строить кворум Khepri |
| TLS | нет | Иначе PKI раньше routing key |
| Classic queues | допустимы | Некому реплицировать; на препроде — quorum |
| Management 15672 | да, только localhost | Отладка |
| Shovel/Federation | не нужны | Один ЦОД |
| `guest` с паролем `guest` | **запрещён** даже на Dev | Привычка уедет в прод |

Чего **не** упрощать: версия 4.3.5; отдельный пользователь не `guest`; confirms + manual ack в клиенте; имена exchange/queue/vhost, не «как в туториале».

### Проверка Dev

1. Версия брокера = 4.3.5, Erlang 27.x.
2. Publish/consume через пользователя `app`; `guest` с сети — отказ.
3. Рестарт контейнера: durable-очередь на volume жива (если volume был). Classic без диска — потеря ожидаема.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный образ | Нет failover, нет Raft, нет Shovel |
| Совпадает с development-практикой RabbitMQ | Успешный publish **не** доказывает прод |
| Дешёво гоняет контракт AMQP | Открытый 5672 на Wi-Fi = дыра |

Препрод: **3 ноды + quorum + TLS в одном ЦОДе**, даже без боевой нагрузки.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой RPO>0 и асинхронного моста / ручного переключения клиентов. Цифр сообщ./с нет — ниже минимум HA внутри площадки, не смета железа.

### Почему не stretch

Confirm quorum queue не быстрее, чем запись majority реплик + диск. При неприемлемом RTT между ЦОДами Raft даёт ложные выборы лидера и «глухой» Khepri, а не защиту. Документация 4.3: при p99 **> 100 мс** или потерях пакетов кластер **не рекомендуют** — отдельные кластеры + Shovel/Federation. Платформа stretch не берёт и при «в среднем нормальном» ping: партиция канала между ЦОДами для распределённой БД метаданных — штатный риск, не теория.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один кластер:

- **3 ноды** (нечётное; majority Khepri = 2) на **разных нодах Kubernetes** одного ЦОДа;
- `default_queue_type = quorum`; classic как HA в 4.x **не работает** (mirroring удалён с 4.0);
- `quorum_queue.initial_cluster_size = 3`; реплики **внутри** этого ЦОДа, не «по городу»;
- клиенты AMQP могут на **любую** ноду (в отличие от Kafka: лидер партиции); LB перед 5671 допустим;
- PVC на ноду, **Parallel** pod management (не `OrderedReady`: rolling restart официально упирается в deadlock);
- образ **4.3.5-management**, не 4.3.4 и не `latest`.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: активный кластер из 3 нод. ЦОД-2: **отдельный** кластер + Shovel/Federation **или** тёплый standby (копия схемы, без боевых потребителей) | Локальный confirm в ЦОД-1 умер. Доставка в ЦОД-2 асинхронная: **RPO > 0**, возможны дубли. Переключение клиентов — **runbook**, не общий кворум |
| **3 ЦОДа** | ЦОД-1: активный. ЦОД-2 и ЦОД-3: независимые кластеры + мост **или** standby | То же; третий кластер не делает «RF=3 между городами» |

Клиенты микросервисов в штате пишут в **локальный** кластер своего ЦОДа (если площадка активна для этих очередей) **или** только в ЦОД-1, а остальные — DR. «Пишем одну quorum queue во все три ЦОДа одним Raft» — это stretch, его нет.

Shovel/Federation — **плагины на конкретных нодах**; падение той ноды — отдельный failover моста, его надо закладывать (не один link «на кластер сам»). Идемпотентность воркера — на вас.

Несколько **разных** кластеров по контурам (задачи интеграционного API ≠ внутренний RPC) — отдельные HA, не один брокер «на всех». Кластер из десятков нод проект считает поводом **разрезать**, не «добавить ноды».

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не один stretch-кластер).
- CSI с local/зональным диском; том **не делится** между нодами. Distributed FS как общий каталог — порча данных.
- NetworkPolicy: 5671 клиентам; 4369/25672/6000–6500 **только** между нодами и jump CLI; epmd не в интернет.
- Erlang cookie — Secret/Vault, права файла **600**, не в Git и не дефолт образа.
- Секреты пользователей — не в Git. Definition import / Topology Operator — схема as code.

### Установка оператора (на каждом Kubernetes, где будет RabbitMQ)

Cluster Operator **v2.22.5** из релизов `rabbitmq/cluster-operator`. Оператор — Deployment, **не** в одном поде с брокером. Topology Operator — отдельно, если очереди/users декларативно.

### Конфигурирование активного кластера (ЦОД-1)

Смысл манифеста (не полный CR — сверять с докой оператора v2.22.5):

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rmq-tasks
spec:
  replicas: 3
  image: rabbitmq:4.3.5-management
  persistence:
    storage: 50Gi   # заглушка: реальный размер = очереди + Raft WAL; цифр нагрузки нет
  rabbitmq:
    additionalConfig: |
      default_queue_type = quorum
      vm_memory_high_watermark.relative = 0.6
      disk_free_limit.absolute = 2GB
      anonymous_login_user = none
  # anti-affinity по ноде обязательна; не оставлять 2 из 3 в одном зале
  # peer discovery задаёт Operator — не ручной join_cluster
```

`disk_free_limit` дефолт **50 МБ** — для туториалов. Официальный ориентир чеклиста: порядка memory watermark, не 50 МБ. Цифра `2GB` выше — не «хватит для прода», а «не оставить дефолт 50 МБ»; считать от **лимита cgroup**, иначе брокер увидит RAM ноды.

Обязательно рядом:

1. Удалить `guest`. Не `loopback_users.guest = false` (это антипаттерн чеклиста). Отдельный пользователь на каждое приложение, fine-grained permissions на vhost.
2. Клиенты — **5671 + TLS**. Management — **15671**, не 15672 с корпоративного «почти интернета». Inter-node TLS — если трафик нод выходит из доверенного сегмента **этого** ЦОДа.
3. Политики: delivery-limit (дефолт 20) + DLX; не `-1` «чтобы не мешало» без расчёта диска.
4. Peer discovery — автоматический (K8s plugin / Operator). Ручной `join_cluster` официально хрупкий для прода.
5. `net_ticktime` **не понижать** (дефолт 60 с). На WAN ложные разрывы хуже.
6. Клиенты: publisher confirms, durable + persistent, manual ack, prefetch по вместимости воркера, auto-recovery. `basic.get` в цикле официально неэффективен.
7. Мост в ЦОД-2: плагин Shovel **или** Federation, HA самого link, TLS. Promote/переключение продюсеров — **ручной/GitOps**. Два активных потребителя одной логической задачи на двух кластерах без идемпотентности = двойная работа.

`pause_minority` / `autoheal` из гайдов 3.x **не применяются**: Mnesia удалена в 4.3. Восстановление — семантика Raft.

### Масштабирование (когда появятся цифры)

1. Замерить сообщ./с, размер, число quorum queues, соединений, confirm latency, alarms памяти/диска.
2. Сначала **вертикаль** (CPU/RAM/диск, не колокация с БД). Ещё нода не ускоряет одну очередь: единица параллелизма — **потребители на этой очереди** + CPU лидера.
3. Добавлять ноды **нечётными** шагами **внутри ЦОДа**, затем `rabbitmq-queues grow` / placement. Ориентир проекта «пересмотри топологию» — порядка **5000** quorum queues (не жёсткий лимит кода).
4. Тяжёлые события с долгим логом — Kafka, не «ещё один fan-out exchange».
5. TLS бьёт по CPU — чеклист; «включили и осталась та же скорость» — неверно.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия 4.3.5 / Erlang 27.x на всех нодах ЦОД-1; `cluster_status` — 3 ноды.
2. Очередь без `x-queue-type` создаётся **quorum**, не classic.
3. Убить под одной ноды: majority жив, confirm проходит, клиенты переподключились. `check_if_node_is_quorum_critical` **до** drain.
4. Publish без ACL / аноним — отказ. `guest` нет.
5. Shovel/Federation догоняет тестовую очередь в ЦОД-2; замерить лаг (это ваш RPO).
6. Учения: остановить ЦОД-1, переключить клиентов на ЦОД-2 **по runbook**.

### Сильные / слабые стороны прод-схемы (кластер в одном ЦОДе + async мост)

| Сильное | Слабое |
|---|---|
| Confirm не зависит от межЦОДового RTT | Падение ЦОД-1 = нет локальной записи, пока переключение |
| Согласовано с «не stretch» и с Raft Khepri | RPO между ЦОДами > 0; дубли; идемпотентность на вас |
| Один кворум на площадку, понятная модель | Link Shovel живёт на ноде; отдельный HA плагина |
| | Два–три кластера: cookie, PKI, definitions — GitOps или разъедутся |

**Не готов к проду**, если: `replicas: 1`, образ 4.3.4/`latest`, classic queues как HA, mirroring из мануала 3.x, `guest` снаружи, PLAINTEXT на общей сети, emptyDir, `OrderedReady` + `check_running` как единственный probe, один кластер «на три ЦОДа в уме», RabbitMQ объявлен «вместо Kafka», мост из одного пода без SLA.

---

## Источники

- Релиз 4.3.5: https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.3.5
- Матрица Erlang: https://www.rabbitmq.com/docs/which-erlang
- Clustering, таблица RTT, Shovel/Federation: https://www.rabbitmq.com/docs/clustering
- Partitions / Raft: https://www.rabbitmq.com/docs/partitions
- Quorum queues, mirroring удалён: https://www.rabbitmq.com/docs/quorum-queues
- Production checklist: https://www.rabbitmq.com/docs/production-checklist
- Cluster Operator: https://github.com/rabbitmq/cluster-operator
- Образ: https://hub.docker.com/_/rabbitmq
- Правила и пробелы: `RabbitMQ.md`

Порога «хватит 3 нод на нашу нагрузку» в документации **нет**. Stretch в этой инструкции не предлагается.
