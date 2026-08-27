# Docker Engine 24.0.9 — схемы устройства

Связанные документы: правила — `Docker 24.md`; установка — `Docker 24.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код CLI не рисуем.

Допущения: линия **24.0 Unmaintained** (после 24.0.9 патчей нет; BuildKit CVE этой сборкой **не** закрыты). Прод-оркестратор — Kubernetes, CRI — **containerd**, не `dockerd`. Swarm Raft **не** путь. Stretch Swarm на 2–3 ЦОДа **нет**. Нагрузки нет.

---

## 1. Контекст

Engine 24 — **сборка и стенд**, не runtime прода и не шина.

```mermaid
flowchart LR
  DEV["Разработчик / CI"]
  DE["Docker Engine 24.0.9\nbuilder / Compose"]
  REG["Реестр образов"]
  K8S["Kubernetes 1.36\nCRI = containerd"]
  WL["Поды: Kafka, Camunda, API"]

  DEV -->|"docker build"| DE
  DE -->|"push OCI"| REG
  K8S -->|"pull"| REG
  K8S --> WL
  DE -.->|"не CRI прода"| K8S
```

Образы OCI можно собрать Docker'ом и запустить containerd. Это **не** значит, что на worker'ах нужен `dockerd`.

| Стрелка | Зачем помнить |
|---|---|
| CI → Engine | Сборка, не оркестрация 24/7 |
| Engine → реестр | Общее состояние сборки. Упал builder — бизнес-поды живы |
| K8s → containerd | dockershim убрали в Kubernetes **1.24**; cri-dockerd — антипаттерн |

---

## 2. Контейнеры (один хост = один демон)

```mermaid
flowchart TB
  CLI["docker CLI"]
  subgraph host["Builder / Dev Linux"]
    SOCK["Unix socket\n/var/run/docker.sock"]
    D["dockerd 24.0.9\nAPI 1.43"]
    CTD["containerd бандла"]
    RUNC["runc 1.1.12"]
    VOL["volumes + overlay2\n/var/lib/docker"]
  end
  IMG["Образ + контейнер"]

  CLI --> SOCK
  SOCK --> D
  D --> CTD
  CTD --> RUNC
  RUNC --> IMG
  IMG --> VOL
```

Один `dockerd` = один хост. Volume **не** реплицируется на соседнюю VM. NFS как data-root демона — путь к порче слоёв.

**Не на этой схеме:** Swarm manager Raft (2377 / overlay 4789). Это второй оркестратор рядом с Kubernetes. Целевой путь платформы — **вариант C**: Engine только на CI/builder.

---

## 3. Компоненты, которые путают с «кластером»

```mermaid
flowchart LR
  subgraph engine["Это Engine 24"]
    DJ["daemon.json"]
    OV["overlay2 слои"]
    LR["live-restore\nтолько standalone"]
    BK["BuildKit"]
  end

  subgraph not["Это не Engine"]
    SW["Swarm Raft\nне выбран"]
    CRI["CRI kubelet"]
    HARB["Harbor / registry"]
  end
```

`live-restore` спасает от рестарта **демона**, не от смерти машины. Со Swarm **несовместим**. containerd image store в 24.0 — experimental, в прод 24 **не** брать.

---

## 4. Поток: сборка образа

```mermaid
sequenceDiagram
  participant CI as CI / разработчик
  participant S as docker.sock
  participant D as dockerd
  participant R as Реестр
  participant K as kubelet + containerd

  CI->>S: docker build / push
  Note over S: кто пишет в сокет = root хоста
  S->>D: API 1.43
  D->>R: push digest
  K->>R: pull в прод
  Note over K: dockerd на worker нет
```

Порт **2375** без TLS = удалённый root. Не слушать. Удалённо — SSH или mTLS **2376**, ключи как root-пароль.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph one["Один dockerd"]
    L1["live-restore на стенде"]
    L2["диск / inode /var/lib/docker"]
    L3["prune по политике"]
  end

  subgraph farm["Сборка на 2–3 ЦОДа"]
    B1["независимый builder ЦОД-1"]
    B2["builder ЦОД-2"]
    B3["builder ЦОД-3"]
    RG["HA реестра\nдругое ПО"]
  end

  one -->|"падение VM"| DEAD["контейнеры хоста мертвы"]
  farm -->|"очередь CI"| RG
```

| Ручка | Если забыть |
|---|---|
| ≥2 builder в разных ЦОДах | Сборка = SPOF одной VM |
| Реестр живой | Builder жив, выкат в K8s встал |
| Не Swarm «чтобы HA» | Лишний Raft, UDP 4789, конфликт с kubelet |
| Не cri-dockerd на worker | Root-демон + EOL Engine под kubelet 1.36 |
| Ротация `json-file` | Дефолт без max-size заполняет диск |

Падение ЦОДа с одним builder'ом: очередь берёт другой. Выкат приложений — Kubernetes и реестр, не Docker.

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> CPU["CPU / диск слоёв"]
  Q --> NET["сеть до реестра"]
  Q --> JOB["параллельные job CI"]

  CPU --> H["больше независимых builder'ов"]
  NET --> REG2["зеркало / свой registry"]
  JOB --> LIM["cgroup memory/cpus\nне безлимитный build"]
```

Горизонталь = **N демонов**, не «один docker на 128 ядер» как единственная стратегия. Терабайты озера **не** в writable-слой overlay2. `userland-proxy` и iptables Docker конфликтуют с CNI — ещё одна причина не ставить Engine рядом с kubelet.

---

## 7. 2–3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph d1["ЦОД-1"]
    K1["K8s + containerd"]
    D1["dockerd builder"]
  end
  subgraph d2["ЦОД-2"]
    K2["свой K8s + containerd"]
    D2["свой builder"]
  end
  subgraph d3["ЦОД-3"]
    D3["третий builder"]
  end
  REGX["Реестр"]

  D1 --> REGX
  D2 --> REGX
  D3 --> REGX
  K1 --> REGX
  K2 --> REGX
```

Общего кворума Docker **нет** и не нужно. Stretch-Swarm 1-1-1 в презентацию **не** класть: порога RTT у Docker нет, оркестратор уже Kubernetes.

**Сильное:** падение builder'а не роняет Kafka/Camunda. **Слабое:** линия 24 без vendor-fix; ИБ может запретить 24 целиком.

---

## 8. Безопасность (сокет = root)

1. Только Unix-сокет; **2375 никогда**; группа `docker` поимённо.
2. Не монтировать сокет в CI-поды и приложения. DinD — изолированная VM, не сокет хоста.
3. Не `--privileged`, не seccomp `unconfined`. `no-new-privileges`.
4. Свой реестр, не Docker Hub из госконтура. Тег = digest, не `:latest`.
5. Честно: **24.0.9 не закрывает требование «безопасность»** вендорскими патчами. Компенсации — не замена ухода с линии.

Источники: `Docker 24.md`. Swarm на схемах — отказ, не целевой HA.
