# OpenSearch 3.8.0 — схемы устройства

Связанные документы: правила — `OpenSearch.md`; установка — `OpenSearch.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код ingest-пайплайна не рисуем.

Допущения: stretch transport **9300** на 2–3 ЦОДа **нет**; self-hosted 3.8.0, **не** Amazon OpenSearch Service и **не** Wazuh indexer; нагрузка не замерена.

---

## 1. Контекст

OpenSearch — **поиск и документная аналитика**. Near-real-time, не ACID-карточка, не Kafka.

```mermaid
flowchart LR
  KF["Kafka"]
  ING["Prepper / Connect / Fluent Bit"]
  OS["OpenSearch 3.8.0"]
  OSD["Dashboards\nдругое ПО"]
  WZ["Wazuh indexer"]
  LAKE["Озеро SoT"]

  KF --> ING
  ING -->|"bulk 9200"| OS
  OSD --> OS
  OS -.->|"не этот кластер"| WZ
  OS -.->|"копия для поиска"| LAKE
```

Склеить SIEM и клиентский поиск в одном Security realm — смешать роли. Буфер новых документов — **в Kafka**, не «Prepper не умрёт».

---

## 2. Контейнеры (кворум + data в одном ЦОДе)

«Три ноды в compose» ≠ HA. Кворум — **большинство cluster-manager-eligible**.

```mermaid
flowchart TB
  subgraph dc["ЦОД-1 = один кластер"]
    subgraph cm["Dedicated cluster_manager ×3"]
      M1["manager"]
      M2["manager"]
      M3["manager"]
    end
    subgraph data["Data + ingest"]
      D1["data"]
      D2["data"]
      D3["data"]
    end
  end

  CLI["Клиенты / Dashboards\nService 9200"]
  SNP["Snapshot repository"]

  CLI --> D1
  CLI --> D2
  D1 <-->|"transport 9300"| D2
  D1 <--> cm
  M1 <--> M2
  M2 <--> M3
  D1 --> SNP
```

**9200** REST (клиенты). **9300** нода↔нода и CCR. На dedicated manager внешний трафик **не** посылают. LB HTTP перед 9200 допустим; VIP «на все 9300» ломает transport.

Диск ноды — SSD POSIX, **не NFS**. PVC RWO на каждую ноду, не `emptyDir`.

**Сильное:** 3 manager переживают падение **одной** ноды. **Слабое:** два manager из трёх мертвы — кластер не выбирает manager. «Never loses quorum» в доке = одна зона из трёх, не две.

---

## 3. Компоненты индекса

```mermaid
flowchart LR
  IDX["Индекс"] --> P["Primary shard"]
  IDX --> R["Replica shard"]
  P --> N1["data-нода A"]
  R --> N2["data-нода B"]
```

Primary задаётся **при создании**; потом только reindex. Дефолт OSS: **1 primary + 1 replica**. На single-node replica **0**, иначе вечный yellow. Replica **не** бэкап: `DELETE index` повторится.

ISM — горячий → delete/snapshot, иначе диск съедят «терабайты» сами. Куча ~**½ RAM**, `Xms = Xmx`; лимит пода **выше** кучи (Lucene mmap). `vm.max_map_count ≥ 262144`.

---

## 4. Поток: bulk и поиск

```mermaid
sequenceDiagram
  participant In as Ingest / клиент
  participant C as Coordinating
  participant P as Primary
  participant R as Replica
  participant M as Cluster manager

  In->>C: bulk HTTPS 9200
  C->>P: индекс
  P->>R: копия шарда
  P-->>C: ack
  Note over M: аллокация и состояние — кворум manager этого ЦОДа
  In->>C: search
  C->>P: шарды
  C->>R: шарды
  C-->>In: hit
```

Green — все primary и replica на месте. Yellow — поиск ещё идёт без части replica. Red — нет primary.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    M3["3 dedicated manager"]
    RP["replica ≥ 1"]
    TLS["TLS 9300 + HTTP TLS 9200"]
    SN["Snapshot + restore прогнан"]
  end

  subgraph cross["Между ЦОДами"]
    CCR["CCR follower read-only"]
    SRCH["или клиенты на 9200 ЦОД-1"]
  end

  inside -->|"1 data-нода"| OK["индекс читается"]
  inside -->|"падение ЦОД-1"| cross
```

| Ручка | Если забыть |
|---|---|
| `initial_cluster_manager_nodes` один раз | Два UUID = split при старте |
| Demo certs выкл | *well known and therefore unsafe* |
| `plugins.security.ssl.http.enabled: true` | Дефолт plugin **false** — 9200 HTTP |
| Snapshot вне ЦОД-1 | Бэкап умер вместе с кластером |

CCR: плагин на **обоих**; follower **не** становится writer сам. Security включён на обоих или выключен на обоих.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Ось"]
  Q --> V["Объём / поиск"]
  Q --> I["Индексация"]
  Q --> A["Тяжёлые aggregations"]

  V --> V1["Data-ноды + размер шарда\nбенчмарк, не слайд"]
  I --> I1["Bulk, не одиночный POST"]
  A --> A1["Coordinating, чтобы не душить heap data"]
```

Manager почти не масштабируют «от QPS»: им RAM под cluster state (много мелких индексов = толстый state).

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1 leader"]
    OS1["Кворум + data + запись"]
  end
  subgraph b["ЦОД-2"]
    OS2["CCR follower или только 9200→ЦОД-1"]
  end
  subgraph c["ЦОД-3"]
    OS3["Ещё follower / snapshot"]
  end
  OS1 -->|"CCR async RPO>0"| OS2
  OS1 -.->|"не 9300 кворума"| OS3
```

**Сильное:** cluster state не ходит по WAN. **Слабое:** падение ЦОД-1 = нет записи; без follower нет и поиска; промоушен CCR — runbook.

---

## 8. Безопасность (ручки на той же схеме)

С 2.12 без `OPENSEARCH_INITIAL_ADMIN_PASSWORD` demo-конфиг не стартует. `DISABLE_SECURITY_PLUGIN` — лаборатория. Transport TLS обязателен при Security plugin. Dual-mode SSL — только миграция. `enforce_hostname_verification: true`. В 3.8.0 дефолт snapshot SSE **AES256** — шлюз может отвергнуть, тогда явно `bucket_default`.

Источники: `OpenSearch.md`. Порога RTT на 9300 у проекта **нет**.
