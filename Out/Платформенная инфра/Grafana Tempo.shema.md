# Grafana Tempo 3.0.3 — схемы устройства

Связанные документы: правила — `Grafana Tempo.md`; установка — `Grafana Tempo.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код SDK не рисуем.

Допущения: stretch микросервисов одного кластера на 2–3 ЦОДа **нет**; линия **3.0** — ingester и compactor **удалены**; Helm `tempo-distributed` 3.0.6 по умолчанию тянет **3.0.2** — образ **переопределить на 3.0.3**; нагрузка не замерена.

---

## 1. Контекст

Tempo — **бэкенд трейсов** (Trace ID дёшево, TraceQL дороже). Не логи, не метрики нод, не SoT карточки.

```mermaid
flowchart LR
  APP["Микросервисы / Camunda / API"]
  COL["OTel Collector\nсэмпл, PII, tenant"]
  TMP["Tempo 3.0.3\nтрейсы"]
  GF["Grafana\nUI Tempo нет"]
  KF["Kafka ingest Tempo\nне бизнес-шина"]

  APP -->|"OTLP"| COL
  COL -->|"4317 / 4318"| TMP
  TMP --> KF
  GF -->|"TraceQL / Trace ID"| TMP
```

Приложения **не** пишут в Tempo в обход Collector. Тело SOAP ведомства в атрибуте — нелегальный архив ПДн, не «observability».

---

## 2. Контейнеры (микросервисы одного ЦОДа)

Монолит `-target=all` — стенд, без Kafka в write path. Прод — роли одного бинаря + внешние Kafka и object storage.

```mermaid
flowchart TB
  subgraph dc["ЦОД-1 = один кластер Tempo"]
    DIST["distributor"]
    LS["live-store\nсвежие трейсы + WAL"]
    BB["block-builder"]
    QF["query-frontend ×2"]
    QR["querier"]
    SCH["backend scheduler\nровно 1"]
    WK["backend worker"]
    MC["Memcached"]
  end

  KFK["Kafka-топик ingest\nRF=3 внутри ЦОДа"]
  S3["Object storage\nистория, RF1 Tempo"]
  GW["Gateway\nу Tempo нет auth"]

  COL2["Collector"] -->|"OTLP"| DIST
  DIST -->|"после ack"| KFK
  KFK --> LS
  KFK --> BB
  BB --> S3
  QF --> QR
  QR --> LS
  QR --> S3
  SCH --> WK
  WK --> S3
  QR --> MC
  GW --> QF
```

Порты: **4317** OTLP gRPC, **4318** HTTP, **3200** API, **9095** внутренний gRPC, **7946** memberlist. С 2.7 OTLP по умолчанию **localhost** — в K8s биндить явно.

**Сильное:** после ack Kafka Tempo работает RF1 — экономия места. **Слабое:** вторую копию истории Tempo **не** делает; её дают Kafka (коротко) и бакет (долго).

---

## 3. Компоненты данных

```mermaid
flowchart LR
  T["Trace ID"] --> PART["Одна Kafka-партиция"]
  PART --> LIVE["live-store"]
  PART --> BLK["Parquet-блок в S3"]
  LIVE -->|"минуты–около часа"| Q["querier"]
  BLK -->|"история"| Q
```

Параллелизм write path = **число партиций**, не «ещё один Deployment». `partitions_per_instance: 1` → число block-builder = число партиций. Autoscaling builder «как HTTP» **ломает** привязку.

Scheduler: цитата доки — *only one at a time*. Два scheduler — конфликт работ, не HA.

---

## 4. Поток записи (ingest)

```mermaid
sequenceDiagram
  participant C as Collector
  participant D as Distributor
  participant K as Kafka ingest
  participant L as Live-store
  participant B as Block-builder
  participant O as Object storage

  C->>D: OTLP span
  D->>K: запись
  K-->>D: ack
  D-->>C: OK
  Note over D,K: без ack клиенту успех не возвращают
  K->>L: свежее окно
  K->>B: сборка блока
  B->>O: Parquet один раз RF1
```

`fail_on_high_lag: true` (дефолт 3.0): лучше ошибка «свежие данные отстают», чем тихий неполный TraceQL.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    D3["distributor ≥2 + Service"]
    PVC["PVC RWO на WAL\nне NFS"]
    S1["scheduler = 1 + runbook"]
    K3["Kafka ingest RF=3 min.ISR=2"]
  end

  subgraph dr["Между ЦОДами — не stretch"]
    COLR["Collector ЦОД-2 → ЦОД-1\nили свой Tempo"]
    BAK["HA бакета — не Tempo"]
  end

  inside -->|"падение пода"| OK["приём жив"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если не настроить |
|---|---|
| Override образа **3.0.3** | Чарт оставит 3.0.2 без CVE-патча Go |
| Топик руками, не auto-create | Дефолт **1000 партиций** на боевом кластере |
| Gateway + `X-Scope-OrgID` с прокси | Анонимный 3200 = читать все трейсы |
| Retention топика > цикл builder | Реплей не успеет — дырка в истории |

Падение scheduler: **приём идёт**, компакция/retention встают. Падение бакета = потеря **всей** истории.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  M["Упёрлись"]
  M --> W["MB/s ingest"]
  M --> Q["Широкий TraceQL"]
  M --> D["Диск бакета"]

  W --> W1["Сначала партиции Kafka\nпотом live-store / builder"]
  Q --> Q1["Querier отдельно\nлимиты overrides"]
  D --> D1["Сэмпл в Collector\nblock_retention не «как озеро»"]
```

Формула объёма порядка: пиковый MB/s × доля сэмпла × 86400 × дни retention. Пика нет — **цифры бакета нет**. Дефолт манифеста retention **336h**.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1"]
    T1["Микросервисы Tempo\n+ Kafka ingest + Memcached"]
  end
  subgraph b["ЦОД-2"]
    T2["Collector → ЦОД-1\nили отдельный Tempo"]
  end
  subgraph c["ЦОД-3"]
    T3["То же, не live-store\nчужого кластера"]
  end
  T1 -.->|"не 7946 / не 9095"| T2
  T1 -.-> T3
```

Не открывать memberlist между ЦОДами «чтобы один кластер». Не ставить второй scheduler в ЦОД-2.

**Сильное:** memberlist и ingest локальные. **Слабое:** падение ЦОД-1 = нет этого Tempo; второй кластер чужие блоки сам не подхватит.

---

## 8. Безопасность (ручки на той же схеме)

Встроенной auth **нет**. Gateway сам ставит OrgID после логина — `X-Scope-OrgID` не пароль. `multitenancy_enabled` дефолт **false**. `log_received_spans` не для прода. Ключи S3/SASL — Vault, не ConfigMap.

Источники: `Grafana Tempo.md`. Порога RTT у Tempo **нет**.
