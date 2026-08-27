# GitLab 19.3.0 — схемы устройства

Связанные документы: правила — `GitLab CI.md`; установка — `GitLab CI.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. YAML джобов не рисуем.

Допущения: GitLab **19.3.0** + Runner **19.3.0**, Helm **`gitlab/gitlab` 10.3.0**. Stretch Gitaly/PostgreSQL/Redis на 2–3 ЦОДа **нет** (вендор HA **< 5 ms**, Praefect — *single location*). Bundled Postgres чарта **не** reference architecture S. Нагрузки/RPS нет.

---

## 1. Контекст

Отдельного продукта «только CI» нет. CI — подсистема GitLab. Пайплайн **поставляет** код, не держит runtime.

```mermaid
flowchart LR
  DEV["Разработчик / MR"]
  GL["GitLab 19.3\nкоординатор"]
  RN["Runner 19.3\nkubernetes-executor"]
  APP["Kafka / Camunda / API\nрантайм 24/7"]
  SQ["SonarQube в джобе"]

  DEV -->|"git push"| GL
  GL -->|"job"| RN
  RN -->|"clone / test / image"| GL
  RN --> SQ
  RN -->|"deploy"| APP
  GL -.->|"падение ≠ простой бизнеса"| APP
```

Runner без живого GitLab/Gitaly/бакета = очередь `pending`. Живые сервисы при мёртвом GitLab **не** должны падать.

---

## 2. Контейнеры координатора

Cloud Native Hybrid: stateless в Kubernetes; Gitaly Cluster на VM; PostgreSQL, Redis, object storage — **снаружи**.

```mermaid
flowchart TB
  subgraph k8s["K8s ЦОД-1 — stateless"]
    WS["Webservice / Workhorse\n443 API и UI"]
    SK["Sidekiq\nочереди CI / логи"]
    REG["Container Registry"]
  end

  subgraph state["Состояние — тот же ЦОД"]
    GT["Gitaly Cluster / Praefect\nVM, не шарды как HA"]
    PG["PostgreSQL writer"]
    RD["Redis Sentinel\nне Cluster"]
    S3["Object storage\nartifacts / logs / LFS"]
  end

  RN2["Runner managers"]
  JOB["Job pods"]

  RN2 -->|"poll 443"| WS
  WS --> PG
  WS --> RD
  SK --> RD
  SK --> S3
  WS --> GT
  REG --> S3
  RN2 -->|"K8s API"| JOB
  JOB -->|"clone"| GT
```

Порты: **443** Runner↔GitLab; Gitaly **8075** (TLS 9999); Praefect **2305**; PG **5432**; Redis **6379** / Sentinel **26379**; Registry **5050/5000**. Incremental logging: куски лога в Redis → бакет. Иначе мультинодовый Rails теряет трейс.

**Дефолтный Helm** с bundled Postgres/Redis/MinIO — PoC. Цитата чарта: *not intended for production*.

---

## 3. Компоненты, которые путают с HA

```mermaid
flowchart LR
  subgraph gitlab["Это GitLab CI"]
    YML[".gitlab-ci.yml"]
    TOK["glrt- токен Runner"]
    ART["artifacts в S3"]
  end

  subgraph not["Это не GitLab"]
    KF["Kafka шина"]
    CAM["Camunda BPMN"]
    GEO["Geo — Premium"]
  end
```

Gitaly в Cloud Native K8s — **шарды**: каждый под = SPOF **своих** репо. Praefect в K8s — **beta**. Redis Cluster **не поддерживается** (в т.ч. incremental logging).

---

## 4. Поток: джоб

```mermaid
sequenceDiagram
  participant U as git push
  participant WS as Webservice
  participant G as Gitaly
  participant R as Runner manager
  participant P as Job pod

  U->>WS: pipeline
  WS->>G: refs
  R->>WS: есть работа?
  WS-->>R: job
  R->>P: создать под
  P->>G: clone 443 или 22
  P->>WS: лог / артефакт
  Note over WS: Sidekiq архивирует лог в бакет
```

`CI_JOB_TOKEN` живёт, пока джоб. Allowlist проектов, иначе токен ходит шире группы.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph dc1["Внутри ЦОД-1"]
    W2[">=2 Webservice"]
    PRA["Praefect RF=3 на VM"]
    PGH["HA writer PG"]
    RS["Redis Sentinel x2 роли"]
    BUCKET["бакет переживает 1 диск"]
  end

  subgraph other["Другие ЦОДы"]
    RUN["Runner managers"]
    DR["бэкап + restore\nGeo только Premium"]
  end

  dc1 -->|"падение 1 Webservice"| OK["LB, Runner переподключится"]
  dc1 -->|"падение ЦОД-1"| other
```

| Ручка | Если забыть |
|---|---|
| Внешние PG/Redis/S3 | Bundled chart ≠ размер S референса |
| Praefect на VM | Sharded Gitaly в K8s — простой репо при рестарте STS |
| Incremental logging + бакет | Лог принял один Rails, Sidekiq на другом — дыра |
| Runner в нескольких ЦОДах | Падение зоны исполнения = все джобы в полёте там мертвы |
| Geo без Premium | Штатного межплощадочного DR инстанса **нет** |

Падение writer PG / Gitaly / бакета = нет CI, сколько ни плодь Runner'ов. Переживание **двух** ЦОДов не обещать.

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart TB
  M["Упёрлись"]
  M --> J["параллельные пайплайны"]
  M --> C["clone больших репо"]
  M --> A["артефакты / логи"]

  J --> R1["managers + concurrent + ноды job"]
  C --> G1["Gitaly CPU/диск; shallow clone"]
  A --> S1["retention expire_in + Sidekiq delete"]
```

Ось №1 CI — job-поды, не «ещё Kafka». HPA Webservice не лечит монорепу. Таблица RPS S/M/L — ориентир вендора, не смета. `concurrent` дефолт чарта **20** — не ёмкость кластера.

Сборка образов: **BuildKit rootless**, не privileged DinD на нодах приложений. Kaniko Google архивирован (июнь 2025) — не стратегия.

---

## 7. 2–3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1 актив"]
    HY["Hybrid: WS/Sidekiq\nPraefect + PG + Redis"]
  end
  subgraph b["ЦОД-2"]
    R2["Runner + job pods"]
    BK["restore / Geo secondary"]
  end
  subgraph c["ЦОД-3"]
    R3["ещё Runner"]
  end
  HY -->|"poll 443"| R2
  HY --> R3
  HY -.->|"не stretch 8075"| BK
```

Один environment GitLab **между регионами** вендор не поддерживает. Три независимых GitLab = три SoT кода.

**Сильное:** латентность git/PG не прибита межЦОДовым RTT. **Слабое:** смерть ЦОД-1 = нет координатора, пока restore (или ручной Geo failover при Premium).

---

## 8. Безопасность

1. Нет legacy registration token (снятие в **20.0**); только `glrt-`.
2. 443 с нужных сетей; 8075/PG/Redis не с мира.
3. Job-namespace: NetworkPolicy, PSA; `deploy-prod` — protected runner.
4. Секреты — Protected/Masked, лучше Vault; scope `CI_JOB_TOKEN`.
5. Pin **10.3.x / 19.3.x**; `gitlab-runner.install` в проде выключить или полностью свои values.

Источники: `GitLab CI.md`. Порог **5 ms** — общее требование HA-сети вендора; на схемах stretch не целевой.
