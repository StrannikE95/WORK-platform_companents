# Apache Flink 2.2.1 — установка и конфигурирование

Связанный документ (глоссарий, чекпоинты, HA JobManager, почему так): `Apache Flink.md`.

Этот файл — **как поставить и настроить**. Stretch одного Flink (JobManager HA / shuffle TM) на несколько ЦОДов **не делаем**: нет rack-awareness как у Kafka; lease Kubernetes HA лежит в etcd **этого** кластера; межЦОДовый ping для RPC и shuffle неприемлем.

Версии: **Flink 2.2.1**, образ `flink:2.2.1-java17` (Java **17** — рекомендуемый рантайм доки 2.2; Java 21 в 2.2 — experimental). На Kubernetes — **Flink Kubernetes Operator 1.15.0** (матрица: `2.2.x`, не **2.3.0**). Коннектор к шине: **Kafka Connector 5.0.0**.  
Документация: https://nightlies.apache.org/flink/flink-docs-release-2.2/ · оператор: https://flink.apache.org/2026/05/26/apache-flink-kubernetes-operator-1.15.0-release-announcement/

На дату соседнего файла latest самого Flink — **2.3.0**. Анонс оператора **1.15.0** линейку 2.3 **не перечисляет**. Прод на Kubernetes — **2.2.1 + 1.15.0**, не 2.3.0 «потому что новее».

Kafka остаётся шиной (кластер **на ЦОД** + MirrorMaker 2, см. `Apache Kafka.install.md`). Flink — обработка, **не** SoT и **не** замена брокера.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** JobManager HA (Kubernetes HA или ZooKeeper) — **внутри одного ЦОДа**. Другие ЦОДы: **отдельные** FlinkDeployment + replay из **локальной** Kafka (копия через MM2). Это не «три JM в трёх залах одного job».
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`). Оператор — Helm **1.15.0**, Native mode (не Standalone: `--host` с IP пода ломает leader election).
3. Application mode, один `execute()` на приложение, если нужна HA. Multi-`execute()` + HA док **не поддерживает**.
4. Dev — изолированная сеть. Нагрузки нет — **нет** числа TM и размера RocksDB «хватит для прода».
5. Чекпоинты прода — S3-совместимое хранилище. `JobManagerCheckpointStorage` (куча JM) — только стенд.
6. Для 2 ЦОДов: активный Flink в ЦОД-1; в ЦОД-2 — независимый выкат (пассив до failover **или** свой job на своей Kafka). Для 3 ЦОДов — то же в третьем. Два активных job с одним `transactionalIdPrefix` на одном Kafka-кластере — конфликт транзакций.
7. ForSt в 2.2 — experimental, в прод не берём.

Критические пробелы: зачем Flink (какой state/sink); retention Kafka vs интервал чекпоинта; где бакет и его репликация; watermark под лаги ведомств. Совместимость коннектора 5.0.0 с **вашим** Kafka 4.3.1 — прогон на стенде, таблица загрузок тестирует линейку Flink, не «любой брокер».

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** написать job, увидеть UI, прочитать тестовый топик, отладить uid/watermark. **Не** цель: отказ ЦОДа и ёмкость RocksDB.

### Предпосылки

- Docker Engine **или** Dev-Kubernetes.
- Порт 8081 (REST/UI JobManager) свободен на localhost.
- Java 17, если собираете job-jar без Docker.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name flink-jm \
  -p 127.0.0.1:8081:8081 \
  flink:2.2.1-java17 jobmanager
```

Привязка к `127.0.0.1` обязательна: `-p 8081:8081` без адреса часто слушает все интерфейсы. UI — `http://127.0.0.1:8081`.

TaskManager в той же Docker-сети, `jobmanager.rpc.address` = имя контейнера JM (не копировать устаревшие хосты из чужих compose). Чекпоинт на стенде — куча JM или `file:///tmp/flink-cp` — **только** здесь.

Проверка: UI открывается, в about/версии — **2.2.1**. Job-jar с коннектором **5.0.0**, не `FlinkKafkaConsumer` из старых гайдов.

### Установка (Kubernetes Dev, если стенд уже в K8s)

Оператор **1.15.0** из https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.15.0/ (Helm-чарт тега 1.15.0, не `latest`, не nightlies `operator-docs-main`).

Минимальный `FlinkDeployment`: 1 JM, 1 TM, parallelism 1, `upgradeMode: stateless`, пока граф прыгает. Образ **2.2.1-java17**. Этот YAML **не** в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| 1 JM / 1 TM | да | Некому строить HA |
| HA / S3 | выкл | Иначе storageDir раньше DataStream API |
| State backend | HashMap (дефолт) | State крошечный |
| REST TLS | нет | Сеть закрыта |
| Session cluster | можно | Несколько учебных job |
| Kafka guarantee | можно NONE на игрушечном топике | **В коде** лучше сразу тот, что в проде |

Чего **не** упрощать: Flink **2.2.1**, коннектор **5.0.0**, Java **17**; **uid** stateful-операторов; понимание, что без checkpointing дефолтный restart = **none** (док: state не восстановится).

### Проверка Dev

1. Версия 2.2.1, UI только с localhost.
2. Submit job, map, cancel.
3. Рестарт JM без checkpoint storage — state потерян: это ожидаемо, не баг стенда.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный образ | Нет HA, нет shuffle между узлами, нет S3 |
| Single JM официален для development | Успешный map ≠ RocksDB-чекпоинт в интервал |
| | PLAINTEXT 8081 приучает «без учётки» |

Препрод: оператор, application mode, HA + S3, RocksDB, 2 TM, прогон kill TM — **в одном ЦОДе**.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **TM или JM внутри ЦОДа** (чекпоинт + новый лидер JM); пережить отказ **целого ЦОДа** ценой RPO>0: поднять **другой** Flink на другой площадке и replay из Kafka этой площадки (MM2). Цифр TM нет.

### Почему не stretch

Чекпоинт и shuffle — не RF=3 на каждое событие. JobManager HA хранит указатели в `high-availability.storageDir` + lease/ZK **этого** ЦОДа. Разнести JM по ЦОДам = Raft/lease по городу (у вас уже запрещён stretch etcd) и RPC с чужим RTT. Официального «stretch Flink на 3 DC» в доке **нет**. Порог RTT проектом **не задан** — поэтому не подставляем.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один логический job (лучше application cluster на пайплайн):

- JobManager HA: `high-availability.type: kubernetes` (типичный выбор оператора) **или** ZooKeeper, оба — **только ноды этого ЦОДа**;
- `spec.jobManager.replicas` ≥ 2 (standby **ускоряет** смену лидера; job **всё равно рестартится** — формулировка оператора);
- TM: несколько подов, anti-affinity **внутри** ЦОДа, **не** topologySpread «по чужим ЦОДам для галочки»;
- checkpoint + HA metadata — filesystem (`s3p://` плагин Presto, если так собираете образ);
- образ: база `flink:2.2.1-java17` + job-jar + `opt/flink-s3-fs-presto-2.2.1.jar` → `plugins/s3-fs-presto/` + коннектор **5.0.0**;
- Kafka bootstrap — **локальный** кластер ЦОД-1 (`Apache Kafka.install.md`).

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: активный Flink. ЦОД-2: **отдельный** оператор + Deployment (тёплый standby или свой job на Kafka ЦОД-2, куда MM2 копирует топики) | Обработка в ЦОД-1 остановилась. Replay на ЦОД-2 с чекпоинта/savepoint **или** с offset локальной Kafka. RPO > 0. Дубли возможны |
| **3 ЦОДа** | Третий независимый экземпляр по той же схеме | То же; третий Flink не делает «RF=3 state» |

Ровно-один раз обработать общекластерный факт «само» при двух активных Flink **нельзя** (`Apache Flink.md`, вариант C): либо двойная обработка, либо active-passive и runbook. `transactionalIdPrefix` уникален среди job на **одном** Kafka-кластере; после failover старый job должен быть мёртв.

Оператор: `replicas: 2` **только** с leader election (иначе два активных контроллера). Падение оператора **не** роняет уже бегущие job, но не вылечит пропавший JM.

### Предпосылки прода

- CSI; RocksDB TM — локальный SSD, не NFS как рабочий каталог. Чекпоинт **не** единственная копия на RWO одной TM.
- Object storage, куда пишут JM **и** TM. Бакет только в ЦОД-1 = restore в ЦОД-2 упирается в этот бакет.
- NetworkPolicy: 8081 — прокси/админы/оператор, не интернет; RPC/blob/data — только JM↔TM; к Kafka и S3 — по документам Kafka/хранилища.
- Секреты MinIO/Kafka — Secret/CSI, **не** поля CR: оператор логирует diff (FLINK-30306).

### Установка оператора (на каждом Kubernetes, где будет Flink)

Helm-чарт **1.15.0** с архива Apache (каталог `flink-kubernetes-operator-1.15.0`). Webhook — включён осознанно. Leader election включить **до** replicas > 1 (миграция с 1 реплики — через scale-to-zero, иначе два активных).

Свой образ собрать **до** первого боевого job: без плагина S3 чекпоинт в бакет не уедет.

### Конфигурирование активного job (ЦОД-1)

Смысл CR (не полный — сверять с докой оператора 1.15.0):

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: proj-job
spec:
  image: <registry>/flink-job:2.2.1-java17
  flinkConfiguration:
    high-availability.type: kubernetes
    high-availability.storageDir: s3p://<bucket>/flink/ha/...
    execution.checkpointing.dir: s3p://<bucket>/flink/checkpoints/...
    # interval чекпоинта — замером: duration < interval с запасом; не копировать 10 с из туториала
    # backend: EmbeddedRocksDB (ключ в доке 2.2, не ForSt)
    # process.size TM — размер пода; не задавать вместе с flink.size (док: конфликт)
  jobManager:
    replicas: 2
  # taskManager / parallelism — после замера; upgradeMode savepoint или last-state; file:// HA — не прод
```

`file:///flink-data/ha` из примеров оператора — том пода, **не** HA на отказ ЦОДа.

Обязательно рядом:

1. Checkpointing **включён**; storage filesystem, не куча JM. Externalized checkpoints RETAIN, пока нет политики savepoint.
2. RocksDB + incremental, пока не доказали что state влезает в heap. Unaligned по умолчанию **не** включать (ломает произвольный upgrade).
3. Restart: exponential-delay при включённом CP (док рекомендует против лавины на Kafka).
4. KafkaSink: не оставлять дефолт `DeliveryGuarantee.NONE` на пайплайне в озеро. Exactly-once: уникальный prefix, `transaction.timeout.ms` брокера **больше**, чем max(чекпоинт)+max(рестарт); потребители `read_committed`. Parallelism source ≤ числа партиций.
5. REST 8081 не с мира; `security.ssl.rest.enabled` + sidecar/SSO (док: REST сам клиента не аутентифицирует). Internal SSL — отдельные флаги RPC/blob/data.
6. PDB: не снять все TM сразу. Память пода = `taskmanager.memory.process.size` (+ запас sidecar), не «только -Xmx».

ZooKeeper HA допустим, если ZK уже есть **в этом ЦОДе**. Это ещё один кворум; на K8s типичен Kubernetes HA. ZK между ЦОДами — тот же запрет stretch.

### Масштабирование (когда появятся цифры)

1. Замерить lag Kafka, duration/size чекпоинта, backpressure, heap TM, диск RocksDB, restarts.
2. Parallelism ↔ слоты (TM × `numberOfTaskSlots`) ↔ партиции Kafka. Пустые TM без слотов job не ускоряют.
3. Сначала партиции Kafka, потом Flink. Rescale stateful — через savepoint/aligned checkpoint.
4. Autoscaler оператора — **после** стабильных чекпоинтов, не в день первого деплоя.
5. JM почти не масштабируют «от MB/s». Много session-job на одном JM — антипаттерн прода.

«Терабайты озера» ≠ терабайты state. State — окна, join, offsets. Restore RocksDB с большого снимка может быть долгим: RTO **замерить**, формул минут в доке нет.

### Проверка прода (пока это не пройдено — это не прод)

Внутри ЦОДа:

1. Образ 2.2.1-java17, оператор 1.15.0, коннектор 5.0.0. Не 2.3.0.
2. Дождаться **успешного** чекпоинта → убить TM → job встал с того же offset/state.
3. Убить лидера JM → standby/новый лидер, job restart, storageDir жив.
4. 8081 снаружи — отказ. Секреты не в тексте CR.

Между ЦОДами:

5. MM2 догоняет тестовый топик в Kafka ЦОД-2.
6. Учения: остановить Flink ЦОД-1, поднять Deployment ЦОД-2 по runbook (тот же образ, не два живых exactly-once sink с одним prefix), replay. Замерить RTO/RPO.

### Сильные / слабые стороны (HA внутри ЦОДа + отдельный Flink + Kafka/MM2)

| Сильное | Слабое |
|---|---|
| Shuffle и lease локальные | Падение ЦОДа Flink = простой обработки, пока replay |
| Согласовано с запретом stretch и с Kafka per-DC | RPO ≠ 0; дубли; ручной failover |
| Kubernetes HA без обязательного ZK | Зависимость от etcd **этого** кластера и от бакета |
| | Два активных job без дисциплины = двойная обработка |

**Не готов к проду**, если: checkpointing выкл; storage в куче JM; `DeliveryGuarantee.NONE` в озеро; HashMap при немереном state; demo `file://` HA; 8081 снаружи; оператор `replicas: 2` без election; ForSt в бою; **Flink 2.3.0 на операторе 1.15.0**; JM HA «на три ЦОДа в уме»; Flink назначен SoT.

---

## Источники

- Flink 2.2.1: https://flink.apache.org/2026/05/15/apache-flink-2.2.1-release-announcement/
- Оператор 1.15.0, матрица 2.2.x: https://flink.apache.org/2026/05/26/apache-flink-kubernetes-operator-1.15.0-release-announcement/
- HA Overview: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/ha/overview/
- Checkpoints: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/state/checkpoints/
- Kafka connector 5.0.0: https://flink.apache.org/downloads/ и https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/connectors/datastream/kafka/
- Helm 1.15.0: https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.15.0/
- Образ: https://hub.docker.com/_/flink
- Правила и пробелы: `Apache Flink.md`

Утверждений «N мс — можно размазать TM» в документации **нет**. Stretch JM HA в этой инструкции не предлагается.
