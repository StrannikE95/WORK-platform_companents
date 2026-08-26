# Wazuh 4.14.7 — установка и конфигурирование

Связанный документ (глоссарий, master/worker, indexer vs server, секреты overlay, почему так): `Wazuh.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch indexer/server-кластера на несколько ЦОДов **не делаем**: порты **9300** (состояние кластера indexer) и **1516** (синхронизация master↔worker) не разносим между площадками; межЦОДовый ping для такого контура запрещён. Wazuh indexer — OpenSearch-based движок **этого** продукта, **не** платформенный кластер OpenSearch 3.8.

Версии: **Wazuh 4.14.7**. Образы: `wazuh/wazuh-manager:4.14.7`, `wazuh/wazuh-indexer:4.14.7`, `wazuh/wazuh-dashboard:4.14.7`, `wazuh/wazuh-agent:4.14.7`.  
Официальный путь на Kubernetes: репозиторий **`wazuh/wazuh-kubernetes`**, тег **`v4.14.7`**, Kustomize (`envs/local-env` или `envs/eks`). Отдельного Helm-оператора у вендора в этом гайде нет.  
Документация: https://documentation.wazuh.com/current/ · K8s: https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-deployment.html

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** HA indexer и server — **внутри одного ЦОДа** (одного Kubernetes). Между ЦОДами — агенты смотрят на manager домашнего ЦОДа (или локальный manager, если сеть изолирована — тогда это уже **другой** SIEM).
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Self-hosted OSS 4.14.7, не Wazuh Cloud.
4. Dev — изолированная сеть; пароли overlay в примере известны всему GitHub — на тесте допустимы **только** в песочнице.
5. Нагрузки нет — нет числа worker и терабайт PVC «хватит для прода». Есть сигналы `events_dropped` / `discarded_count` и рычаги.
6. Три изолированных Wazuh **без** склейки событий = **три SIEM** (три индекса, три правды для SOC). Так не закрыть единый разбор.
7. Для 2 ЦОДов: мозг в ЦОД-1, агенты ЦОД-2 → VIP manager ЦОД-1. Для 3 ЦОДов: то же + агенты ЦОД-3. Третий ЦОД **не** добавляет второго master.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** агент → алерт → документ в индексе → UI. Понять шум штатных правил на ваших образах. **Не** цель: отказ площадки и поток auditd с Kafka.

### Предпосылки

- Linux x86_64/ARM64 **или** Dev-Kubernetes с CSI (PVC).
- Порты 1514/1515/443 не обязаны торчать с ноутбука в Wi-Fi.
- Сеть стенда закрыта.

Два официальных пути — **выберите один**, не оба сразу.

### Установка (путь A — all-in-one Linux)

Быстрее понять продукт. Server + indexer + dashboard на **одной** машине. Вендор: обычно до ~100 endpoints и 90 дней индекса — это не ваш прод.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

После установки **отключить репозиторий пакетов**, чтобы случайный `yum update` не разъехал версии (рекомендация quickstart). Образы/пакеты линии **4.14.7**.

### Установка (путь B — Kubernetes `envs/local-env`)

Ближе к прод-оркестратору. Не копировать полный EKS overlay на ноутбук (indexer 3 / worker 2 — это уже не «тест без нагрузки»).

```bash
git clone -b v4.14.7 --depth 1 https://github.com/wazuh/wazuh-kubernetes.git
cd wazuh-kubernetes
# сертификаты скриптом репозитория (self-signed — только стенд)
kubectl apply -k envs/local-env
```

Ожидаемо в local-env: indexer **1**, master **1**, worker **1**, dashboard **1**. StorageClass — ваш. Образы проверить: тег **4.14.7**, не `main`.

Порядок зависимости тот же, что в installation guide, даже если Kustomize применяет манифесты пакетом: **indexer → server → dashboard**. Пока indexer не принимает 9200, Filebeat на manager некуда писать.

### Конфигурирование Dev

| Параметр | На тесте | Почему можно |
|---|---|---|
| Replica indexer | 0 при одном узле | Иначе yellow. Вендор: на single-node replicas **ставить 0** |
| Worker'ы | 1 | Некому балансировать |
| PKI | self-signed | Иначе сертификаты раньше правил |
| Пароли overlay | дефолт **только в изоляции** | В чек-лист прода не переносить |
| FIM/SCA | минимум путей | Сначала увидеть шум |
| CTI / VD | можно выкл | Часто нет интернета со стенда |
| Archives (все события) | не включать «из любопытства» | Диск и привычка в прод |

Чего **не** упрощать: версия **4.14.7** на всех четырёх ролях; хотя бы один агент, который шлёт не пусто; помнить, что `ossec.conf` master **не** синхронизируется на worker автоматически.

Агент: пакет на VM **или** DaemonSet из гайда 4.14.7. Агент в контейнере **без** hostPath/`/var/log`/`/proc` хост не мониторит — это декорация.

### Проверка Dev

1. Версии manager/indexer/dashboard/agent = **4.14.7**.
2. Контролируемое событие (файл из FIM или лог) → документ в `wazuh-alerts-*` → видно в dashboard.
3. Рестарт единственного indexer: на local-env без replica поиск дырявится — это ожидаемо, не баг стенда.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный quickstart / local-env | Нет LB, нет replica, нет отказа ЦОДа |
| Дешево показывает шум на *ваших* Java/Kafka логах | Успешный алерт на простой ноде ≠ analysisd под нагрузкой брокера |
| | Дефолт `password` / `SecretPassword` приучают открыть UI |

Препрод: 3 indexer, 1 master + ≥2 worker, свои секреты, replica шардов ≥ 1, ISM, агент на tainted-нодах — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **ноды indexer/worker внутри домашнего ЦОДа**; пережить отказ **чужого ЦОДа** (агенты той площадки молчат, мозг жив); отказ **домашнего ЦОДа** = SIEM нет, пока restore/перенос с PVC. Цифр APS нет.

### Почему не stretch

Indexer cluster state идёт по **9300**; server-кластер — **1516**. Документация Wazuh **не задаёт** порог RTT, но stretch на 2–3 ЦОДа при запрете ping для кворумных/cluster-протоколов — лотерея yellow/split, не HA. Один Kubernetes на три площадки у вас тоже не предполагается (см. `Kubernetes.install.md`).

Три полных Wazuh «по кластеру как Falco» **без** склейки событий — три SIEM. Falco — агент на ноде. Wazuh server/indexer — **общая** система.

### Топология

**В домашнем ЦОДе (ЦОД-1)** — один логический Wazuh:

- **Indexer:** ≥ 3 ноды StatefulSet, все в этом ЦОДе; шаблон `wazuh-alerts-4.x-*`: **не** дефолт 3 primary / **0 replica**. Для HA — **хотя бы 1 replica**. PVC RWO, не NFS «один том на троих»;
- **Server:** ровно **1 master** + **≥ 2 worker** в этом ЦОДе. Второго master штатно нельзя;
- **Dashboard:** Deployment, в проде **≥ 2** реплики (вендорский YAML даёт 1 — SPOF просмотра, не детекта);
- перед **1514** (и обычно **1515**) — **TCP LB** этой площадки. Агенты знают VIP, не список подов;
- `vm.max_map_count ≥ 262144` на нодах indexer;
- Filebeat (в образе manager) → indexer **этого** же кластера на 9200. Не в платформенный OpenSearch 3.8.

**Между ЦОДами — агенты, не cluster-порты:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: indexer+master+worker+dashboard. ЦОД-2: агенты (DaemonSet/пакет) → VIP 1514/1515 ЦОД-1 | SIEM нет для обеих площадок, пока restore мозга. Агенты ЦОД-2 копят очередь, бесконечный буфер вендор не обещает |
| **3 ЦОДа** | То же + агенты ЦОД-3 на тот же VIP | То же; третий ЦОД не даёт второго master |

Если ЦОД-2 **изолирован** и 1514 до ЦОД-1 нет — ставите локальный manager+indexer. Это **второй SIEM**: события сами не сольются. SOC должен это принять явно, не «потом склеим».

Клиенты dashboard — только внутренняя сеть. Не LoadBalancer из примера в интернет.

### Предпосылки прода

- Kubernetes ЦОД-1 с CSI; ноды indexer — диск с нормальным I/O (медленный диск = очередь Filebeat = дырки в SIEM).
- Свои сертификаты (скрипт `generate_certs.sh` — self-signed для теста). Свои секреты **до** первого агента. Overlay v4.14.7 кладёт в git известные значения (`wazuh-authd-pass`, `indexer-cred`, `dashboard-cred`, `wazuh-api-cred`, `wazuh-cluster-key`) — в прод не оставлять (перечень и смысл: `Wazuh.md`).
- NetworkPolicy: 9200/9300/1516/55000 не с мира; 1514/1515 только от агентов/LB; **1516** только между узлами server.
- Зеркало образов `wazuh/*:4.14.7`, если закрытый контур. CTI/Vulnerability Detection без исходящего доступа к Wazuh CTI — не обещать «сканер CVE из коробки».
- Бэкап PVC indexer **и** `/var/ossec` master (ключи агентов). Потеря ключей = пере-enrollment флота.

### Установка (Kustomize v4.14.7, домашний ЦОД)

Порядок логический: **indexer → server → dashboard**. Overlay применяет сразу; смотреть, что indexer green *до* боевого Filebeat.

```bash
git clone -b v4.14.7 --depth 1 https://github.com/wazuh/wazuh-kubernetes.git
# 1) заменить все secrets, 2) свои cert, 3) StorageClass, 4) ресурсы не из EKS-патча 1–2Gi
kubectl apply -k envs/eks   # или свой overlay от eks, не local-env
```

Базовый полный overlay: indexer **3** / master **1** / worker **2**. Это старт HA **внутри** ЦОДа, не смета APS. Ресурсы `envs/eks` (500m–1 CPU, 1–2 Gi, 10 Gi PVC) — «чтобы стартануло», **не** recommended hardware (indexer ориентир вендора 16 ГиБ / 8 CPU на узел).

Дальше обязательно:

1. Шаблон индексов: replica ≥ 1. ISM (пример вендора 90d — ваш срок = решение ИБ) **до** боевого потока.
2. Cluster key 32 символа с энтропией; `authd` не `password`; enrollment лучше с проверкой сертификата агента, 1515 не в интернет.
3. `ossec.conf` worker копировать GitOps'ом после правок на master (синхронизируются ключи/кастомные rules, **не** весь `ossec.conf`).
4. LB :1514 leastconn (HAProxy helper — опция **на master**, сам HAProxy — отдельное ПО).
5. Агенты: DaemonSet 4.14.7 на **каждом** Linux-кластере, `spec.nodeName` как имя агента, **tolerations** на Kafka/Camunda/control-plane. Windows/VM — пакет. FIM не вешать на data-dir брокера «всё подряд».
6. Active Response **выкл** на интеграционном контуре.
7. После установки — выключить автообновление репозитория Wazuh.

### Конфигурирование (смысл)

| Параметр | Прод | Зачем |
|---|---|---|
| Indexer replicas шардов | ≥ 1 | Дефолт шаблона 0 replica — не HA |
| Server master | ровно 1, в ЦОД-1 | Второго штатно нет |
| Worker | ≥ 2, потом по `events_dropped=0` | Приём событий |
| Dashboard | ≥ 2, не в интернет | Просмотр; детект живёт на server/indexer |
| JVM indexer | Xms=Xmx, ориентир половина RAM | Overlay 1g/2Gi — стенд |
| Archives | выкл, пока нет отдельной ёмкости | Иначе диск съест «терабайты» сам |
| Pin образов | 4.14.7 | Не смешивать 4.13 indexer с 4.14 manager |

### Масштабирование (когда появятся цифры)

1. Замерить APS, `events_dropped`, `discarded_count`, heap, диск, yellow/red.
2. Больше событий → worker'ы, не «ещё один master».
3. Больше хранения/поиска → ноды indexer + ISM, не безразмерный PVC.
4. Больше K8s-нод → DaemonSet агентов растёт сам; на мозг давит **поток**, не «число микросервисов».
5. Шумные пулы (Kafka + auditd + широкий FIM) мерить отдельно.

### Проверка прода (пока это не пройдено — это не прод)

1. Все роли **4.14.7**; indexer `_cluster/health` green; replica на месте.
2. Алерт с агента ЦОД-1 и с агента ЦОД-2 доехал в **один** индекс.
3. Убить worker: агенты через LB на живых; `events_dropped=0` после догона.
4. Убить одну ноду indexer: индекс полный, не red.
5. Учение «ЦОД master мёртв»: enrollment/API/dashboard; сколько анализа осталось на worker. Второго master нет — runbook restore с PVC, не «как etcd».
6. Попытка с дефолтным `password` overlay — отказ. UI с мира — отказ.
7. Rolling indexer по одному поду: без replica выкат = деградация поиска.

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + агенты на площадках)

| Сильное | Слабое |
|---|---|
| Indexer/1516 не зависят от межЦОДового RTT | ЦОД-1 — SPOF SIEM для всех площадок |
| Один ruleset, один индекс, один SOC | Путь агентов ЦОД-2/3 до 1514 пересекает город |
| Согласовано с запретом stretch и с «не три SIEM» | Master — единая точка управления; автоfailover нет |
| Агент мультиплатформенный | DaemonSet без host visibility = галочка, не покрытие |

**Не готов к проду**, если: all-in-one на 2–3 ЦОДа; один indexer без replica; дефолты overlay из git; три независимых Wazuh без склейки «как будто один SOC»; stretch indexer на 9300 между ЦОДами; два master «для HA»; агенты без LB/failover; агент без видимости хоста; `latest`; dashboard в интернет; нет ISM; indexer = платформенный OpenSearch 3.8; ждут, что Wazuh заменит WAF/Falco/NetworkPolicy.

---

## Источники

- Релиз 4.14.7: https://documentation.wazuh.com/current/release-notes/release-4-14-7.html
- Архитектура и порты: https://documentation.wazuh.com/current/getting-started/architecture.html
- Quickstart all-in-one: https://documentation.wazuh.com/current/quickstart.html
- Один master, `ossec.conf` не синхронизируется: https://documentation.wazuh.com/current/user-manual/wazuh-server-cluster/types-of-nodes.html
- Агенты: LB vs список: https://documentation.wazuh.com/current/user-manual/wazuh-server-cluster/agent-connections.html
- Шарды/реплики, дефолт 0 replica: https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-tuning.html
- K8s, clone `-b v4.14.7`, StatefulSet: https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-deployment.html
- Репозиторий: https://github.com/wazuh/wazuh-kubernetes/tree/v4.14.7
- Правила и пробелы: `Wazuh.md`

Порога RTT для stretch в документации Wazuh **нет** — поэтому stretch в этой инструкции не предлагается.
