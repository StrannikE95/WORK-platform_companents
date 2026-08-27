# PostgreSQL 18.6 — схемы устройства

Связанные документы: правила — `PostgreSQL.md`; установка — `PostgreSQL.install.md`.

Ниже **C4** (четыре уровня: контекст → контейнеры → компоненты → код). В запросе это «D4»: тот же каркас. Уровень кода пропускаем — для эксплуатации важнее роли, порты и ручки HA/масштаба.

Допущения (их не было в исходном ТЗ платформы, без них схема врёт):

1. Stretch одного кластера на 2–3 ЦОДа **нет** (RTT не позволяет sync `COMMIT` / Raft). HA — **внутри ЦОД-1**.
2. Community PostgreSQL 18.6 + CloudNativePG 1.30.0. Один writer на кластер.
3. Нагрузки нет — на схемах нет «N ядер». Есть *что крутить*, когда цифры появятся.

---

## 1. Контекст (C4 system context)

Где PostgreSQL в вашей картине. Он **не** шина и **не** аналитический склад.

```mermaid
flowchart LR
  subgraph people["Люди и сервисы"]
    MS["Микросервисы / интеграционное API"]
    CW["Camunda job workers"]
    BI["Grafana / Superset / metadata"]
  end

  PG["PostgreSQL 18.6\nOLTP / карточки / справочники"]
  KF["Apache Kafka\nшина событий"]
  CH["ClickHouse / озеро файлов\nаналитика и сырьё"]
  RD["Redis / Valkey\nкэш, не истина"]

  MS -->|"COMMIT, outbox"| PG
  CW -->|"состояние не в Zeebe"| PG
  BI -->|"отдельный Cluster, не SoT"| PG
  PG -->|"факт изменилось"| KF
  PG -.->|"не замена"| CH
  RD -.->|"не карточка клиента"| PG
```

| Стрелка | Зачем помнить |
|---|---|
| Сервис → PG | Транзакция и внешние ключи. Истина «текущая карточка» |
| PG → Kafka | Событие *после* `COMMIT` (outbox). Иначе БД и шина разъедутся |
| BI → свой Cluster | Падение отчёта не должно валить SoT клиентов |

**Слабое место контекста:** положить сырые XML ведомств и вложения на годы в одну БД — диск и vacuum съедят primary. Файлы — object storage.

---

## 2. Контейнеры (из чего состоит решение)

Один логический кластер = **один primary** + standby + то, что *рядом*, но уже другое ПО.

```mermaid
flowchart TB
  subgraph k8s["Один Kubernetes = один ЦОД"]
    OP["Оператор CNPG\nне хранит ваши таблицы"]
    PL["Pooler / PgBouncer\nмного клиентов → мало сессий"]
    subgraph cnpg["CNPG Cluster"]
      RW["Service -rw\nзапись"]
      RO["Service -ro\nчтение с лагом"]
      P1["Pod primary\nPGDATA + WAL"]
      S1["Pod standby"]
      S2["Pod standby"]
    end
    CSI["CSI / локальный диск\nне NFS как единственный диск"]
  end

  OBJ["Object storage\nWAL archive + бэкап PITR"]
  APP["Приложения"]

  APP --> PL
  PL --> RW
  APP -->|"SELECT с лагом"| RO
  RW --> P1
  RO --> S1
  RO --> S2
  P1 -->|"streaming 5432"| S1
  P1 -->|"streaming 5432"| S2
  P1 --> CSI
  S1 --> CSI
  S2 --> CSI
  P1 -->|"WAL archive"| OBJ
  OP -->|"promote, сертификаты, Endpoints"| cnpg
```

Порт **5432/TCP** — и клиенты, и репликация. Отдельного «порта кластера» нет. Балансировщик HTTP «на все 5432» без rw/ro **ломает** модель: запись попадёт на replica.

**Сильное:** оператор сам перекладывает `-rw` после failover. **Слабое:** `DROP TABLE` уедет на все standby; replica **не** бэкап.

---

## 3. Компоненты внутри инстанса (что крутится в одном поде)

```mermaid
flowchart TB
  subgraph instance["Один инстанс postgres"]
    PM["postmaster"]
    BE["backend-процессы\nпо сессии"]
    WALp["WAL writer / checkpointer"]
    AV["autovacuum"]
    HBA["pg_hba.conf\nкого пускать"]
    SB["shared_buffers\nкэш страниц внутри PG"]
  end

  CLI["Клиент libpq / JDBC"]
  DISK["PGDATA + WAL на диске"]

  CLI -->|"5432 TLS + SCRAM"| HBA
  HBA --> BE
  PM --> BE
  BE --> SB
  BE --> WALp
  WALp --> DISK
  AV --> DISK
```

| Компонент | Для чего настраивать |
|---|---|
| `pg_hba` + роли | Безопасность: не `trust`, не superuser в приложении |
| `shared_buffers` / `work_mem` | Производительность; дефолт ~128MB для прода мал |
| `max_connections` | Потолок процессов. Рост — **PgBouncer**, не 5000 backend |
| autovacuum | Без него bloat и остановка записи (TXID wraparound) |
| `synchronous_*` | RPO≈0 vs «пишем всегда». См. ниже |

---

## 4. Поток записи (как взаимодействуют)

```mermaid
sequenceDiagram
  participant App as Приложение
  participant P as PgBouncer / -rw
  participant Pri as Primary
  participant St as Sync standby
  participant Arc as WAL archive

  App->>P: BEGIN / INSERT / COMMIT
  P->>Pri: та же сессия или transaction pooling
  Pri->>Pri: WAL на локальный диск fsync
  Pri->>St: streaming WAL
  Note over Pri,St: sync ANY 1: COMMIT ждёт flush на standby
  St-->>Pri: подтверждение
  Pri-->>App: COMMIT OK
  Pri->>Arc: архив сегмента позже
```

Асинхронная репликация: клиент получает OK **до** догона replica → падение primary = потеря уже «подтверждённого». Это не баг, это выбор RPO.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    I3["instances: 3\nразные ноды"]
    SY["synchronous ANY 1\ndataDurability required"]
    SLOT["replication slots\nчтобы WAL не стёрся"]
    PDB["PDB / anti-affinity"]
  end

  subgraph dr["Между ЦОДами — не stretch"]
    RC["Replica cluster async\nили только PITR"]
    MAN["Promote вручную / GitOps"]
  end

  inside -->|"падение пода"| FA["CNPG promote\n-rw переезжает"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если не настроить |
|---|---|
| 3 инстанса + anti-affinity | Два пода на одной VM = «HA на бумаге» |
| sync `ANY 1` + `required` | Дефолт без блока sync = RPO не 0 |
| `preferred` вместо `required` | Запись жива без replica, окно потери |
| WAL archive + проверка restore | Три replica не вернут «вчера 15:03» |
| Клиент на DNS `-rw`, не IP пода | После failover приложение «висит» |

Падение **двух** ЦОДов при мозге в одном — нет записи, пока restore. Это цена запрета stretch.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> W["Запись / WAL / CPU primary"]
  Q --> R["Чтение"]
  Q --> C["Число соединений"]
  Q --> D["Диск / bloat"]

  W --> W1["Вертикаль primary\nили другой Cluster на домен"]
  W --> W2["Не добавлять replica для записи"]
  R --> R1["-ro + лаг\nтяжёлое → ClickHouse"]
  C --> C1["Pooler session затем transaction"]
  D --> D1["Партиции + архив в озеро\nautovacuum"]
```

Горизонтали **записи** у community PostgreSQL нет. Партиции режут vacuum, не дают второго writer.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    CL["Cluster instances 3\nsync внутри площадки"]
  end
  subgraph dc2["ЦОД-2"]
    DR1["Replica cluster async\nили холодный PITR"]
  end
  subgraph dc3["ЦОД-3 если есть"]
    DR2["Вторая копия бэкапа\nне третий writer"]
  end

  CL -->|"streaming / object store"| DR1
  CL -->|"архив WAL"| DR2
```

Клиенты в штате пишут в `-rw` ЦОД-1 (TLS через город). «Три primary» community PostgreSQL **не умеет**.

**Сильное:** латентность `COMMIT` не прибита межЦОДовым RTT. **Слабое:** смерть ЦОД-1 = простой записи до ручного promote; RPO между площадками > 0, если догон архивом.

---

## 8. Безопасность (ручки на той же схеме)

Не отдельный «кластер ИБ», а слои на 5432:

1. NetworkPolicy — кто видит порт.
2. `hostssl` + SCRAM / cert — кто ты.
3. GRANT / RLS — что можно. Приложение ≠ superuser.

At-rest — шифрование тома CSI, не настройка `ssl`.

---

Источники фактов: `PostgreSQL.md` (роли, порты, CNPG, sync). Порога RTT у проекта PostgreSQL **нет** — stretch на схемах не рисуем как целевой.
