# PostgreSQL 18.6 — установка и конфигурирование

Связанный документ (глоссарий, HA, безопасность, почему так): `PostgreSQL.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного кластера на несколько ЦОДов **не делаем**: межЦОДовый RTT для sync-репликации и для Raft Kubernetes неприемлем.

Версии: **PostgreSQL 18.6**, на Kubernetes — оператор **CloudNativePG 1.30.0**, образ инстансов `ghcr.io/cloudnative-pg/postgresql` с тегом линии **18.6** (не дефолт оператора 18.4, не `latest`).  
Документация: https://www.postgresql.org/docs/18/ · оператор: https://cloudnative-pg.io/documentation/1.30/

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Один CNPG `Cluster` живёт **внутри одного ЦОДа** (одного Kubernetes). Между ЦОДами — только асинхронный replica cluster и/или PITR из object storage. Автоfailover между Kubernetes у CNPG **нет**.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Dev — изолированная сеть; пароль в примере не секрет.
4. Нагрузки нет — поэтому **нет** цифры CPU/RAM/диска «хватит для прода». Есть минимум, чтобы процесс жил, и рычаги роста.
5. Object storage (S3/Swift) для WAL-архива в проде **есть или будет**. Без него нет PITR.
6. Берём community PostgreSQL, не Postgres Pro / EDB.
7. Для 2 ЦОДов: активный кластер в ЦОД-1, DR-копия в ЦОД-2. Для 3 ЦОДов: то же + вторая DR-копия или только бэкапы в ЦОД-3. Третий ЦОД **не** добавляет третий writer.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** схемы, миграции, подключение микросервиса. **Не** цель: отказ площадки и терабайты.

### Предпосылки

- Docker Engine (стенд разработчика) **или** любой однонодовый Kubernetes.
- Порт 5432 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name pg-dev \
  -e POSTGRES_PASSWORD=dev \
  -e POSTGRES_DB=app \
  -p 127.0.0.1:5432:5432 \
  postgres:18.6
```

Привязка к `127.0.0.1` обязательна: `-p 5432:5432` без адреса часто слушает все интерфейсы.

Проверка:

```bash
docker exec -it pg-dev psql -U postgres -c 'SELECT version();'
```

В строке должна быть **18.6**. Затем завести **не**-суперпользователя:

```bash
docker exec -it pg-dev psql -U postgres -d app -c "
CREATE ROLE app LOGIN PASSWORD 'dev-app';
GRANT CONNECT ON DATABASE app TO app;
GRANT CREATE, USAGE ON SCHEMA public TO app;
"
```

Клиент (JDBC/`libpq`) — на `localhost:5432`, БД `app`, роль `app`. Не `postgres`.

### Установка (Kubernetes Dev, если стенд уже в K8s)

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.0.yaml
kubectl -n cnpg-system rollout status deployment/cnpg-controller-manager
```

Минимальный `Cluster` (не прод-манифест):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-dev
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:18.6
  storage:
    size: 10Gi
  bootstrap:
    initdb:
      database: app
      owner: app
```

Сервис записи: `pg-dev-rw:5432`. Образ пинить на 18.6 явно.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Один инстанс | да | Некому строить sync |
| TLS | нет | Иначе PKI раньше схемы |
| `shared_buffers` | дефолт | Нет нагрузки |
| PgBouncer | не нужен | Десяток соединений |
| WAL archive | можно без на сутки | На препроде включить **тот же** механизм, что в проде |
| `POSTGRES_HOST_AUTH_METHOD=trust` | **запрещён** даже на Dev | Привычка уедет в прод |

Чего **не** упрощать: версия 18.6; отдельная БД/роль; клиент ходит на hostname, не на IP контейнера.

### Проверка Dev

1. `SELECT version()` = 18.6.
2. Приложение коннектится ролью `app`, суперпользователь из приложения — отказ.
3. Рестарт контейнера/пода: данные на volume живы (если volume был).

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный образ | Нет failover, нет лага replica, нет PITR |
| Совпадает с development-практикой PostgreSQL | Успешный `INSERT` **не** доказывает прод |
| Дешёво гоняет миграции | Открытый 5432 на Wi-Fi = дыра |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой RPO>0 и **ручного** promote / restore. Цифр TPS нет — ниже минимум HA внутри площадки, не смета железа.

### Почему не stretch

Sync `COMMIT` не быстрее, чем flush WAL на standby. При неприемлемом RTT между ЦОДами stretch даёт таймауты приложений и ложный failover, а не защиту. CNPG: автоfailover **между кластерами Kubernetes не в scope**.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один CNPG `Cluster`:

- `instances: 3` (1 primary + 2 hot standby) на **разных нодах** одного ЦОДа;
- sync `ANY 1` + `dataDurability: required` — RPO≈0 **внутри** площадки;
- клиенты пишут в Service `…-rw`, читают с лагом в `…-ro`;
- перед `-rw` — CNPG `Pooler` (PgBouncer), начать с режима **session**;
- PVC локальные/зональные, `WaitForFirstConsumer`, **не NFS** как единственный диск primary;
- образ **18.6**, не 18.4.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: активный Cluster. ЦОД-2: CNPG **replica cluster** (streaming и/или WAL из object store) **или** только PITR в object storage | Запись стоит, пока оператор не promote / restore. RPO при чисто архивном догоне **> 0** |
| **3 ЦОДа** | ЦОД-1: активный. ЦОД-2: replica cluster. ЦОД-3: второй replica cluster **или** только бэкапы | То же; третий ЦОД не даёт второго writer |

Клиенты ЦОД-2/3 в штате ходят в `-rw` ЦОД-1 по городу (TLS обязателен) **или** работают read-only с локальной replica (лаг, без записи). «Пишем во все три ЦОДа» community PostgreSQL **не умеет**.

Несколько **разных** Cluster на платформе (SoT карточек ≠ Grafana metadata ≠ Camunda Identity) — отдельные HA, не одна БД «на всех».

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не один stretch-кластер).
- CSI с local/зональным диском; object storage для Barman Cloud **plugin** (in-tree Barman в CNPG deprecated, снятие обещано в 1.31.0).
- NetworkPolicy: 5432 только приложениям и инстансам PG.
- Секреты ролей — в Secret/Vault, не в Git.

### Установка оператора (на каждом Kubernetes, где будет PostgreSQL)

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.0.yaml
```

Оператор — Deployment с несколькими репликами, **не** в одном поде с primary.

### Конфигурирование активного кластера (ЦОД-1)

Смысл манифеста (не полный CR — сверять с докой 1.30):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-sot
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:18.6
  postgresql:
    synchronous:
      method: any
      number: 1
      dataDurability: required
    parameters:
      ssl: "on"
      # shared_buffers / max_connections — после замера, не «25% с потолка»
  storage:
    size: 100Gi   # заглушка: реальный размер = данные + WAL + запас; цифр нагрузки нет
  # bootstrap, backup (Barman Cloud plugin), monitoring — по доке CNPG 1.30
```

Обязательно рядом:

1. **Pooler** перед `-rw` (и отдельно перед `-ro`, если нужно). `max_connections` на сервере держать скромным.
2. **ScheduledBackup** + WAL archive в object storage. Проверяют **restore**, не наличие файла в бакете.
3. Роли на сервис (`DatabaseRole` / SQL): без `SUPERUSER`, без `CREATEDB`. `pg_hba`: `hostssl` + SCRAM, не `trust`, не `0.0.0.0/0`.
4. Anti-affinity по ноде. PDB: не эвакуировать primary и единственную sync-replica одним drain.
5. Клиенты: короткий timeout, reconnect, **не** sticky на pod IP. `sslmode=verify-full` в проде.

Replica cluster в ЦОД-2 — отдельный `Cluster` с bootstrap от источника (streaming / object store). Promote — **ручной/GitOps**. После кривого promote возможны две ветки WAL: это потеря данных, не «потом смержим».

`dataDurability: preferred` — запись продолжится без живой sync-replica (окно потери). Если руководство требует «пишем любой ценой» — это **другое** обещание, не RPO≈0.

### Масштабирование (когда появятся цифры)

1. Замерить TPS, WAL/с, p95 `COMMIT`, число сессий, bloat.
2. Упёрлись CPU/WAL primary — вертикаль или вынести домен в другой Cluster. **Ещё replica не ускоряет запись.**
3. Упёрлись соединения — Pooler, не `max_connections=5000`.
4. Упёрлись диск — партиции + архив старого в озеро, не бесконечный retention в OLTP.
5. Тяжёлые отчёты — ClickHouse/склад, не primary SoT.

`fsync` **не** выключать «для скорости».

### Проверка прода (пока это не пройдено — это не прод)

1. `SELECT version()` = 18.6 на всех инстансах ЦОД-1.
2. Запись в `-rw`, чтение `-ro`, запись в `-ro` — отказ.
3. Убить под primary: `-rw` переехал, приложение переподключилось, число строк совпало с ожиданием sync.
4. Restore PITR на чистый Cluster **на стенде**.
5. Promote replica cluster в ЦОД-2 на препроде (с замером RTO) — хотя бы один раз.
6. Суперпользователь из приложения не проходит.

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + async DR)

| Сильное | Слабое |
|---|---|
| Sync не зависит от межЦОДового RTT | Падение ЦОД-1 = нет записи, пока promote/restore |
| Согласовано с «не stretch» и с CNPG (HA внутри K8s) | RPO между ЦОДами > 0, если догон архивом |
| Один writer, понятная модель для разработчиков | Ручной DR; ошибка promote = split-brain |
| | Клиентский 5432 в штате идёт через город — TLS и сеть ваши |

**Не готов к проду**, если: `instances: 1`, образ 18.4/`latest`, нет WAL archive, NFS как диск primary, `trust`/`sslmode=disable` между площадками, клиент на IP пода, Dev-манифест скопирован в бой.

---

## Источники

- PostgreSQL 18: https://www.postgresql.org/docs/18/
- CNPG 1.30: https://cloudnative-pg.io/documentation/1.30/
- Манифест оператора: ветка `release-1.30`, файл `cnpg-1.30.0.yaml`
- Правила и пробелы (RTT, SoT vs metadata): `PostgreSQL.md`
