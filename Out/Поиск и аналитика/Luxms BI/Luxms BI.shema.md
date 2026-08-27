# Luxms BI 12.x — схемы устройства

Связанные документы: правила — `Luxms BI.md`; установка — `Luxms BI.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код PL/pgSQL атласов не рисуем.

Допущения: stretch Consul/Patroni/NATS на 2–3 ЦОДа **нет**; коммерческие **пакеты** вендора (реестр 3366), не выдуманный docker-compose прод; публичного Helm 12.x **нет**; не замена Kafka/Camunda/Grafana; нагрузка не замерена.

---

## 1. Контекст

Luxms — **бизнес-дэшборды** (ВУК). Двухзвенность: логика в PostgreSQL, короткий путь через Nginx. Не озеро-эталон и не шина микросервисов.

```mermaid
flowchart LR
  PPL["Руководители / аналитики"]
  LX["Luxms BI 12.x"]
  LAKE["Озеро / ClickHouse витрины"]
  KF["Kafka вашей платформы"]
  CAM["Camunda"]
  GF["Grafana ops"]

  PPL -->|"HTTPS :443"| LX
  LX -->|"JDBC / FDW"| LAKE
  LX -.->|"источник витрин, не ядро"| KF
  LX -.->|"не BPMN"| CAM
  LX -.->|"не этот UI"| GF
```

Data Boring/JDBC в СМЭВ — обход интеграционного API. NATS внутри Luxms **не** замена Apache Kafka.

ФСТЭК № 5055 — сборка **11.0.x**, не «12.x с тем же сертификатом».

---

## 2. Контейнеры (мозг в одном ЦОДе)

Штатный путь — пакеты Linux на VM (матрица ОС вендора) + Ansible после согласования схемы. Самодельный Compose = вы сами support.

```mermaid
flowchart TB
  HAP["HAProxy :443 TLS"]
  subgraph dc["ЦОД-1 мозг"]
    WEB["luxmsbi-web ×N\nNginx + Lua"]
    APP["appserver / datagate"]
    PG["PostgreSQL 15/17 + Patroni"]
    CS["Consul 1.16.1 ×3"]
    KD["KeyDB :6379"]
    NT["NATS cluster\n4222 / 6222"]
  end

  HAP --> WEB
  WEB --> APP
  WEB --> PG
  APP --> PG
  PG --> CS
  WEB --> KD
  APP --> NT
```

Порты NATS **12.x**: клиенты 4222, cluster 6222, monitor 8222. Java/web (8080, 8200, 8888, …) — сверять с **sysadm-guide вашей сборки**; гайд 9.2 не замена. Пример NATS `no_tls` / `no_auth_user` / пароли `"x"`/`"y"` — демо, не прод.

Клиенты Postgres **только** через VIP/HAProxy, не hostname лидера. В гайде 9.2 липкость к appserver **желательна** (`hash $upstream_cookie`).

**Сильное:** падение одного web — LB жив. **Слабое:** упал primary без кворума Consul — nginx зелёный, записи нет.

---

## 3. Компоненты платформы

```mermaid
flowchart LR
  subgraph core["Ядро"]
    PLS["PL/pgSQL / атласы"]
    EXT["plv8, pgsql-http,\nredis_fdw, pubsub"]
  end
  DG["datagate JDBC"]
  DB["Data Boring ETL"]
  BINS["BINS WebSocket"]

  PLS --> EXT
  DG --> PLS
  DB --> PLS
  BINS --> KD2["KeyDB"]
```

Importer линейки 9.2 для **новых** внедрений не берём. Terabytes фактов — MPP/озеро, **не** единственный Postgres метаданных. Расширения — **из пакетов вендора**, не «с GitHub на глаз». Пакеты PG **13** вендор прекратил с 01.12.2025.

Лицензия UL: в описании потолок **1000** одновременных сессий. Enterprise Cluster в tech-info — формулировка **2** промышленных сервера; три площадки согласовать отдельно.

---

## 4. Поток: открыть дэш

```mermaid
sequenceDiagram
  participant B as Браузер
  participant W as luxmsbi-web
  participant A as Appserver
  participant P as Postgres leader
  participant V as Витрина / FDW

  B->>W: HTTPS
  W->>A: API
  A->>P: логика PL/pgSQL
  P->>V: факты
  V-->>P: строки
  P-->>W: ответ
  W-->>B: дэш
```

Падение озера: UI жив, дэши пустые — **ожидаемо**, это не «упал Luxms».

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    C3["Consul 3"]
    PAT["Patroni 3 + VIP"]
    N3["NATS cluster + диск JetStream"]
    W2["≥2 web, ≥2 appserver"]
  end

  subgraph dr["Между ЦОДами"]
    ASYNC["Async реплика / бэкап ядра"]
    RUN["Promote + VIP вручную"]
  end

  inside -->|"1 VM"| OK["switchover Patroni"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если забыть |
|---|---|
| Кворум Consul в **этом** ЦОДе | Stretch gossip = запись встаёт при split |
| Бэкап **ядра** (атласы, роли) | Реплика повторит `DROP`; витрины пересчитаешь, метаданные — нет |
| Пароли не `bi`/`bi` | Пример Lua гайда 9.2 |
| Антибрут | `max_login_attempts = 0` в 9.2 — **выкл** |

Реплика Patroni не заменяет backup ядра.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Ось"]
  Q --> U["Зрители"]
  Q --> J["Живой JDBC / self-service"]
  Q --> T["Терабайты"]

  U --> U1["CPU web+БД, пул HAProxy\nпотолок лицензии сессий"]
  J --> J1["datagate и источник\nне ещё nginx"]
  T --> T1["ClickHouse / озеро\nядро остаётся Postgres"]
```

Tech-info: минимум **8 ядер / 24 ГиБ / 500 ГиБ** на сервер; ориентир **100** одновременных — **16/32/1 ТиБ**. Это репер вендора, не ваша смета. На одном хосте «обычно до 100 активных».

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1"]
    BR["Consul + Patroni + NATS + KeyDB + web"]
  end
  subgraph dc2["ЦОД-2"]
    DR["Cold пакеты + async PG / бэкап"]
  end
  subgraph dc3["ЦОД-3"]
    B2["Второй DR или только бэкап"]
  end
  BR -->|"не член кворума Consul"| DR
  BR --> B2
```

Web в чужом ЦОДе без согласованного KeyDB — «то залогинен, то нет». Три независимых атласа «для HA» — три правды прав.

**Сильное:** кворум DCS предсказуем. **Слабое:** падение ЦОДа мозга = нет BI для всех; нет публичного оператора K8s.

---

## 8. Безопасность (ручки на той же схеме)

SSO Keycloak/AD; Admin — break-glass. Datagate исходящие **только** на витрины, не в подсеть ведомств. RLS и запрет бесконтрольного Excel на ПДн. ИИ-модуль 12.x в первом проде выкл. Журналы в SIEM. TLS на LB; plaintext Postgres «для нагрузки» из гайда — слабая рекомендация для контура с ПДн. `protected-mode no` у KeyDB в прод не тащить.

Источники: `Luxms BI.md`. Порога RTT у Luxms **нет**.
