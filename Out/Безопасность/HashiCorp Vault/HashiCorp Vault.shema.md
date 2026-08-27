# HashiCorp Vault 2.0.4 — схемы устройства

Связанные документы: правила — `HashiCorp Vault.md`; установка — `HashiCorp Vault.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код клиентов не рисуем.

Допущения (без них схема врёт):

1. Stretch одного Raft на 2–3 ЦОДа **нет** (запрет платформы). Порог HashiCorp RTT &lt; 8 мс для AZ **не** делает stretch целевым.
2. Community 2.0.4 + Helm `hashicorp/vault` **0.34.1**. Performance Replication и DR replication **нет**.
3. Нагрузки нет — на схемах нет «N RPS». Есть *что крутить*, когда цифры появятся.

---

## 1. Контекст (C4 system context)

Vault — **источник истины секретов и ключей**, не SoT карточек и не WAF.

```mermaid
flowchart LR
  subgraph people["Люди и сервисы"]
    MS["Микросервисы / интеграционное API"]
    CW["Camunda job workers"]
    ADM["Админы / OIDC"]
  end

  VT["Vault 2.0.4\nсекреты и ключи"]
  PG["PostgreSQL\nдинамический пароль"]
  KF["Kafka SASL / TLS"]
  LK["Озеро: ciphertext\nключ в transit"]

  MS -->|"login JWT / AppRole"| VT
  CW --> VT
  ADM --> VT
  VT -->|"database engine"| PG
  VT -->|"cert / creds"| KF
  VT -.->|"не карточка клиента"| LK
```

Сервис ходит за коротким токеном (KV / database / pki / transit), не в ConfigMap. Озеро остаётся SoT данных: ciphertext там, ключ в transit; терабайты в KV — антипаттерн. Компрометация Vault хуже падения одного микросервиса.

---

## 2. Контейнеры (из чего состоит решение)

Дефолт Helm — **standalone + file**. Это не прод (Security Warning HashiCorp). Прод = HA Raft, нечётные voter.

```mermaid
flowchart TB
  subgraph k8s["Один Kubernetes = один ЦОД"]
    LB["LB / Ingress passthrough\nhealth /v1/sys/health"]
    INJ["Agent Injector webhook\nsidecar секретов"]
    subgraph raft["Raft Integrated Storage"]
      ACT["Pod active\nлидер, запись"]
      SB1["Pod standby"]
      SB2["Pod standby"]
    end
    PVC["PVC data + audit\nне NFS как raft path"]
  end

  APP["Приложения"]
  SNAP["Snapshot\nв другие ЦОДы"]

  APP -->|"8200 TLS"| LB
  INJ -.-> APP
  LB --> ACT
  LB -->|"forwarding 8201"| SB1
  ACT <-->|"Raft 8201"| SB1
  ACT <--> SB2
  ACT --> PVC
  SB1 --> PVC
  ACT -->|"ручной snapshot Community"| SNAP
```

Порт **8200** — API клиентов. **8201** — Raft, forwarding, mTLS между узлами. Клиент «всё равно, на какой под», standby пересылает лидеру. Community **performance standby нет**: чтения тоже часто идут через active.

**Сильное:** смена лидера внутри ЦОДа. **Слабое:** нет кворума → нет записи (login, KV put, смена политик).

---

## 3. Компоненты внутри инстанса

```mermaid
flowchart TB
  subgraph instance["Один процесс vault"]
    LIS["listener 8200"]
    AUT["Auth: K8s JWT / AppRole / OIDC"]
    POL["ACL policy"]
    ENG["Engines: KV, database, pki, transit"]
    SEAL["Seal / unseal\nShamir или auto-unseal"]
    AUD["Audit device\nесли нельзя писать — API встаёт"]
  end

  CLI["Клиент / Agent"]
  DISK["Raft log на PVC"]

  CLI -->|"TLS"| LIS
  LIS --> AUT
  AUT --> POL
  POL --> ENG
  SEAL --> DISK
  ENG --> DISK
  ENG --> AUD
```

| Компонент | Для чего настраивать |
|---|---|
| Shamir / auto-unseal | `init` один раз. На K8s без auto-unseal reschedule = sealed |
| KV / database / pki / transit | Четыре движка платформы; не класть озеро в KV |
| Audit ≥ 2 | Один file на полном диске = Vault отказывается обслуживать |
| Root token | После auth **отозвать**. Держать root «на всякий» — дыра |

---

## 4. Поток: приложение получает секрет

```mermaid
sequenceDiagram
  participant App as Приложение
  participant V as Vault active
  participant St as Standby Raft
  participant Eng as Engine KV / database / pki

  App->>V: login Kubernetes JWT
  V->>V: policy → короткий токен
  App->>V: read secret / lease
  Note over V,St: запись в storage ждёт кворум 2 из 3 в ЭТОМ ЦОДе
  V->>St: Raft replicate
  St-->>V: commit
  V->>Eng: KV get / dynamic creds / cert / transit
  V-->>App: секрет + lease
```

Unseal: после старта мастер-ключ в памяти нет. Shamir — люди вводят K из N долей **на каждый** под. Auto-unseal (KMS / Transit-Vault) — иначе Kubernetes не прод.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    N35["3 или 5 voter Raft\nanti-affinity"]
    AU["auto-unseal\nне Shamir без runbook"]
    AUD2["два audit device"]
  end

  subgraph dr["Между ЦОДами — не stretch"]
    SN["snapshot save/restore\nшифрованный object store"]
    MAN["Restore вручную / GitOps"]
  end

  inside -->|"падение пода"| FA["выборы лидера\nкворум жив"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если не настроить |
|---|---|
| HA Raft, не standalone+file | Нет выборов, один диск = весь контур |
| 3–5 voter + anti-affinity | Два пода на одной VM = «HA на бумаге» |
| Auto-unseal | Drain ноды = ночной звонок держателям Shamir |
| Snapshot вне кластера | Community **не** даёт DR-реплику |
| `/v1/sys/health` на LB | TCP 8200 «открыт» на sealed — ложь |

Падение **домашнего** ЦОДа — нет секретов, пока restore. Это цена запрета stretch.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> W["Запись / диск лидера"]
  Q --> R["Чтение / forwarding"]
  Q --> P["Пик Injector login"]

  W --> W1["Вертикаль active\nIOPS SSD, не +10 подов"]
  W --> W2["Лишние voter не ускоряют запись"]
  R --> R1["Community: всё через active"]
  P --> P1["Квоты lease; свой нагрузочный профиль"]
```

Горизонтали **записи** у Community нет. Локальные чтения в другом ЦОДе — **Performance Replication (Enterprise)**. Без лицензии этого нет.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    CL["Raft 3-5 voter\nодин SoT секретов"]
  end
  subgraph dc2["ЦОД-2"]
    DR1["Копия snapshot\nне второй кластер-истина"]
  end
  subgraph dc3["ЦОД-3 если есть"]
    DR2["Вторая копия снимка"]
  end

  CL -->|"raft snapshot"| DR1
  CL -->|"raft snapshot"| DR2
```

Клиенты в штате пишут в active ЦОД-1. Три независимых Community-кластера **без** PR = **три SoT**: политики и ключи разъедутся. HashiCorp для «единых ключей на 30 интеграций» сделал PR — в Community его нет.

**Сильное:** Raft не ждёт межЦОдовый RTT; порог 8 мс не ваш путь. **Слабое:** смерть ЦОД-1 = простой секретов до restore; RPO = интервал снимка.

---

## 8. Безопасность (ручки на той же схеме)

1. NetworkPolicy: клиенты → 8200; узлы ↔ 8200/8201.
2. TLS e2e; HashiCorp не рекомендует терминировать TLS на Ingress и дальше plaintext.
3. Least privilege ACL, короткие TTL, UI не в интернет.
4. VSO кладёт секрет в etcd — только с encryption-at-rest.

Источники фактов: `HashiCorp Vault.md` (Helm 0.34.1, Raft, Shamir, редакции). Stretch на схемах не рисуем как целевой.
