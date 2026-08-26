# Falco 0.44.1 — схемы устройства

Связанные документы: правила — `Falco.md`; установка — `Falco.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код eBPF не рисуем.

Допущения:

1. Глобального кластера Falco на 2–3 ЦОДа **нет**. Агент живёт на ноде; stretch не применим.
2. Open-source **0.44.1**, Operator v0.4.1 (K8s ≥ 1.29) или Helm `falco` 9.1.0. Драйвер прода — **modern eBPF**.
3. Нагрузки syscall нет — на схемах нет «N millicores». Есть *что крутить*.

---

## 1. Контекст (C4 system context)

Falco — **runtime-детектор syscalls**. Алертит, не блокирует пакет и не заменяет WAF / NetworkPolicy / сканер CVE.

```mermaid
flowchart LR
  subgraph nodes["Ноды Kubernetes"]
    WL["Поды: API, Kafka, Camunda"]
  end

  FC["Falco 0.44.1\nDaemonSet на ноде"]
  SK["Falcosidekick"]
  SIEM["SIEM / on-call"]
  WZ["Wazuh\nлоги / FIM"]
  WF["WAF\nHTTP периметр"]

  WL -->|"syscall"| FC
  FC -->|"JSON HTTP"| SK
  SK --> SIEM
  WZ -.->|"другой слой"| FC
  WF -.->|"не syscall"| FC
```

| Стрелка | Зачем помнить |
|---|---|
| Ядро → Falco | Поток syscalls этой машины. Чужую ноду агент не видит |
| Falco → Sidekick → SIEM | Без приёмника это логи на ноде, не безопасность |
| Wazuh рядом | Хостовые логи/FIM; syscall-поток — у Falco |

**Слабое место контекста:** «один Falco на страну» не существует. Ставить **в каждый** Kubernetes.

---

## 2. Контейнеры (агент, не кворум)

Единица = **DaemonSet**: по одному поду на ноду. Deployment с `replicas: 2` для ядра **не** покрывает ноды и даёт ложное HA.

```mermaid
flowchart TB
  subgraph k8s["Один Kubernetes = свой контур Falco"]
    OP["Operator / Helm\nCRD правил"]
    subgraph ds["DaemonSet modern eBPF"]
      F1["falco на ноде A"]
      F2["falco на ноде B"]
    end
    META["k8s-metacollector\nобогащение подов"]
    SK2["Falcosidekick 2+"]
  end

  SI["SIEM"]
  LOG["stdout / syslog ноды"]

  OP --> ds
  F1 --> META
  F1 -->|"http_output TLS"| SK2
  F1 --> LOG
  SK2 --> SI
```

Падение пода Falco = **эта нода слепая**, нагрузка жива. Падение ЦОДа = нет и нагрузки, и детекта площадки. «Выборов лидера Falco» нет.

**Сильное:** живые ноды других кластеров мониторятся независимо. **Слабое:** без tolerations слепы Kafka/control-plane — самые важные машины.

---

## 3. Компоненты внутри агента

```mermaid
flowchart TB
  subgraph agent["Один процесс falco"]
    DRV["modern eBPF\nкольцевой буфер"]
    USR["userspace правила\nоднопоточный разбор"]
    PLG["plugin container + k8smeta"]
    OUT["stdout / syslog / HTTP"]
  end

  KER["Ядро Linux BTF / ringbuf"]
  CRI["сокет containerd"]

  KER --> DRV
  CRI --> PLG
  DRV --> USR
  PLG --> USR
  USR --> OUT
```

| Компонент | Для чего настраивать |
|---|---|
| modern eBPF | Default 0.44; legacy eBPF **удалён**. Нужны BTF + ringbuf (обычно ядро ≥ 5.8) |
| container plugin | Без него поля `container.*` в штатных правилах мертвы |
| Правила YAML / OCI | Макросы, priority, overlay exceptions под JVM/Kafka |
| buf_size_preset | Больше буфер → меньше дропов, больше RAM |
| kmod | Только если eBPF физически не встаёт; тогда full privileges |

Falco **не убивает** под. Talon — другой проект, в 0.44.1 не входит.

---

## 4. Поток: syscall → алерт

```mermaid
sequenceDiagram
  participant P as Процесс в поде
  participant K as Ядро / eBPF
  participant F as falco userspace
  participant S as Falcosidekick
  participant M as SIEM

  P->>K: syscall (exec, open, connect)
  K->>F: событие в ring buffer
  alt буфер полон
    K-->>K: drop — Falco не видел
  else событие дошло
    F->>F: rule match + container/k8s meta
    F->>S: HTTPS JSON
    S->>M: маршрутизация
  end
```

Дроп — слепая зона, не оптимизация. `syscall_event_drops: ignore` в проде нельзя. Self-signed в `http_output` Falco **не** принимает.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph local["Внутри каждого K8s"]
    DS["DaemonSet все Linux-ноды\ntolerations"]
    SKN["Sidekick 2+ и PDB"]
    CH2["два канала: HTTP + syslog"]
  end

  subgraph islands["2-3 ЦОДа"]
    K1["Falco в кластере ЦОД-1"]
    K2["Falco в кластере ЦОД-2"]
    SOC["Один SIEM на всех"]
  end

  local --> islands
  K1 --> SOC
  K2 --> SOC
```

| Ручка | Если не настроить |
|---|---|
| DaemonSet, не Deployment | Часть нод без агента = постоянная слепота |
| Tolerations на tainted | Dedicated Kafka без Falco |
| Sidekick ≥ 2 | Выкат приёмника = потеря HTTP-алертов |
| Запасной syslog | Смерть Sidekick не должна обнулить след |
| Низкий maxUnavailable | Rolling update = окно атаки на слепых нодах |

Оператор ≥ 2 реплик: уже запущенный DaemonSet переживает простой Operator; ломается выкат правил.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> N["Число нод"]
  Q --> D["Дропы scap.n_drops"]
  Q --> R["Шум правил"]

  N --> N1["DaemonSet сам добавит агент\nне второй процесс на ноду"]
  D --> D1["Сузить base_syscalls\nзатем buf_size_preset"]
  R --> R1["Overlay exceptions\nне глушить детект целиком"]
```

Масштаб = **число нод**, не «ещё один кластер Falco». Userspace однопоточен; ориентир troubleshooting ~1–1.5k events/s на CPU обычно терпимо, &gt;~3k часто тяжело — не смета, зерно соли. Терабайты озера почти не влияют; влияет syscall-шум Kafka/Java.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 Kubernetes"]
    FDC1["DaemonSet Falco"]
  end
  subgraph dc2["ЦОД-2 Kubernetes"]
    FDC2["свой Operator + DaemonSet"]
  end
  subgraph dc3["ЦОД-3"]
    FDC3["свой контур"]
  end
  SIEM["Общий SIEM"]

  FDC1 -->|"алерты"| SIEM
  FDC2 --> SIEM
  FDC3 --> SIEM
```

Ставить **в каждый** Kubernetes. Нет глобального Falco-кластера и нет кворума. Правила сводить GitOps'ом, иначе три «правды» SOC.

**Сильное:** RTT между ЦОДами на детект не влияет; blast radius = нода/площадка. **Слабое:** легко забыть включить агент в новом кластере; ошибка ruleset в общем Git — сразу везде, если один выкат на все.

---

## 8. Безопасность (слои)

1. Least privilege (`CAP_BPF` / `CAP_PERFMON` / …), не `privileged: true`; NetworkPolicy Falco → только Sidekick, UI не в интернет.
2. Pin **0.44.1** (драйвер 0.44 ≠ userspace 0.43). Cmdline в алертах может нести ПДн. Falco не *останавливает* атаку — monitoring and detection agent.

Источники: `Falco.md` (DaemonSet, modern eBPF, Sidekick, нет единого кластера). Stretch на схемах не рисуем: модели нет.
