# Kubernetes 1.36.4 — схемы устройства

Связанные документы: `Kubernetes.md`, `Kubernetes.install.md`.

C4 / «D4»: платформа оркестрации, не озеро данных. Stretch etcd на 2–3 ЦОДа **нет**.

Допущения: kubeadm, containerd, Linux; CNI обязан уметь NetworkPolicy (конкретный продукт не зафиксирован); нагрузки нет.

---

## 1. Контекст

```mermaid
flowchart TB
  ADM["Админы / GitOps / CI"]
  K8S["Kubernetes 1.36.4\nодин кластер = один ЦОД"]
  WL["Поды: Kafka, Camunda, API, PG…"]
  ETCD["etcd\nтолько метаданные кластера\nне терабайты озера"]

  ADM -->|"6443 TLS API"| K8S
  K8S --> WL
  K8S --> ETCD
```

Уже запущенные поды переживают короткое падение API. Новые деплои, failover подов, смена Service — **нет**, пока control plane глухой. Терабайты живут в PVC/Kafka/БД, не в etcd (квота порядка 2–8 ГиБ метаданных).

---

## 2. Контейнеры control plane и ноды

```mermaid
flowchart TB
  LB["HAProxy / Keepalived\nTCP 6443 этого ЦОДа"]

  subgraph cp["Control plane stacked x3"]
    API1["kube-apiserver"]
    SCH["kube-scheduler"]
    CM["controller-manager"]
    E1["etcd member"]
  end

  subgraph workers["Workers"]
    KL["kubelet"]
    PX["kube-proxy или CNI"]
    POD["Pods"]
  end

  CNI["CNI сеть подов"]
  CSI["CSI диски"]

  ADM2["kubectl"] --> LB
  KL --> LB
  LB --> API1
  API1 --> E1
  SCH --> API1
  CM --> API1
  KL --> POD
  POD --> CNI
  POD --> CSI
```

Без CNI после `kubeadm init` поды не станут Running. Без CSI нет нормальных дисков для Kafka/Postgres. LoadBalancer «из облачного гайда» на железе сам не появится.

---

## 3. Компоненты, которые путают с «кластером приложений»

```mermaid
flowchart LR
  subgraph k8s["Это Kubernetes"]
    OBJ["Deployment / STS / Service / Secret"]
    RBAC["RBAC / SA"]
    PSA["Pod Security Admission"]
    NP["NetworkPolicy"]
    HPA["HPA"]
  end

  subgraph not["Это не Kubernetes"]
    APP["RF Kafka, реплики PG"]
    REG["Реестр образов"]
    ING["Ingress-контроллер"]
  end
```

HPA масштабирует **поды**. Для Kafka «ещё под» не создаёт партиции. PDB защищает от **добровольного** drain, не от падения ЦОДа.

---

## 4. Поток: создать под

```mermaid
sequenceDiagram
  participant U as kubectl / GitOps
  participant API as kube-apiserver
  participant ET as etcd
  participant S as scheduler
  participant K as kubelet

  U->>API: apply Deployment
  API->>ET: запись состояния
  Note over ET: кворум 2 из 3 в ЭТОМ ЦОДе
  S->>API: bind на ноду
  K->>API: видит назначение
  K->>K: CRI запускает контейнер
```

Каждая запись в API платит за Raft etcd **локальной** площадки.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph local["Внутри ЦОДа"]
    CP3["3 control-plane\nControlPlaneEndpoint"]
    DS["SSD под etcd не NFS"]
    BK["снимок etcd + проверка restore"]
    Z["topology.kubernetes.io/zone\nхотя бы залы"]
  end

  subgraph islands["2–3 ЦОДа"]
    K1["Кластер ЦОД-1"]
    K2["Кластер ЦОД-2"]
    K3["Кластер ЦОД-3"]
    GO["GitOps overlays\nразные endpoint БД/Kafka"]
  end

  local --> islands
```

| Ручка | Если забыть |
|---|---|
| LB :6443 переживает 1 машину | HA control plane на бумаге |
| 3 etcd | Один член = нет переживания отказа |
| zone labels + topologySpread | Все реплики приложения в одном зале |
| encryption at rest Secrets | plaintext в etcd |
| сертификаты kubeadm ~1 год | Внезапный простой API |

Антипаттерн: etcd только в ЦОД-1, воркеры везде. Смерть ЦОД-1 = нельзя поднять поды на живых площадках.

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart LR
  P["Ось подов HPA"] --> CPU["CPU/RAM сервиса\nпока влезает на ноды"]
  N["Ось нод"] --> CAP["Суммарные ресурсы кластера"]
  N --> ET["Давление на etcd/API"]
```

Requests обязательны, иначе BestEffort выкинут первыми. Числа «50/250/1000 нод» в доке etcd — примеры нагрузки **на etcd**, не смета вашей системы.

---

## 7. Безопасность (слои)

```mermaid
flowchart TB
  A["Сеть: 6443 не в интернет"] --> B["TLS kubeadm"]
  B --> C["Люди: OIDC не admin.conf на 50 человек"]
  C --> D["RBAC least privilege"]
  D --> E["PSA restricted / baseline"]
  E --> F["NetworkPolicy default-deny"]
  F --> G["EncryptionConfiguration до боевых Secret"]
  G --> H["Audit"]
```

NetworkPolicy без CNI с поддержкой политик — декорация. KMSv2 если есть KMS; иначе aesgcm с ключом не в Git. KMSv1 не использовать.

---

## 8. Сильное / слабое острова на ЦОД

**Сильное:** etcd локальный, blast radius = площадка, согласовано с документацией etcd.  
**Слабое:** 2–3 PKI, 2–3 upgrade; приложение само активно на нескольких кластерах; без GitOps конфиги разъедутся.

Источники: `Kubernetes.md`. Порога RTT «для etcd» у проекта Kubernetes **нет**.
