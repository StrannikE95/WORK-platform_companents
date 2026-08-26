# Prometheus 3.13.2 LTS — схемы устройства

Связанные документы: правила — `Prometheus.md`; установка — `Prometheus.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код экспортёров не рисуем.

Допущения: stretch scrape и gossip Alertmanager на 2–3 ЦОДа **нет**; **не** кластерный TSDB; kube-prometheus-stack **88.3.0** (Operator 0.93.0, Alertmanager 0.33.1); не подмешивать latest 3.14.0; нагрузка не замерена.

---

## 1. Контекст

Prometheus — **pull-метрики и правила**. Не логи, не шина, не биллинг с точностью до копейки.

```mermaid
flowchart LR
  TGT["Kafka / Camunda / API / ноды"]
  PR["Prometheus 3.13.2\nлокальный TSDB"]
  AM["Alertmanager"]
  GF["Grafana\nдругое ПО"]

  PR -->|"scrape /metrics"| TGT
  PR -->|"алерты на ВСЕ AM\nне через LB"| AM
  GF -->|"PromQL"| PR
```

Проект: надёжность важнее биллинговой точности. `user_id` / UUID / ИНН в label — классический способ **убить** инстанс кардинальностью.

---

## 2. Контейнеры (экосистема, не один TSDB)

Две HA-реплики Prometheus — **два независимых диска**, одинаковый scrape. Это не шарды одной базы.

```mermaid
flowchart TB
  subgraph dc["Один ЦОД = свой контур"]
    PA["Prometheus replica A\nexternal_labels"]
    PB["Prometheus replica B"]
    subgraph amc["Alertmanager cluster"]
      AM1["AM"]
      AM2["AM"]
      AM3["AM"]
    end
    NE["node_exporter DaemonSet"]
    KS["kube-state-metrics"]
    OP["Operator 1 под"]
  end

  UI["Grafana / API :9090"]

  UI --> PA
  UI --> PB
  PA -->|"alerts"| AM1
  PA --> AM2
  PA --> AM3
  PB --> AM1
  PB --> AM2
  PB --> AM3
  AM1 <-->|"gossip 9094 TCP+UDP"| AM2
  AM2 <--> AM3
  PA -->|"pull независимо"| NE
  PB -->|"pull независимо"| NE
  PA --> KS
  PB --> KS
  OP -.->|"новые CR"| PA
```

Порты: **9090** Prometheus, **9093** AM UI, **9094** gossip, **9100** node_exporter. Диск TSDB — block CSI, **не NFS** (официальный CAUTION). У каждой реплики **свой** PVC.

**Сильное:** падение одной реплики — вторая продолжает снимать. **Слабое:** PromQL на A и B чуть расходятся; без Thanos графики «прыгают». Thanos — отдельный проект, не обязателен «чтобы считать Prometheus поставленным».

---

## 3. Компоненты внутри сервера

```mermaid
flowchart TB
  subgraph srv["Один процесс Prometheus"]
    SC["scrape + relabel"]
    HD["Head в RAM"]
    WAL["WAL + блоки 2ч"]
    RL["recording / alerting rules"]
  end

  DISK["Локальный POSIX\nне кластер"]
  AMX["Alertmanager"]

  SC --> HD
  HD --> WAL
  WAL --> DISK
  RL --> HD
  RL --> AMX
```

| Компонент | Для чего настраивать |
|---|---|
| Labels / relabel | Главный фильтр мусора **до** записи |
| `scrape_timeout` | Дефолт **10s**. Чужой ЦОД съест окно → ложный `DOWN` |
| Retention | Без флагов **15d**; чарт 88.3.0 — **10d**. `retention.size` ≤ 80–85% PVC |
| `sample_limit` / `label_limit` | Лучше отвалить цель, чем положить head |

Operator падение: уже запущенный scrape жив; новые ServiceMonitor/Rule не применятся.

---

## 4. Поток: scrape и алерт

```mermaid
sequenceDiagram
  participant P as Prometheus replica
  participant T as Цель /metrics
  participant R as Rules
  participant A as Все Alertmanager

  P->>T: GET scrape_interval
  T-->>P: samples
  P->>P: WAL / head
  P->>R: evaluation
  Note over R: for: держит шум одного плохого scrape
  R->>A: firing на каждый AM
  Note over A: fail-open: split = дубли писем, не тишина
```

LB **между** Prometheus и Alertmanager — официальный anti-pattern.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    R2["Prometheus replicas: 2\nразные external_labels"]
    AMN["AM 2–3 все здесь"]
    PVC["свой PVC на под\nне NFS"]
    PDB["PDB / drain"]
  end

  subgraph cross["Между ЦОДами"]
    FED["Federation /federate\nагрегаты, узкий match"]
  end

  inside -->|"падение пода"| OK["съём со второй реплики"]
  inside -->|"падение ЦОД-1"| cross
```

| Ручка | Если забыть |
|---|---|
| `replicas: 2` (дефолт чарта **1**) | Выкат = дырка съёма |
| Алерты на все AM, не Ingress | SPOF и ломает дедуп |
| `replicaExternalLabelNameClear: false` | Чарт запрещает чистить для HA |
| Снимок TSDB | `cp` живого каталога + NFS = порча |

Падение двух ЦОДов при мозге в одном — нет детальных рядов этой площадки. AM gossip **не** растягивать: probe 1s, порога RTT нет.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  M["Упёрлись"]
  M --> C["Кардинальность"]
  M --> H["RAM / PromQL"]
  M --> D["Диск"]

  C --> C1["Убрать UUID из labels\nэффективнее, чем удлинить interval"]
  H --> H1["Вертикаль или functional shard\nне автошард с порога"]
  D --> D1["needed_disk ≈ retention_s × samples/s × 1…2 байта"]
```

FAQ: один инстанс тянет *tens of millions* active series — потолок **порядка**, не ваш замер. Озеро клиентов и TSDB — разные объёмы.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1"]
    L1["Пара Prometheus + AM"]
    GL["Federator агрегатов"]
  end
  subgraph dc2["ЦОД-2"]
    L2["Свой контур scrape"]
  end
  subgraph dc3["ЦОД-3"]
    L3["Свой контур"]
  end
  L2 -->|"/federate узкий match"| GL
  L3 --> GL
```

Один Prometheus, который scrape'ит три Kubernetes, при плохом RTT рисует `DOWN` живых целей. Третий ЦОД **не** входит в gossip AM ЦОД-1.

**Сильное:** съём локальный — авария канала ≠ ложный DOWN всего. **Слабое:** два слоя правил; широкий `match[]` = второй полный TSDB; federator — SPOF картины (его тоже HA внутри его ЦОДа).

---

## 8. Безопасность (ручки на той же схеме)

Кто открыл HTTP 9090 — видит **все** ряды (модель проекта). 9090/9093 не в интернет. TLS + bcrypt или mTLS; basic без TLS модель запрещает смыслом. Admin/lifecycle API дефолт выкл. `honor_labels` + открытый Pushgateway = подмена рядов. Метрики не секрет: ПДн в label — ваша утечка.

Источники: `Prometheus.md`. Порога RTT у проекта **нет**.
