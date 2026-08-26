# ClickHouse 26.7.5.10 — схемы устройства

Связанные документы: правила — `ClickHouse.md`; установка — `ClickHouse.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код SQL клиентов не рисуем.

Допущения: stretch Raft Keeper **9234** и interserver **9009** на 2–3 ЦОДа **нет**; OSS 26.7.5.10 + Altinity Operator **0.27.3**, не Cloud/Private; не OLTP; нагрузка не замерена.

---

## 1. Контекст

ClickHouse — **колоночный OLAP**. Не карточка клиента, не шина, не Camunda.

```mermaid
flowchart LR
  KF["Kafka"]
  SN["Connect Sink\nexactlyOnce в проде"]
  CH["ClickHouse 26.7.5.10\nвитрины, агрегаты"]
  PG["PostgreSQL SoT"]
  BI["Grafana / Superset"]

  KF --> SN
  SN -->|"батч INSERT"| CH
  CH -.->|"не замена карточки"| PG
  BI -->|"SQL / HTTP"| CH
```

Точечный `UPDATE` по ИНН и многотабличные транзакции здесь живут плохо. Мутации дорогие. Kafka engine на сервере — **at-least-once**; exactly-once — у Connect + KeeperMap.

---

## 2. Контейнеры (один кластер = один ЦОД)

Три контейнера с голым MergeTree — три разные базы, не HA.

```mermaid
flowchart TB
  subgraph dc["ЦОД-1"]
    subgraph kep["Dedicated Keeper ×3"]
      K1["Keeper"]
      K2["Keeper"]
      K3["Keeper"]
    end
    subgraph sh["1 шард × 3 реплики"]
      R1["clickhouse-server"]
      R2["clickhouse-server"]
      R3["clickhouse-server"]
    end
  end

  CLI["Клиенты 8443 / 9440"]
  BAK["BACKUP TO S3"]

  CLI --> R1
  CLI --> R2
  R1 <-->|"9009 / 9010 parts"| R2
  R1 <--> R3
  R1 -->|"9181 / 9281"| kep
  K1 <-->|"Raft 9234"| K2
  K2 <--> K3
  R1 --> BAK
```

Клиенты: **8123/8443** HTTP, **9000/9440** native. **На Keeper и на 9009 внешний трафик не пускают.** VIP «на все 9009» ломает копирование parts. Диск — local SSD, не NFS, не `emptyDir` для `/var/lib/clickhouse`.

**Сильное:** падение одной реплики при живом кворуме — чтение живо. **Слабое:** дефолт INSERT подтверждает **одну** реплику; блок может пропасть до копирования.

---

## 3. Компоненты таблиц

```mermaid
flowchart LR
  D["Distributed\nпрокси, сама не хранит"]
  D --> S1["Шард 0 ReplicatedMergeTree"]
  D --> S2["Шард 1"]
  S1 --> C1["Реплика A"]
  S1 --> C2["Реплика B"]
  S1 --> C3["Реплика C"]
```

`ON CLUSTER` + макросы `{shard}`/`{replica}` — иначе `CREATE` только на той ноде, куда подключились. `internal_replication=true`: Distributed пишет в **одну** здоровую реплику, дальше копирует ReplicatedMergeTree.

`insert_quorum=2` (или `'auto'` при 3 репликах): клиент ждёт большинство. `insert_quorum=3` остановит запись при падении одной реплики.

---

## 4. Поток INSERT с кворумом

```mermaid
sequenceDiagram
  participant App as Sink / клиент
  participant R as Реплика-приёмник
  participant K as Keeper
  participant O as Другая реплика

  App->>R: INSERT батч
  R->>K: метаданные блока
  R->>O: parts 9009
  Note over R,O: quorum=2: COMMIT ждёт вторую копию
  O-->>R: ок
  R-->>App: успех
```

Без quorum: OK клиенту **до** догона. Реплика не бэкап: `DROP TABLE` уедет на всех. XML-учётки не входят в `BACKUP` — прод SQL-driven.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    K3["3 dedicated Keeper"]
    R3["3 реплики шарда"]
    IQ["insert_quorum=2"]
    PDB["PDB: не два Keeper сразу"]
  end

  subgraph dr["Между ЦОДами — не 9009/9234"]
    RS["RESTORE из бакета\nили свой кластер"]
  end

  inside -->|"1 реплика / 1 Keeper"| OK["чтение и запись живы"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если забыть |
|---|---|
| Dedicated Keeper | Тяжёлый merge бьёт по fsync Raft → DDL и репликация «странные» |
| `internal_replication=true` | Distributed сам пишет во все реплики — расхождения |
| Бакет переживает ЦОД-1 | Бэкап умер вместе с кластером |
| Один мажор 26.7 | Мешать с LTS 26.3 в одном кластере нельзя |

Два мёртвых Keeper из трёх — нет лидера, ReplicatedMergeTree **read-only**. Не класть «все Keeper в ЦОД-1, данные в трёх»: падение мозга = данные без координации.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> V["CPU/RAM/диск одной реплики"]
  Q --> S["Объём / скан"]
  Q --> P["Мелкие INSERT / parts"]

  V --> V1["Сначала вертикаль\nвендор: не шардировать рано"]
  S --> S1["Шарды, когда машина мала\nрешардинг — отдельный проект"]
  P --> P1["Батчи; async insert с 26.3 дефолтен\nстрока из REST всё равно душит"]
```

TTL и `ORDER BY` с первого дня: ключ сортировки дешево не поменять. Ориентиры RAM:диск вендора — не смета вашего прода.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1"]
    CL["Keeper 3 + shard 1×3"]
  end
  subgraph dc2["ЦОД-2"]
    DR1["RESTORE или свой кластер\nнет единого SELECT без слоя"]
  end
  subgraph dc3["ЦОД-3"]
    DR2["То же; не третий Keeper\nчужого кворума"]
  end
  CL -->|"S3 BACKUP"| DR1
  CL --> DR2
```

Heartbeat Keeper дефолт **500 мс**. Если p99 RTT сопоставим — ложные выборы. Порога в доке **нет**, поэтому 9009/9234 между ЦОДами **не** рисуем.

**Сильное:** Raft локальный. **Слабое:** падение ЦОД-1 = нет аналитики, пока restore; RPO > 0; второй кластер — вторая правда таблиц.

---

## 8. Безопасность (ручки на той же схеме)

Нет `CLICKHOUSE_SKIP_USER_SETUP`. `default` без пароля в сеть — дыра. Прод: 8443/9440, interserver 9010, Keeper TLS. `secret` в `remote_servers`, иначе чужой 9000 прикинется шардом. Эмуляции 9004/9005/9100 выключить. В 26.7 пользовательский SQL по умолчанию **не** берёт server-side S3 credentials.

Источники: `ClickHouse.md`. Порога RTT у проекта **нет**.
