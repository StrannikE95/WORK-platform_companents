# Apache Superset 6.1.0 — схемы устройства

Связанные документы: правила — `Apache Superset.md`; установка — `Apache Superset.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код чартов не рисуем.

Допущения: stretch metadata Postgres + Redis на 2–3 ЦОДа **нет**; Helm `superset/superset` **0.21.1**, образ **6.1.0**; не Grafana и не склад; нагрузка не замерена.

---

## 1. Контекст

Superset — **BI поверх SQL**. Терабайты живут в озере/витрине. Он *спрашивает* склад и рисует картинку.

```mermaid
flowchart LR
  PPL["Аналитики / SSO"]
  SS["Superset 6.1.0"]
  LAKE["Озеро / витрина SQL"]
  GF["Grafana"]
  KF["Kafka"]

  PPL -->|"HTTPS :8088"| SS
  SS -->|"read-only SQL"| LAKE
  SS -.->|"не метрики K8s"| GF
  SS -.->|"не шина"| KF
```

Без витрины в SQL Superset — пустая рамка. Он **не создаёт** озеро. Операционные БД Camunda/интеграций аналитикам не открывать — смотреть проекции.

---

## 2. Контейнеры (несколько процессов одного ПО)

Это **не** кворум. Stateless web/worker + **одна** metadata DB + **один** Redis **внутри ЦОД-1**.

```mermaid
flowchart TB
  LB["HAProxy / Ingress\n/health → OK"]
  subgraph dc["ЦОД-1"]
    W1["web Gunicorn"]
    W2["web"]
    WK1["Celery worker"]
    WK2["Celery worker"]
    BT["Celery beat = 1"]
    PG["Postgres база superset"]
    RD["Redis broker + cache"]
  end

  LAKE2["Склад / реплика озера"]

  LB --> W1
  LB --> W2
  W1 --> PG
  W2 --> PG
  W1 --> RD
  W2 --> RD
  WK1 --> PG
  WK2 --> PG
  WK1 --> RD
  WK2 --> RD
  WK1 --> LAKE2
  WK2 --> LAKE2
  W1 --> LAKE2
  W2 --> LAKE2
  BT --> RD
```

Порт **8088** — UI/API. **5432** и **6379** — не Superset. Beat **ровно один**: два beat = двойные письма. `/health` **не** проверяет Postgres/Redis/склад.

Сабчарты Bitnami в чарте — стенд. Прод: `postgresql.enabled: false`, `redis.enabled: false`. SQLite — не несколько подов.

**Сильное:** падение одного web — UI жив (cookie, sticky не обязателен). **Слабое:** падение metadata DB = нет логина, даже если поды зелёные.

---

## 3. Компоненты внутри стека

```mermaid
flowchart TB
  subgraph web["Web"]
    UI["Explore / SQL Lab / API"]
    SK["SECRET_KEY\ncookie + шифр паролей"]
  end

  META["Метаданные: дашборды, RLS,\nзашифрованные URI складов"]
  CACHE["DATA_CACHE_CONFIG"]
  CEL["Очередь Celery"]

  UI --> META
  SK --> META
  UI --> CACHE
  CEL --> CACHE
```

| Компонент | Для чего настраивать |
|---|---|
| `SECRET_KEY` | С 2.1.0 дефолт на не-debug **запрещает старт**. Helm-дока знает `thisISaSECRET_1234` — тоже не оставлять |
| NullCache | Дефолт `CACHE_CONFIG` / `DATA_CACHE_CONFIG`. Refresh дашборда = шторм SQL |
| Results backend | Общий Redis, не диск пода: иначе web-A не заберёт результат worker-B |
| SQL Lab | Консоль к складу от имени учётки datasource. RLS приложения **обходима** |

`superset run` / `flask run` — не прод. Alerts & Reports — beat + worker + webdriver; в первом проде можно выкл.

---

## 4. Поток: открыть чарт

```mermaid
sequenceDiagram
  participant U as Браузер
  participant W as Web
  participant C as Redis cache
  participant D as Metadata Postgres
  participant L as Склад SQL

  U->>W: дашборд
  W->>D: определение чарта / RLS
  W->>C: hit?
  alt NullCache или miss
    W->>L: SQL
    L-->>W: строки ≤ ROW_LIMIT
    W->>C: положить, если кэш включён
  end
  W-->>U: картинка
```

Нагрузка на склад ≈ N web × gunicorn × одновременные чарты × (1 − cache hit). При NullCache hit = 0. Добавить реплик «для HA» можно съесть `max_connections` озера.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    WR["web ≥2 worker ≥2"]
    B1["beat ≤1"]
    HA["Postgres HA + Redis HA"]
    PDB["PDB web; в чарте PDB дефолт выкл"]
  end

  subgraph dr["Между ЦОДами"]
    GIT["git-export дашбордов"]
    ST2["Люди → UI ЦОД-1\nили свой стек"]
  end

  inside -->|"падение web"| OK["LB жив"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если забыть |
|---|---|
| Свой SECRET_KEY **до** datasources | Смена потом — `re-encrypt-secrets` |
| `ENABLE_PROXY_FIX` + cookie Secure | OAuth/`redirect_uri` ломаются за TLS-прокси |
| Non-root образ с драйвером склада | Чарт `runAsUser: 0` + apt на старте — не прод |
| Init Job один раз | `db upgrade` с трёх подов сразу |

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Ось"]
  Q --> P["Зрители"]
  Q --> S["Тяжёлый SQL"]
  Q --> T["Терабайты"]

  P --> P1["Реплики web + кэш Redis\nне refresh 10s вслепую"]
  S --> S1["Worker + витрины склада\nне ещё один web"]
  T --> T1["Масштаб озера\nmetadata DB маленькая"]
```

Gunicorn `-w 10` gevent в доке — **пример**. `ROW_LIMIT` 50k / `SQL_MAX_ROW` 100k — потолки UI, не индекс озера. Читать **реплику** склада, не primary SoT.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1"]
    ST["web/worker/beat + PG + Redis"]
  end
  subgraph dc2["ЦОД-2"]
    A["Доступ к UI ЦОД-1 или независимый стек"]
  end
  subgraph dc3["ЦОД-3"]
    B["То же; не второй writer метаданных"]
  end
  ST -.-> A
  ST -.-> B
```

Три полных Superset — тройные права, обычно хуже. Склад в другом ЦОДе: учесть `SQLLAB_TIMEOUT` / Gunicorn / Ingress.

**Сильное:** Postgres/Redis не на WAN. **Слабое:** падение ЦОД-1 = нет этого BI; `/health` слеп к складу; Admin = ключ ко всем URI и к SQL.

---

## 8. Безопасность (ручки на той же схеме)

Нет `admin`/`admin`. SSO, регистрация **не** в Admin. Public/anonymous на ПДн не включать. `PREVENT_UNSAFE_DB_CONNECTIONS` не выключать. `DATASET_IMPORT_ALLOWED_DATA_URLS` сузить (дефолт `.*` — SSRF). Scarf выкл. SQL Lab — узкая роль, учётка БД **SELECT**. Flower не в интернет.

Источники: `Apache Superset.md`. Порога RTT у проекта **нет**.
