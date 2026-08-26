# Grafana 13.2.0 — установка и конфигурирование

Связанный документ (глоссарий, HA, безопасность, почему так): `Grafana.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Grafana (одна БД + одни `ha_peers`) на несколько ЦОДов **не делаем**: межЦОДовый RTT для Postgres и gossip 9094 (TCP+UDP) неприемлем.

Версии: **Grafana OSS 13.2.0**, образ `grafana/grafana:13.2.0`. На Kubernetes — Helm **`grafana-community/grafana` 12.11.2** (`appVersion` 13.2.0). Репозиторий: `https://grafana-community.github.io/helm-charts`.  
Документация: https://grafana.com/docs/grafana/latest/  
HA: https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Реплики Grafana с общей БД и `ha_peers` живут **внутри одного ЦОДа** (одного Kubernetes). Между ЦОДами — отдельный экземпляр **или** люди ходят в Grafana ЦОД-1. Автоfailover Grafana между Kubernetes **нет**.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Dev — изолированная сеть; пароль в примере не секрет.
4. Нагрузки нет — поэтому **нет** цифры CPU/RAM «хватит для прода». Есть минимум, чтобы процесс жил, и рычаги роста.
5. Общая БД Grafana в проде — **PostgreSQL 12+** (отдельная база `grafana`, не SoT и не Camunda). HA этой БД — `PostgreSQL.install.md`, не этот файл.
6. Дашборды в проде — **provisioning из git**, не только клики в UI. Иначе два ЦОДа = две правды.
7. Для 2 ЦОДов: активный Grafana в ЦОД-1 **или** независимый экземпляр в каждом ЦОДе с одним git. Для 3 ЦОДов: то же + третья площадка как ещё один независимый UI **или** только доступ к ЦОД-1. Третий ЦОД **не** добавляет второй writer в ту же БД Grafana.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** открыть UI, подключить Prometheus/Tempo, понять Explore и алерт. **Не** цель: отказ площадки и 200 зрителей.

### Предпосылки

- Docker Engine (стенд разработчика) **или** любой однонодовый Kubernetes.
- Порт 3000 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name grafana-dev \
  -p 127.0.0.1:3000:3000 \
  grafana/grafana:13.2.0
```

Привязка к `127.0.0.1` обязательна: `-p 3000:3000` без адреса часто слушает все интерфейсы.

Первый вход: `admin` / `admin`, вендор просит сменить пароль. Данные — SQLite внутри контейнера. Удалили контейнер без volume — удалили дашборды.

Проверка:

```bash
curl -s http://127.0.0.1:3000/api/health
docker exec grafana-dev grafana cli --version
```

В ответе health — JSON со статусом БД. Версия процесса — **13.2.0**, не `latest`.

Сохраните JSON дашборда в git сразу: без volume и без provisioning стенд не переживает пересоздание.

### Установка (Kubernetes Dev, если стенд уже в K8s)

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
helm install grafana-dev grafana-community/grafana --version 12.11.2 \
  --set image.tag=13.2.0 \
  --set replicas=1
```

`replicas: 1`, SQLite. **Не** этот values в прод. Port-forward только на localhost.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Один процесс | да | Некому строить HA |
| БД | SQLite | Нет требования пережить выкат |
| `ha_peers` | не нужен | Один Alertmanager |
| TLS | нет | Иначе PKI раньше PromQL |
| Image Renderer / Live Redis | выкл | Не цель |
| `admin`/`admin` | сменить, сеть закрыта | В прод не копировать |
| reporting / gravatar / snapshots | можно оставить на изолированном стенде | В чек-лист прода не переносить |

Чего **не** упрощать: тег **13.2.0**; хотя бы один источник и панель, которая не пустая; дашборд в git; 3000 не с мира; unsigned plugins нет.

### Проверка Dev

1. `/api/health` отвечает, UI открывается с localhost.
2. Источник (например Prometheus) в Explore живой, панель обновилась.
3. Рестарт контейнера без volume: дашборды пропали — это ожидаемо, не баг стенда.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный образ | Нет failover, нет общей БД, нет gossip |
| Совпадает с quickstart Grafana | Успешный график **не** доказывает прод |
| Дешёво гоняет PromQL | Открытый 3000 на Wi-Fi = дыра; привычка `admin`/`admin` уедет в бой |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой отсутствия UI Grafana (пока не поднимете второй экземпляр / не пойдёте в живой ЦОД) и **ручного** DR. Цифр зрителей нет — ниже минимум HA внутри площадки, не смета железа.

### Почему не stretch

HA Grafana = несколько процессов + **одна** Postgres + gossip **9094 TCP и UDP**. Sync `COMMIT` в Postgres не быстрее flush на standby; Memberlist не рассчитан на WAN. При неприемлемом RTT stretch даёт таймауты логина, дубли алертов и ложный «кластер», а не защиту. Документация Grafana **не задаёт** порог миллисекунд.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один логический Grafana:

- Deployment **≥ 2** реплик (для Medium вендор пишет 2, для Large 3+) на **разных нодах** одного ЦОДа;
- **не** SQLite: внешний PostgreSQL **этого** ЦОДа (CNPG `instances: 3` — см. `PostgreSQL.install.md`);
- `headlessService: true`; в `grafana.ini`: `ha_listen_address` / `ha_advertise_address` = `${POD_IP}:9094`, `ha_peers` = headless:9094;
- дашборды, datasources, alert rules — **provisioning из git** (ConfigMap/sidecar). Клики в UI будут перезатёрты — выбрать один источник правды;
- перед 3000 — HTTPS LB (HAProxy/Ingress). Sticky **не** обязателен: сессии в общей БД;
- образ **13.2.0**, чарт **12.11.2**, не `latest`.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: Grafana HA + Postgres Grafana. ЦОД-2: люди ходят в UI ЦОД-1 **или** независимый Grafana (своя БД, тот же git provisioning), **без** `ha_peers` и без общей БД через город | Нет UI ЦОД-1, пока DR/второй экземпляр. Алерты встроенные молчат вместе с этим Grafana |
| **3 ЦОДа** | То же + ЦОД-3: ещё один независимый UI с git **или** только доступ к ЦОД-1 / бэкап БД | То же; третий ЦОД не даёт второго writer в ту же БД |

Не растягивать: один Postgres Grafana на 2–3 Kubernetes; gossip 9094 между ЦОДами; «три реплики Grafana в трёх ЦОДах, одна SQLite».

Метрики с других площадок — **удалённый Prometheus / federation** (см. `Prometheus.install.md`), не второй Grafana «как правда».

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не один stretch-кластер).
- HA Postgres для базы `grafana` **внутри ЦОД-1**; WAL archive для DR.
- NetworkPolicy: 3000 только от LB; 9094 только между подами Grafana **этого** ЦОДа; 5432 только Grafana → Postgres.
- Секреты (`secret_key`, пароль БД, SSO) — в Secret/Vault, не в Git.
- Свой `secret_key` **до** заведения источников. Дефолт `SW2YcwTIb9zpOOhoPsMm` известен всем.

### Установка (Helm, активный ЦОД)

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm upgrade --install grafana grafana-community/grafana --version 12.11.2 \
  --namespace monitoring --create-namespace \
  -f grafana-prod-values.yaml
```

Смысл values (не полный файл — сверять с README чарта 12.11.2):

```yaml
image:
  repository: grafana/grafana
  tag: "13.2.0"
replicas: 2
headlessService: true
persistence:
  enabled: false   # состояние в Postgres, не PVC SQLite
grafana.ini:
  database:
    type: postgres
    host: pg-grafana-rw:5432
    name: grafana
    ssl_mode: verify-full
  unified_alerting:
    ha_peers: grafana-headless:9094
  security:
    admin_user: break-glass   # пароль — Secret, не admin/admin
    cookie_secure: true
  analytics:
    reporting_enabled: false
    check_for_updates: false
  snapshots:
    external_enabled: false
  server:
    root_url: https://grafana.example.internal/
```

Оператор Grafana 5.25.0 — допустимая альтернатива с теми же правилами (общая БД + `ha_peers` **внутри ЦОДа**), не «тот же чарт».

### Конфигурирование активного экземпляра (ЦОД-1)

Обязательно рядом:

1. **Postgres HA** в том же ЦОДе. Падение БД = падение всех реплик Grafana, даже если поды зелёные.
2. Gossip **в тот же день**, когда `replicas > 1`. Иначе UI «HA», письма дублируются. Метрика `grafana_alertmanager_cluster_members`.
3. Свой `secret_key` (длинный случайный). Нет дефолтного `admin`/`admin`. SSO (OAuth/LDAP); Admin — break-glass в Vault.
4. `reporting_enabled = false`, `check_for_updates = false`, `disable_gravatar = true`, `snapshots.external_enabled = false`, anonymous выкл.
5. `/metrics` с basic auth или только из сети Prometheus. `data_source_proxy_whitelist` — не пустой, когда источники известны.
6. Anti-affinity по ноде. PDB: не эвакуировать все реплики одним drain.
7. Понимать дефолт: **каждый** под считает **все** alert rules → нагрузка на Prometheus × N. Третью реплику «для красоты» без спроса к источнику не ставить.
8. Критичные алерты, которые должны жить без Grafana — дублировать во **внешнем** Alertmanager Prometheus.

Независимый Grafana в ЦОД-2 — отдельный Helm, **своя** Postgres, тот же git. Не `ha_peers` на сервис чужого ЦОДа.

### Масштабирование (когда появятся цифры)

1. Замерить зрителей, число дашбордов, refresh, число alert rules.
2. Больше людей → реплики Grafana + CPU/RAM **внутри ЦОДа**. Не умножать оценку алертов бездумно.
3. Терабайты рядов → масштаб Prometheus/Mimir, не `replicas` Grafana.
4. Тиры вендора (Small/Medium/Large) — стартовые точки, не смета.

### Проверка прода (пока это не пройдено — это не прод)

1. UI 13.2.0, `/api/health`, запись в Postgres (созданный folder жив после рестарта пода).
2. `grafana_alertmanager_cluster_members` = числу реплик **этого** ЦОДа.
3. Убить под Grafana: LB жив, UI жив, алерт не задублировался (gossip есть).
4. Контролируемый алерт дошёл до contact point.
5. Provisioning из git перезатирает клик в UI (если git — правда).
6. Учение «ЦОД-1 выключен»: либо UI ЦОД-2 с git поднялся, либо честный простой. Restore Postgres Grafana на стенде.

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + git / независимый UI)

| Сильное | Слабое |
|---|---|
| Gossip и Postgres не зависят от межЦОДового RTT | Падение ЦОД-1 = нет этого Grafana, пока DR/второй экземпляр |
| Один набор дашбордов через git | Два независимых Grafana без git = две правды алертов |
| Официальная схема HA внутри площадки | Admin Grafana = ключ к паролям источников |
| | Клиентский 3000 в штате может идти через город — TLS и сеть ваши |

**Не готов к проду**, если: одна реплика с SQLite; три реплики без `ha_peers`; `admin`/`admin` или дефолтный `secret_key`; `latest`; Grafana в интернет; reporting/snapshots наружу; дашборды только в UI без git и без бэкапа Postgres; stretch БД/`ha_peers` на 2–3 ЦОДа; Dev-манифест скопирован в бой.

---

## Источники

- Релиз 13.2.0: https://github.com/grafana/grafana/releases/tag/v13.2.0
- Установка, SQLite vs Postgres 12+: https://grafana.com/docs/grafana/latest/setup-grafana/installation/
- HA: https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/
- Alerting HA, 9094 TCP+UDP: https://grafana.com/docs/grafana/latest/alerting/set-up/configure-high-availability/
- Helm 12.11.2: https://github.com/grafana-community/helm-charts/releases/tag/grafana-12.11.2
- Правила и пробелы: `Grafana.md`
