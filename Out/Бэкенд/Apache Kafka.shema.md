# Apache Kafka 4.3.1 — схемы устройства

Связанные документы: правила — `Apache Kafka.md`; установка — `Apache Kafka.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код клиентов не рисуем.

Допущения: stretch брокеров на 2–3 ЦОДа **нет**; ZooKeeper нет (KRaft); Java ≥ 17 на брокерах; нагрузка не замерена.

---

## 1. Контекст

Kafka — **лог событий**, не источник истины карточки и не Camunda.

```mermaid
flowchart LR
  MS["Микросервисы"]
  CAM["Camunda workers"]
  INT["Интеграционное API"]
  KF["Kafka 4.3.1\nшина"]
  PG["PostgreSQL SoT"]
  LK["Озеро / ClickHouse"]

  MS -->|"produce / consume"| KF
  CAM --> KF
  INT --> KF
  PG -->|"outbox: факт"| KF
  KF -->|"сырьё, аудит"| LK
```

Несколько независимых читателей одного топика **без** повторной записи в каждый. SoT «дай клиента по ИНН» — не Kafka.

---

## 2. Контейнеры (одно ПО, разные процессы)

```mermaid
flowchart TB
  subgraph dc["Один ЦОД = один кластер Kafka"]
    subgraph ctrl["KRaft controllers\nкворум метаданных"]
      C1["controller"]
      C2["controller"]
      C3["controller"]
    end
    subgraph br["Брокеры\nлоги топиков"]
      B1["broker rack=зал"]
      B2["broker"]
      B3["broker"]
    end
    MM["MirrorMaker 2\nдругое ПО рядом\nмежду ЦОДами"]
  end

  PR["Продюсер"]
  CS["Консьюмер / group"]

  PR -->|"bootstrap затем лидер партиции"| B1
  CS --> B2
  B1 <-->|"replication / ISR"| B2
  B1 <--> B3
  C1 <--> C2
  C2 <--> C3
  C1 -->|"метаданные"| br
  B1 -.->|"не stretch"| MM
```

Клиент **не** ходит на все брокеры через HTTP-LB: он узнаёт кластер с bootstrap, пишет в **лидера партиции**. Combined mode (брокер+контроллер в одном процессе) — только Dev.

---

## 3. Компоненты данных

```mermaid
flowchart LR
  T["Топик"] --> P1["Партиция 0"]
  T --> P2["Партиция 1"]
  P1 --> L["Лидер на брокере A"]
  P1 --> F1["Replica на B"]
  P1 --> F2["Replica на C"]
  L -->|"ISR"| F1
  L --> F2
```

Параллелизм = **число партиций**, не «ещё один Kafka». RF=3 / min.ISR=2: запись с `acks=all` ждёт две живые копии **внутри ЦОДа**.

---

## 4. Поток produce

```mermaid
sequenceDiagram
  participant P as Продюсер acks=all
  participant L as Лидер партиции
  participant R as Followers ISR

  P->>L: запись
  L->>R: репликация
  R-->>L: в ISR
  Note over L: ISR не меньше min.ISR
  L-->>P: OK
```

`unclean.leader.election=false`: отставшую replica лидером не делают ценой потери уже подтверждённого.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph must["Обязательные ручки внутри ЦОДа"]
    ISO["isolated controllers\nне combined"]
    Q["3 или 5 контроллеров"]
    RF["default.RF=3 min.ISR=2"]
    RACK["broker.rack = зал/стойка"]
    UCE["unclean election off"]
  end

  subgraph cross["Между ЦОДами"]
    MM2["MM2 async\nRPO больше 0, дубли, другие offsets"]
  end

  must -->|"падение 1 брокера"| OK["запись жива если min.ISR=2"]
  must -->|"падение ЦОДа"| cross
```

| Ручка | Если забыть |
|---|---|
| `broker.rack` | Три копии в одном шкафу |
| min.ISR=3 при RF=3 | Один мёртвый брокер останавливает запись |
| auto.create.topics | Топики с RF=1 |
| offsets.topic RF=3 | Падение брокера ломает consumer groups |
| PDB / Drain Cleaner | Kubernetes убивает лидеров «вежливо» |

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart TB
  M["Упёрлись"]
  M --> Tput["MB/s / msg/s"]
  M --> Cons["Число консьюмеров"]
  M --> Disk["Retention / диск"]

  Tput --> ADD["Брокеры пачками внутри ЦОДа\nзатем reassign партиций"]
  Cons --> PART["Больше партиций\nлишние воркеры без партиций простаивают"]
  Disk --> RET["log.retention / compaction\nопционально tiered storage"]
```

Новые брокеры **пусты**, пока партиции не переложены. Куча JVM ~ориентир 6 ГиБ в примере Apache; остальная RAM — **page cache**, не `-Xmx32g`.

TLS бьёт по CPU и zero-copy — запас ядер, не «та же скорость».

---

## 7. 2–3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1"]
    K1["Кластер KRaft\nклиенты пишут сюда"]
  end
  subgraph b["ЦОД-2"]
    K2["Свой кластер"]
  end
  subgraph c["ЦОД-3"]
    K3["Свой кластер или только sink"]
  end
  K1 -->|"MM2"| K2
  K1 -->|"MM2"| K3
```

Один `bootstrap.servers` «на страну» при запрете stretch **нет**. Failover площадки — runbook и контракт топиков, не автомагия.

**Сильное:** локальный `acks=all` не ждёт чужой ЦОД. **Слабое:** RPO≠0, offsets не те же, MM2 сам должен быть HA.

---

## 8. Безопасность на той же картине

Отдельные listener'ы: CONTROLLER / inter-broker / CLIENT. CLIENT = SASL_SSL. StandardAuthorizer: нет ACL → нет доступа. SCRAM без TLS вендор считает опасным.

Источники: `Apache Kafka.md`. Порога RTT для stretch у Apache **нет**.
