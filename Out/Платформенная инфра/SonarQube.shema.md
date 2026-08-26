# SonarQube Server 2026.1.5 LTA — схемы устройства

Связанные документы: правила — `SonarQube.md`; установка — `SonarQube.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код сканера в CI не рисуем.

Допущения: stretch app/search на 2–3 ЦОДа **нет**; прод — **2026.1.5 LTA**, не Helm latest **2026.4.1**; HA приложения — только **DCE**; Community Build — **другая линейка**; нагрузка и LOC не замерены.

---

## 1. Контекст

SonarQube — **сервер отчётов SAST**, не сканер на ноде Kafka и не Falco.

```mermaid
flowchart LR
  CI["CI + SonarScanner"]
  SQ["SonarQube Server 2026.1.5\nQuality Gate"]
  PG["PostgreSQL\nистина настроек и истории"]
  RT["Kafka / Camunda / runtime"]

  CI -->|"отчёт + sonar.token"| SQ
  SQ --> PG
  SQ -.->|"не рантайм"| RT
```

Падение SQ **не должно** ронять шину. Политика CI (ждать / падать / обход gate) — ваша, её нет в продукте. Лицензия Server — **на инстанс / LOC**, не «терабайты озера».

---

## 2. Контейнеры (DCE в одном ЦОДе)

Community/Developer/Enterprise = **один** процесс. Второй под без DCE — второй **инстанс**, не кластер.

```mermaid
flowchart TB
  LB["Ingress / Gateway\nHTTPS, без sticky, health"]
  subgraph dc["ЦОД-1 = один DCE"]
    subgraph app["Application nodes ≥2"]
      A1["Web + CE"]
      A2["Web + CE"]
    end
    subgraph es["Search nodes = 3"]
      S1["ES"]
      S2["ES"]
      S3["ES"]
    end
    PGW["PostgreSQL writer HA\nне read-only replica"]
  end

  CI2["Сканеры CI"] --> LB
  LB --> A1
  LB --> A2
  A1 <-->|"Hazelcast 9003"| A2
  A1 -->|"9001"| es
  A2 --> es
  S1 <-->|"9002"| S2
  S2 <--> S3
  A1 -->|"JDBC 5432"| PGW
```

Порты DCE: **9000** UI/API, **9001** app→search, **9002** ES↔ES, **9003** Hazelcast. С 2026.1 Helm **не** кладёт Postgres в чарт. H2 в проде запрещена вендором.

Вендор: *one application node and one search node can be lost*. Два app в одной зоне = один ЦОД убивает оба.

---

## 3. Компоненты внутри инстанса

```mermaid
flowchart TB
  subgraph sq["Один application node"]
    WEB["Web :9000"]
    CE["Compute Engine\nочередь отчётов"]
    HZ["Hazelcast"]
  end

  SCAN["Scanner в CI"]
  ES["Search / ES"]
  DB["Postgres"]

  SCAN --> WEB
  WEB --> CE
  WEB --> DB
  CE --> DB
  CE --> ES
  WEB --> HZ
```

| Компонент | Для чего настраивать |
|---|---|
| CE workers | Enterprise+: параллель отчётов. На DCE число **глобальное**, на каждом app **повторяется** |
| ES диск | SSD, не NFS; watermark ~90–95% → индекс read-only. Дефолт PVC чарта **5G** — не план |
| `vm.max_map_count` | ≥ **524288** на нодах search (дока 2026.1) |
| JWT | Одинаковый `jwtSecret` на всех app; sticky не нужен |

Плагины **не шарятся**: ставить на все app; выкат = простой **всего** кластера.

---

## 4. Поток: скан → Quality Gate

```mermaid
sequenceDiagram
  participant Sc as Scanner CI
  participant W as Application :9000
  participant Q as Очередь CE
  participant E as Search
  participant D as Postgres

  Sc->>W: отчёт + token
  W->>D: метаданные
  W->>Q: background task
  Q->>E: индекс issues
  Q->>D: метрики / gate
  W-->>Sc: Quality Gate
```

После failover K8s вендор требует **forced ES reindex**. UI может встать, Postgres жив — индекс отстраивают заново.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    DCE["DCE 2+ app / 3 search\nanti-affinity нод"]
    EST["ES auth + TLS"]
    PG["Writer HA Postgres"]
    SSD["SSD PVC search"]
  end

  subgraph dr["Между ЦОДами"]
    COLD["Active-Cold: второй кластер\nвыкл в LB"]
    RIDX["Promote + forced reindex"]
  end

  inside -->|"1 app или 1 search"| OK["пользователи живы"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если забыть |
|---|---|
| Ключ DCE | Community/EE «в трёх ЦОДах» ≠ HA приложения |
| Pin **2026.1.5** | Чарт 2026.4.1 поставит Latest |
| Не ходить в read-only replica | Вендор: *does not support* Active/Active на RO БД |
| Свои пароли | `admin`/`admin` и Helm `AdminAdmin_12$` публичны |

Без DCE честный прод приложения — один узел + HA Postgres + холодный DR.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где очередь?"]
  Q --> CE["pending_count CE"]
  Q --> UI["UI / приём отчётов"]
  Q --> IX["Индекс issues"]
  Q --> SC["Тяжёлый сам скан"]

  CE --> CE1["Workers + heap CE\nпотом ещё app node"]
  UI --> UI1["Ещё application nodes\nHPA min 2, на апгрейд выкл"]
  IX --> IX1["RAM/диск search\nHPA search не масштабирует"]
  SC --> SC1["Агент CI, не StatefulSet SQ"]
```

LOC и таблицы 10M/50M вендора — единица пересчёта, не смета. Search HPA **нет**.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    LIVE["DCE + writer Postgres"]
  end
  subgraph dc2["ЦОД-2"]
    CLD["Cold DCE + replica Postgres"]
  end
  subgraph dc3["ЦОД-3"]
    BAK["Второй cold или только бэкап"]
  end
  LIVE -->|"streaming / PITR"| CLD
  LIVE --> BAK
```

Не класть search «по одному на ЦОД» без замера 9001–9003. Три полноценных инстанса = тройной LOC и три Quality Gate.

**Сильное:** ES и JDBC локальные. **Слабое:** падение ЦОД-1 = нет SQ, пока DR; RTO включает reindex (минут в цитате вендора нет).

---

## 8. Безопасность (ручки на той же схеме)

Force authentication оставить. Переименовать `sonar-administrators` **до** синхронизации групп с IdP. 9001–9003 не с мира. Сканер — `sonar.token`, не пароль в Git. ArgoCD вендор помечает как *not currently fully supported*.

Источники: `SonarQube.md`. Порога RTT у вендора **нет**.
