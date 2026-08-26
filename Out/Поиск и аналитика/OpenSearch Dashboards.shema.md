# OpenSearch Dashboards 3.8.0 — схемы устройства

Связанные документы: правила — `OpenSearch Dashboards.md`; установка — `OpenSearch Dashboards.install.md`. Кластер — `OpenSearch.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код плагинов UI не рисуем.

Допущения: OSD **рядом** с кластером своего ЦОДа; `opensearch.hosts` — ноды **одного** кластера, не leader+follower как fallback; версия = OpenSearch **3.8.0**; нагрузка (число аналитиков) не замерена.

---

## 1. Контекст

Dashboards — **Node.js UI и прокси к 9200**. Кластер живёт без UI; UI без кластера — красный экран.

```mermaid
flowchart LR
  PPL["Аналитики / SSO"]
  OSD["OSD 3.8.0 :5601"]
  OS["OpenSearch 3.8.0"]
  GF["Grafana"]
  WZ["Wazuh UI"]

  PPL -->|"HTTPS"| OSD
  OSD -->|"9200 от имени пользователя"| OS
  OSD -.->|"не PromQL"| GF
  OSD -.->|"не этот UI"| WZ
```

Падение OSD = «не видим картинки». Запись ingest и API 9200 могут быть живы. SLO поиска ≠ SLO браузера.

---

## 2. Контейнеры (stateless рядом с кластером)

PVC поиска **не нужен**. Saved objects живут **в индексах** кластера.

```mermaid
flowchart TB
  ING["Ingress TLS :443"]
  subgraph dc["ЦОД кластера"]
    O1["OSD pod"]
    O2["OSD pod"]
    SVC["OpenSearch coordinating :9200"]
  end

  ING --> O1
  ING --> O2
  O1 -->|"kibanaserver + user"| SVC
  O2 --> SVC
```

Порт **5601** — UI. Исходящий **9200** — не слушает OSD. Deployment, не StatefulSet. Helm дефолт `updateStrategy: Recreate` — на выкате **ноль** подов; в проде **RollingUpdate**.

**Сильное:** убить один под — LB уводит трафик. **Слабое:** реплики OSD в «пустом» ЦОДе без 9200 дают ложное HA.

---

## 3. Компоненты состояния

```mermaid
flowchart TB
  subgraph osd["Процесс OSD"]
    HTTP["HTTP :5601"]
    SEC["Security Dashboards plugin\ncookie-сессия"]
    YML["opensearch_dashboards.yml"]
  end

  KB["Индексы .kibana*\n/ .opensearch_dashboards*"]
  KS["Пользователь kibanaserver\nроль kibana_server"]

  HTTP --> SEC
  SEC --> KB
  KS --> KB
  YML --> HTTP
```

| Компонент | Для чего настраивать |
|---|---|
| Saved objects | Не на диске пода. Удалили volume OSD — дашборды на месте; удалили `.kibana*` — нет |
| `cookie.password` | ≥ 32 символа, **один** Secret на Deployment. Дефолт `security_cookie_default_password` общеизвестен |
| `kibanaserver` | Служебный процесс, не учётка аналитика. Demo-пароль только в изоляции |
| Тенанты | Каждый private tenant = ещё индекс `.kibana_*` в cluster state |

Сессия **не** в Redis. Любая реплика примет cookie, если секрет одинаковый.

---

## 4. Поток: логин и Discover

```mermaid
sequenceDiagram
  participant B as Браузер
  participant O as OSD
  participant S as OpenSearch 9200

  B->>O: HTTPS 5601
  O->>S: служебный .kibana* как kibanaserver
  B->>O: логин SSO / basic
  O-->>B: cookie security_authentication
  B->>O: Discover
  O->>S: запрос от имени пользователя
  S-->>O: hits
  O-->>B: таблица
```

Заголовок `securitytenant` должен быть в `requestHeadersAllowlist` вместе с `Authorization` — иначе OSD стартует **red** (дока).

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["В том ЦОДе, где кластер"]
    R2["replicas ≥ 2"]
    CK["общий cookie.password"]
    RU["RollingUpdate + PDB"]
    SN["snapshot .kibana*"]
  end

  subgraph other["Чужой ЦОД"]
    NO["Без 9200 — не ставить OSD «для HA»"]
    FOLL["Отдельный OSD только на CCR follower"]
  end

  inside -->|"падение пода"| OK["UI жив, cookie принимается"]
  inside -->|"падение OpenSearch"| RED["честно красный UI"]
```

| Ручка | Если забыть |
|---|---|
| Версия = 3.8.0 = кластеру | Плагины major.minor.patch; смесь 3.8 UI с 2.x — лотерея |
| `verificationMode: full` | Demo `none` в проде |
| `cookie.secure` согласовать с Ingress | HTTPS снаружи + HTTP до пода + secure cookie = логин «не работает» |
| NDJSON / snapshot | Единственная копия дашбордов — в кластере |

Пробы TCP 5601 = «порт открыт», не «`.kibana` зелёный».

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Ось"]
  Q --> P["Люди / вкладки"]
  Q --> D["Тяжёлый Discover"]

  P --> P1["Реплики OSD в том же ЦОДе"]
  D --> D1["Кластер OpenSearch\nне пятый Dashboards"]
```

Дефолт Helm **512M** не копировать. `NODE_OPTIONS=--max-old-space-size` **ниже** memory limit. HPA только после профиля: иначе реплики вырастут из-за aggregation, который лечат шардами.

Нет формулы «N аналитиков = M реплик».

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1 + leader"]
    UI1["OSD ≥2 → 9200 этого кластера"]
  end
  subgraph b["ЦОД-2"]
    UI2["Нет OSD или свой OSD на follower"]
  end
  subgraph c["ЦОД-3"]
    UI3["Не размазывать OSD без 9200"]
  end
  UI1 -.->|"не смешивать hosts"| UI2
```

`data_source.enabled` — второй Security realm в одной сессии; Timeline с этим режимом **не** поддерживается. Wazuh indexer в этот UI не подключать.

**Сильное:** короткий путь до 9200, горизонталь без Redis. **Слабое:** падение ЦОДа кластера = нет UI; утечка cookie.password = подделка любой сессии.

---

## 8. Безопасность (ручки на той же схеме)

5601 не в интернет без SSO. Людей `kibanaserver` не пускать. `admin` ежедневно запрещён. SAML ACS — стабильный URL LB, не Pod IP. Аноним на ПДн не включать. Proxy auth только если OSD недоступен в обход proxy.

Источники: `OpenSearch Dashboards.md`.
