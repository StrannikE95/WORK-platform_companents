# Grafana 13.2.0 — схемы устройства

Связанные документы: правила — `Grafana.md`; установка — `Grafana.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код UI не рисуем.

Допущения: stretch одной Postgres + `ha_peers` на 2–3 ЦОДа **нет**; Grafana **не** TSDB; Helm `grafana-community/grafana` 12.11.2, образ `grafana/grafana:13.2.0`; нагрузка не замерена.

---

## 1. Контекст

Grafana — **визуализация и unified alerting**. Ряды живут в Prometheus/Mimir/Tempo/Loki, не в ней.

```mermaid
flowchart LR
  PEOPLE["Дежурные / SSO"]
  GF["Grafana 13.2.0\nдашборды, не склад метрик"]
  PR["Prometheus / Mimir"]
  TP["Tempo / Loki"]
  WZ["Wazuh / Falco"]

  PEOPLE -->|"HTTPS :443"| GF
  GF -->|"PromQL / TraceQL"| PR
  GF --> TP
  GF -.->|"не SIEM"| WZ
```

| Стрелка | Зачем помнить |
|---|---|
| Люди → Grafana | Картинка и встроенный алерт. Падение Grafana ≠ пропажа рядов |
| Grafana → источник | Без живого scrape панель пустая. Ёмкость — у TSDB |
| Grafana ↛ Wazuh | Красный график не заменяет разбор security-событий |

**Слабое место контекста:** «поставили Grafana» ≠ «мониторинг выдержал нагрузку».

---

## 2. Контейнеры (из чего состоит решение)

Несколько **одинаковых** процессов + **одна** Postgres + gossip **внутри ЦОД-1**. Это не Raft.

```mermaid
flowchart TB
  subgraph dc["Один Kubernetes = один ЦОД"]
    LB["HAProxy / Ingress\n:443 → :3000"]
    subgraph gset["Deployment Grafana"]
      G1["Pod grafana"]
      G2["Pod grafana"]
    end
    HS["Headless :9094\nha_peers"]
    PG["PostgreSQL\nбаза grafana, не SoT"]
  end

  GIT["Git: provisioning\nдашборды / datasources / alerts"]
  SRC["Prometheus / Tempo"]

  LB --> G1
  LB --> G2
  G1 <-->|"gossip 9094 TCP+UDP"| HS
  G2 <--> HS
  G1 --> PG
  G2 --> PG
  GIT --> gset
  G1 -->|"запрос к источнику"| SRC
  G2 --> SRC
```

Порт **3000/TCP** — UI, API, `/api/health`, Live. **9094 TCP и UDP** — Memberlist встроенных Alertmanager. Sticky на LB **не** обязателен: сессии в общей БД.

**Сильное:** любой живой под обслуживает UI. **Слабое:** SQLite + `replicas > 1` — три расходящихся файла; без `ha_peers` UI «HA», письма **дублируются**.

---

## 3. Компоненты внутри процесса

```mermaid
flowchart TB
  subgraph proc["Один процесс grafana"]
    HTTP["HTTP :3000"]
    SCH["Scheduler\nоценивает ВСЕ правила"]
    IAM["Встроенный Alertmanager"]
    INI["grafana.ini / GF_*"]
    PROV["Provisioning YAML/JSON"]
  end

  DB["Postgres: юзеры, дашборды,\nзашифрованные пароли источников"]
  DS["Data source"]

  HTTP --> DB
  SCH --> DS
  SCH --> IAM
  INI --> SCH
  PROV --> HTTP
```

| Компонент | Для чего настраивать |
|---|---|
| Unified alerting | Старый alert-на-панели выключен. Дефолт: **каждый** под считает **все** правила → нагрузка на Prometheus × N |
| `secret_key` | AES паролей источников. Дефолт `SW2YcwTIb9zpOOhoPsMm` известен всем |
| Provisioning из git | Иначе два ЦОДа = две правды дашбордов |
| Live | In-memory; HA стримов — отдельный Redis, не цель первого прода |

---

## 4. Поток: алерт (как взаимодействуют)

```mermaid
sequenceDiagram
  participant Sch as Scheduler пода A
  participant Src as Prometheus
  participant Peer as Alertmanager пода B
  participant CP as Contact point

  Sch->>Src: запрос правила
  Src-->>Sch: ряды
  Note over Sch: без ha_peers каждый под шлёт сам
  Sch->>Peer: gossip 9094 TCP+UDP
  Peer-->>Sch: дедуп уведомления
  Sch->>CP: одно письмо / webhook
```

Встроенный Alertmanager: *availability over consistency* — редкий дубль лучше тишины. Критичные сигналы без Grafana — дублировать во **внешнем** Prometheus Alertmanager.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    R2["replicas ≥ 2\nanti-affinity"]
    HP["ha_peers + 9094"]
    PGHA["Postgres HA\nне SQLite"]
    PDB["PDB / не drain всех"]
  end

  subgraph dr["Между ЦОДами — не stretch"]
    GIT2["Тот же git"]
    IND["Независимый Grafana\nсвоя БД, без ha_peers через город"]
  end

  inside -->|"падение пода"| OK["LB жив, UI жив"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если не настроить |
|---|---|
| Общая Postgres | Зелёные поды, логина нет |
| `ha_peers` в день `replicas > 1` | Дубли писем |
| Свой `secret_key` до источников | Дамп БД расшифровывается дефолтом |
| git provisioning | Клики в UI — единственная копия |

Падение **ЦОД-1** = нет этого Grafana, пока DR/второй экземпляр. Gossip 9094 между ЦОДами не открывать.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> U["Зрители / refresh"]
  Q --> A["Число alert rules"]
  Q --> T["Терабайты рядов"]

  U --> U1["Реплики + CPU/RAM\nвнутри ЦОДа"]
  A --> A1["Нагрузка на источник × N\nне третья реплика «для красоты»"]
  T --> T1["Масштаб Prometheus/Mimir\nне replicas Grafana"]
```

Тиры вендора (Small/Medium/Large) — стартовые точки, не смета. Preview `ha_single_node_evaluation` — не GA.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    HA["Grafana ≥2 + Postgres\ngossip только здесь"]
  end
  subgraph dc2["ЦОД-2"]
    UI2["Люди → UI ЦОД-1\nили свой Grafana + git"]
  end
  subgraph dc3["ЦОД-3 если есть"]
    UI3["Ещё независимый UI\nили только доступ / бэкап БД"]
  end

  HA -.->|"не 9094 / не общая БД"| UI2
  HA -.-> UI3
```

Клиенты в штате ходят на HTTPS ЦОД-1. Третий ЦОД **не** второй writer в ту же БД Grafana.

**Сильное:** латентность логина и gossip не прибита межЦОДовым RTT. **Слабое:** смерть ЦОД-1 = простой UI/встроенных алертов; два Grafana без git = две правды.

---

## 8. Безопасность (ручки на той же схеме)

Не отдельный «кластер ИБ»:

1. NetworkPolicy — 3000 только от LB; 9094 только между подами **этого** ЦОДа.
2. Нет `admin`/`admin`; SSO; `/metrics` с auth.
3. `reporting_enabled` / gravatar / external snapshots **выкл** в закрытом контуре.

Admin Grafana = ключ ко всем паролям data source. Image Renderer — отдельный Chromium, дефолтный токен `-` не оставлять.

Источники фактов: `Grafana.md`. Порога RTT у Grafana **нет** — stretch на схемах не рисуем как целевой.
