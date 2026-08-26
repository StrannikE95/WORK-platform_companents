# Camunda 8.9 — установка и конфигурирование

Связанный документ (глоссарий, Raft/Zeebe, dual-region, почему так): `Camunda 8.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного кластера Zeebe на несколько ЦОДов **не делаем**: commit ждёт кворум по **26502**; межЦОДовый RTT это запрещает. Официальный **dual-region** Camunda — тоже stretch (Raft на две площадки, RF=4) и **не выбран**.

Версии: **Camunda 8.9**, Helm **`camunda-platform` 14.8.5**, образ Orchestration Cluster **`camunda/camunda:8.9.17`**. Соседние образы той же строки матрицы: Connectors 8.9.8, Optimize 8.9.17, Management Identity 8.9.8, Web Modeler 8.9.7, Console 8.9.88.  
Матрица: https://helm.camunda.io/camunda-platform/version-matrix/camunda-8.9/  
Документация: https://docs.camunda.io/  
**8.10** на дату соседнего файла — Alpha; в прод не брать. Апгрейд на 8.9 — только с 8.8.x.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён**, включая dual-region. Вариант A соседнего файла (брокеры в трёх ЦОДах) и вариант B (dual-region) **несовместимы** с ограничением ping. Выбран **вариант C**: RF=3 **внутри одного ЦОДа** + Cold Recovery.
2. Прод — Kubernetes 1.36.x в каждом ЦОДе (`Kubernetes.install.md`). Совместимость чарта 14.8.5 с 1.36 — проверить CSI/PDB на момент установки.
3. Self-Managed, не SaaS. Прод **требует лицензию** с 8.6 (`CAMUNDA_LICENSE_KEY`). Bitnami Elasticsearch/PostgreSQL из чарта в проде **выключены**.
4. Secondary storage прода — **OpenSearch 3.8.0** (ветка 3.6+ для 8.9). С 8.9 без `orchestration.data.secondaryStorage.type` чарт **не ставится**. Dual-region с OpenSearch вендор **не поддерживает** — ещё одна причина не брать B.
5. Dev — изолированная сеть. C8 Run / H2 / Compose — только Dev.
6. Нагрузки нет — нет числа партиций «хватит».
7. Для 2 ЦОДов: активный кластер в ЦОД-1, в ЦОД-2 — прогнанный restore. Для 3 — то же + вторая копия бэкапа или второй restore-стенд. Третий ЦОД **не** член Raft.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** BPMN, `@JobWorker`, Operate. **Не** цель: Raft между площадками.

### Предпосылки

- Docker **или** Dev-Kubernetes. Java воркера совместима с 8.9 (брокеры: OpenJDK 21–25).
- Не смешивать C8 Run и Helm в одной привычке «это почти прод».

### Установка (C8 Run / Compose — основной путь Dev)

**Camunda 8 Run** или Docker Compose из developer quickstart: Orchestration Cluster + Connectors, secondary по умолчанию **H2**. Вендор: **не для production**. Windows/macOS как host — только dev. Образы Linux amd64/arm64.

Клиенты (воркеры, UI) ходят на gateway **8080** (REST) и **26500** (gRPC). **26501/26502** — внутренние (command / Raft+Gossip); на Dev не публиковать их на все интерфейсы. **9600** — actuator/metrics, не в интернет.

Минимум после старта: задеплоить процесс с одной сервисной задачей, живой воркер снаружи движка, экземпляр доходит до конца. Payload — идентификатор, не XML досье (потолок порядка **~3 МБ** на экземпляр — эвристика вендора).

### Установка (Kubernetes Dev)

Чарт **14.8.5**, `values-local.yaml`. Явно:

```yaml
orchestration:
  clusterSize: 1
  replicationFactor: 1
  partitionCount: 1
  data:
    secondaryStorage:
      type: elasticsearch   # или opensearch / rdbms — задать ДО install
```

H2 в Helm-проде нет; на Dev допустим маленький OpenSearch в соседнем namespace **или** сознательный H2 только в C8 Run. **Не** этот values в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Брокеры / RF / партиции | 1 / 1 / 1 | Не отлаживать Raft вместо BPMN |
| Secondary | H2 (C8 Run) или маленький OS | H2 не учит экспорт — на препроде OS обязателен |
| Auth | basic | Иначе IdP раньше воркера |
| Лицензия | Non-Production | Стенд не прод |
| Optimize / Web Modeler | выкл | Не нужны, чтобы понять job worker |
| Переменные процесса | id/статус, не досье | Иначе в проде раздует партиции |

Чего **не** упрощать: версия **8.9.x** и клиент воркера под 8.9; воркеры **снаружи** JVM; не открывать 8080 «навсегда».

### Проверка Dev

1. Образ/версия 8.9.x. Процесс виден в Operate (если не H2-only без экспорта — хотя бы complete job).
2. Рестарт единственного брокера при RF=1: незареплицированное теряется — ожидаемо.
3. Не копировать C8 Run values в GitOps прода.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный C8 Run | Нет Raft, нет экспорта в OS 3.8.0 |
| Учит C8, не `JavaDelegate` C7 | Успех на H2 **не** доказывает backpressure при падении OS |
| | Basic `demo` приучает вход без IdP |

Препрод: 3 брокера, RF=3, свой OpenSearch, OIDC stage, backup API — в **одном** ЦОДе, даже без боя.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **брокера/ноды внутри ЦОДа** при RF=3; пережить отказ **целого ЦОДа** ценой простоя до Cold Recovery. Dual-region (RPO 0, ручной failover, Elasticsearch) **не выбран**: это Raft по 26502 между регионами, OpenSearch нельзя, ping запрещает.

### Почему не stretch и почему не dual-region

Запись коммитится, когда большинство копий партиции подтвердило. 26502 между ЦОДами при неприемлемом RTT → медленные процессы, ложный fail, backpressure. Порога RTT для **трёх** площадок у вендора **нет**. Dual-region: ориентир **≤ 100 мс**, минимум 8 брокеров, RF=4, при потере региона обработка **встаёт** до runbook, secondary **только Elasticsearch**. Это форма stretch — в этой инструкции **не ставится**.

Выбор: **вариант C** соседнего файла.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один Orchestration Cluster:

- `clusterSize: 3`, `replicationFactor: 3`, `partitionCount` кратно 3 (старт часто **3**, не 1);
- брокеры на **разных нодах** этого ЦОДа (anti-affinity / topology spread **зала**, не id чужого ЦОДа);
- Gateway: несколько реплик входа (с 8.8 часто embedded; масштабируют репликами), клиенты на **8080 / 26500**, не на 26501/26502;
- Secondary: отдельный **OpenSearch 3.8.0**, `orchestration.data.secondaryStorage.type: opensearch`. Не бизнес-поиск, не Wazuh. Один кластер OS на Orchestration Cluster (не «Operate на одном, Tasklist на другом»);
- PVC **RWO** на брокера, SSD, вендор: **≥ 1000 IOPS**, HDD не поддерживается. `emptyDir` запрещён. NFS — только POSIX single-writer;
- PostgreSQL Identity/Modeler — внешний CNPG **в этом ЦОДе** (`PostgreSQL.install.md`), не Bitnami-subchart;
- Optimize в первом проде **выкл** (соседний файл: ×3–4 к OS).

**Между ЦОДами — только Cold Recovery:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: кластер RF=3. ЦОД-2: объектное хранилище бэкапов Zeebe + snapshot OS; процедура restore в свой Kubernetes | Оркестрация стоит, пока restore. RPO = период backup. RTO **мерить** |
| **3 ЦОДа** | То же + копия бэкапа / второй restore-кластер в ЦОД-3 | То же; третий ЦОД не член 26502 |

Клиенты ЦОД-2/3 в штате ходят на gateway ЦОД-1 (TLS, gRPC-aware LB на 26500). «Пишем Raft во все ЦОДы» **не делаем**.

### Предпосылки прода

- Kubernetes в каждом ЦОДе. Лицензия Enterprise Self-Managed. OIDC, не `demo:demo`.
- OpenSearch 3.8.0 для Camunda **до** `helm install`. Смена семейства secondary потом — не in-place.
- Бакет: (1) snapshot repository OS, (2) store **partition backup Zeebe**. Снимок диска брокера **не** согласованный бэкап.
- Helm CLI **v3** (матрица 14.8.x: Helm 3.20.x). Helm v4 — требование 8.10.

### Установка (ЦОД-1)

```bash
helm repo add camunda https://helm.camunda.io
helm install camunda camunda/camunda-platform --version 14.8.5 \
  -f values-prod.yaml
```

В `values-prod.yaml` обязательно (смысл, сверять с докой 8.9):

- `orchestration.data.secondaryStorage.type: opensearch` (+ URL/credentials вашего OS 3.8.0);
- `clusterSize` / `partitionCount` / `replicationFactor`: **3 / 3 / 3** пока бенчмарк не скажет иначе;
- `elasticsearch.enabled: false`; свой OS;
- `global.license.secret`; `global.security.authentication.method` — OIDC, не basic;
- два namespace как в production-гайде: `orchestration` и при необходимости `management-and-modeling`.

`initialContactPoints` — **все** брокеры. PDB: не снять сразу два из трёх. NetworkPolicy: 26501/26502 только брокеры+gateway; 9600 — Prometheus/backup.

### Конфигурирование

| Параметр | Прод | Зачем |
|---|---|---|
| RF | 3 внутри ЦОДа | Пережить 1 брокер; два мёртвых из трёх = нет кворума |
| Secondary type | `opensearch` явно | С 8.9 чарт без type не ставится |
| Bitnami ES/PG | false | Не оценка «чтобы чарт поднялся» |
| 26502 наружу / в чужой ЦОД | нет | Stretch |
| Dual-region values | не применять | Несовместимо с ping и с OpenSearch |
| Variables | килобайты, id | Потолок порядка ~3 МБ на экземпляр — не озеро |
| Connectors в ведомства | нет | Воркер → ваше интеграционное API |

Backup: API компонентов с **одним** backup ID (Zeebe + индексы экспорта). Restore — на другой namespace **и** на кластер ЦОД-2. Без прогона restore нет DR.

Воркеры: Deployment ≥2 на `job type`, timeout > лага госапи, идемпотентность. HPA по своим метрикам, не по CPU брокера. Kafka: воркер **или** inbound connector стартует процесс; не дублировать «и коннектор, и свой consumer» без ключа экземпляра. Интеграционное API — единственный выход в ведомства; Connectors Camunda туда не плодим.

`values-tls.yaml` чарта закрывает TLS **к** OpenSearch/PostgreSQL/OIDC. **Не** закрывает весь gRPC между подами: для 26501/26502 вендор отсылает на service mesh или свою схему. NetworkPolicy обязателен в любом случае.

Helm `clusterSize` / `partitionCount` / `replicationFactor` на всех брокерах **одинаковые**. RF не больше числа брокеров. После старта новые брокеры — процедурой scaling, не второй кластер с тем же именем.

### Масштабирование (когда появятся цифры)

1. Параллелизм ≈ партиции; добавлять брокеров процедурой scaling, не второй кластер с тем же именем.
2. Operate упирается в OS, не в «ещё брокер».
3. SSD/IOPS брокера: медленный PVC = «движок тупит» при живом кворуме.
4. `fsync`/диск брокера не подменять NFS на троих ReadWriteMany.

### Проверка прода (пока это не пройдено — это не прод)

1. `camunda/camunda:8.9.17`, чарт 14.8.5; type secondary = opensearch.
2. Процесс доходит до конца, виден в Operate. Убить 1 брокер: экземпляр продолжается.
3. Остановить OS на минуты: понять backpressure (это не «баг»).
4. Backup+restore на чистый namespace и на ЦОД-2 (замер RTO).
5. Нет `demo`; 26502 не слушает периметр. Лицензия не Non-Production.
6. Dual-region **не** включён.

### Сильные / слабые стороны (RF=3 в одном ЦОДе + Cold Recovery)

| Сильное | Слабое |
|---|---|
| 26502 не ходит между городами | Падение ЦОД-1 = нет оркестрации, пока restore |
| OpenSearch 3.8.0 остаётся secondary | RPO/RTO = частота backup, надо мерить |
| Согласовано с запретом stretch и с dual-region≠OS | Клиенты с других ЦОДов ходят на gateway через город |
| | Dual-region RPO 0 сознательно не берём |

**Не готов к проду**, если: C8 Run / H2, RF=1, брокеры без anti-affinity внутри ЦОДа, stretch/dual-region, нет `secondaryStorage.type`, Camunda пишет в OS поиска/Wazuh, нет лицензии, нет прогнанного restore, терабайты в variables, коннекторы в обход интеграционного API.

---

## Источники

- Матрица 14.8.5 / `camunda/camunda:8.9.17`: https://helm.camunda.io/camunda-platform/version-matrix/camunda-8.9/
- Supported environments (OS 3.6+, диск ≥1000 IOPS): https://docs.camunda.io/docs/reference/supported-environments/
- Dual-region (OS не поддерживается): https://docs.camunda.io/docs/self-managed/concepts/multi-region/dual-region/
- Backup API, не disk snapshot: https://docs.camunda.io/docs/self-managed/operational-guides/backup-restore/backup-and-restore/
- C8 Run не прод: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/
- Правила: `Camunda 8.md`

Dual-region в этой инструкции не предлагается: это stretch Raft, он несовместим с ограничением ping и с уже выбранным OpenSearch 3.8.0.
