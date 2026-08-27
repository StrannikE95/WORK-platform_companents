# Camunda 8.9 — схемы устройства

Связанные документы: правила — `Camunda 8.md`; установка — `Camunda 8.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. BPMN-схемы заявок не рисуем.

Допущения: Helm **`camunda-platform` 14.8.5** → образ **`camunda/camunda:8.9.17`**. Stretch брокеров на 2–3 ЦОДа **нет**. Официальный **dual-region** (RF=4, две площадки) — тоже stretch, **не выбран**. Целевой прод: **вариант C** — RF=3 **внутри одного ЦОДа** + Cold Recovery. Secondary — **OpenSearch 3.8**, не бизнес-поиск. Нагрузки нет.

---

## 1. Контекст

Camunda — **оркестратор длинных сценариев**, не шина и не SoT карточки.

```mermaid
flowchart LR
  API["Люди / API старт процесса"]
  ZB["Camunda 8.9\nZeebe"]
  WK["Job workers\nмикросервисы"]
  KF["Kafka шина"]
  PG["Озеро / PostgreSQL SoT"]
  INT["Интеграционное API"]

  API -->|"REST 8080 / gRPC 26500"| ZB
  ZB -->|"job"| WK
  WK -->|"complete / fail"| ZB
  WK --> KF
  WK --> PG
  WK --> INT
```

Воркеры **снаружи** JVM движка. Коннектор «напрямую в ведомство» плодит второй контур интеграций. Переменные процесса — id/статус, не XML досье (ориентир вендора ~**3 МБ** на экземпляр).

---

## 2. Контейнеры Orchestration Cluster

```mermaid
flowchart TB
  subgraph dc["ЦОД-1 = один Zeebe"]
    GW["Gateway\n8080 REST / 26500 gRPC"]
    subgraph brokers["Брокеры RF=3"]
      B1["broker"]
      B2["broker"]
      B3["broker"]
    end
    OP["Operate / Tasklist / Admin"]
    EXP["Camunda Exporter"]
  end

  OS["OpenSearch 3.8\nsecondary Camunda"]
  WK2["Workers Deployment"]
  BAK["Object storage\npartition backup"]

  WK2 --> GW
  GW -->|"26501 command"| brokers
  B1 <-->|"26502 Raft / Gossip"| B2
  B2 <--> B3
  EXP --> OS
  OP --> OS
  brokers --> BAK
```

С 8.8 gateway в Helm часто **embedded** в брокер; вход всё равно масштабируют репликами. Клиенты **не** ходят на 26501/26502. **9600** — actuator/metrics/backup API, не в интернет.

Bitnami Elasticsearch/PostgreSQL subchart в проде **выключены**. С 8.9 без `orchestration.data.secondaryStorage.type` чарт не ставится.

---

## 3. Компоненты данных

```mermaid
flowchart LR
  P["Партиция"] --> L["Лидер"]
  P --> F1["Follower"]
  P --> F2["Follower"]
  L -->|"commit кворум 2 из 3"| F1
  DISK["журнал + snapshot\nдиск брокера"]
  L --> DISK
```

Параллелизм ≈ **partitionCount**, не «ещё Operate». RF не больше числа брокеров. Тройка `clusterSize` / `partitionCount` / `replicationFactor` **одинаковая** на всех брокерах. HDD не поддерживается; PVC RWO, SSD, ≥1000 IOPS.

Optimize в первом проде **выкл** (×3–4 диск OS). Web Modeler / Identity — PostgreSQL рядом, не на пути исполнения.

---

## 4. Поток: complete job

```mermaid
sequenceDiagram
  participant W as Job worker
  participant G as Gateway 26500
  participant L as Лидер партиции
  participant F as Followers
  participant E as Exporter
  participant O as OpenSearch Camunda

  W->>G: activate / complete
  G->>L: 26501
  L->>F: Raft 26502
  Note over L,F: кворум внутри ЭТОГО ЦОДа
  F-->>L: ok
  L-->>W: committed
  L->>E: событие
  E->>O: индекс Operate
```

Падение OS: сначала Operate отстаёт, потом **backpressure** душит команды по кластеру. HA Zeebe без HA secondary — дырявая.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph inside["Вариант C — внутри ЦОД-1"]
    RF["clusterSize 3 RF=3"]
    AFF["anti-affinity / залы"]
    PDB["PDB: не снять 2 из 3"]
    BU["backup Zeebe+OS один ID"]
  end

  subgraph cold["Между ЦОДами — не Raft"]
    CR["Cold Recovery restore"]
    DC2["стенд restore ЦОД-2"]
  end

  inside -->|"падение 1 брокера"| OK["выборы лидера"]
  inside -->|"падение ЦОД-1"| cold
```

| Ручка | Если забыть |
|---|---|
| RF=3 в **одном** ЦОДе | Stretch 26502 = латентность каждого commit; порога RTT на 3 площадки нет |
| Dual-region | Stretch RF=4; OpenSearch **нельзя**; failover **ручной**; не выбран |
| Отдельный OS 3.8 | Экспорт в бизнес-поиск / Wazuh убивает оба контура |
| Прогнанный restore | Реплики не спасают от DROP PVC |
| Воркеры ≥2 на job type | Zeebe green, API лежит → процессы стоят на jobs |

Два мёртвых брокера из трёх снимают кворум. Dual-region при потере региона **сразу останавливает** обработку — это не «тихо пережили».

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> PI["экземпляры / сервисные задачи"]
  Q --> IN["вход воркеров"]
  Q --> VAR["размер variables"]
  Q --> UI["лаг Operate"]

  PI --> PC["partitionCount заранее"]
  IN --> GW2["реплики gateway"]
  VAR --> SMALL["id в процессе, тело в озере"]
  UI --> OS2["мощность OS Camunda"]
```

Лишние воркеры одного типа делят работы. Встроенной метрики backlog вендор **нет**. Gateway не ускорит партиции, которых нет. Терабайты — не в Zeebe.

---

## 7. 2–3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1 актив"]
    Z["Zeebe RF=3 + OS 3.8"]
  end
  subgraph b["ЦОД-2"]
    R["Cold Recovery"]
  end
  subgraph c["ЦОД-3"]
    B2["вторая копия бэкапа"]
  end
  W["Workers в живых ЦОДах"]

  W -->|"8080 / 26500 через город"| Z
  Z -->|"backup"| R
  Z --> B2
```

Третий ЦОД **не** член Raft. Клиенты в штате пишут в gateway ЦОД-1.

**Сильное:** commit не ждёт чужой ЦОД; OpenSearch 3.8 остаётся secondary. **Слабое:** смерть площадки движка = нет оркестрации, пока restore; RPO/RTO = как часто бэкап и как быстро его прогнали.

---

## 8. Безопасность

1. Лицензия `CAMUNDA_LICENSE_KEY` с 8.6; без ключа прод Self-Managed нелегален.
2. OIDC, не `demo:demo` / basic. Admin кластера ≠ Management Identity.
3. Ingress 8080/26500 TLS; 26501/26502 только брокеры+gateway (NetworkPolicy).
4. Helm `values-tls` **не** закрывает весь pod-to-pod gRPC — mesh отдельно.
5. L4 или gRPC-aware LB на 26500: HTTP L7 рвёт стримы воркеров.

Источники: `Camunda 8.md`. Dual-region и stretch 3 AZ на схемах не целевые.
