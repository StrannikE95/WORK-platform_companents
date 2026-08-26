# Apache Superset 6.1.0 — установка и конфигурирование

Связанный документ (глоссарий, SECRET_KEY, Celery, почему так): `Apache Superset.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного стека (одна metadata DB + один Redis) на несколько ЦОДов **не делаем**: web/worker — stateless, но Postgres и Redis синхронны; порога RTT у проекта нет.

Версии: **Apache Superset 6.1.0**, образ `apache/superset:6.1.0`. На Kubernetes официальная страница установки описывает **Helm** `https://apache.github.io/superset`, чарт **`superset/superset` 0.21.1** (`appVersion` 6.1.0).  
**Superset Operator 0.2.0** — альтернатива (BYO Postgres/Redis), **не** дефолт страницы «Installing on Kubernetes».  
Документация: https://superset.apache.org/docs/installation/kubernetes

Нужны **PostgreSQL** (метаданные, в доке 10.x–17.x) и **Redis/Valkey** (кэш, Celery broker, results). `SECRET_KEY` **обязан** быть своим: с 2.1.0 дефолт на не-debug запрещает старт; Helm-дока ещё знает строку `thisISaSECRET_1234` — её тоже не оставлять.

---

## Допущения этой инструкции

1. **Stretch запрещён.** Web, worker, beat, metadata Postgres и Redis **одного** стека — **внутри одного ЦОДа**. Между ЦОДами — независимый стек (своя БД, свой Redis, git-export дашбордов) **или** люди ходят в UI ЦОД-1.
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`). Сабчарты Bitnami Postgres/Redis чарта — **стенд**, в прод `postgresql.enabled: false`, `redis.enabled: false`.
3. Dev — изолированная сеть; `admin`/`admin` только там.
4. Нагрузки нет — нет числа реплик «хватит». Терабайты живут в **складе**, не в metadata DB.
5. Склад/озеро как SQL-источник будет. Без витрины Superset — пустая рамка.
6. Для 2 ЦОДов: стек в ЦОД-1; ЦОД-2 — доступ к UI ЦОД-1 **или** свой стек. Для 3 ЦОДов: то же. Третий ЦОД **не** второй writer в ту же metadata DB.
7. Celery beat — **ровно 1**. Alerts & Reports в первом проде не обязательны (иначе Chromium + SMTP).
8. Официальной поддержки Windows нет (дока Compose).

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** Explore, один SQL-источник, понять отличие от Grafana. **Не** цель: отказ ЦОДа и SQL Lab по боевому озеру.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 8088 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

Compose вендор **не** поддерживает как HA/прод; для «пощупать» — тег релиза. Пример доки иногда приводит `TAG=5.0.0` — брать **6.1.0**, сверять digest на Docker Hub.

Минимально (один контейнер + внешний Postgres на стенде предпочтительнее SQLite в volume):

```bash
docker run -d --name superset-dev \
  -p 127.0.0.1:8088:8088 \
  -e SUPERSET_SECRET_KEY="$(openssl rand -base64 42)" \
  apache/superset:6.1.0
```

Привязка к `127.0.0.1` обязательна. После старта — `superset db upgrade`, `superset init`, создание admin **внутри** контейнера (команды из getting started текущей 6.1.0). Не стартовать с `CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET`.

Вход: `http://127.0.0.1:8088`. Compose-путь (`docker-compose-image-tag.yml`, `TAG=6.1.0`) — volume метаданных **не бэкапится** (дока).

На Kubernetes Dev: Helm 0.21.1, `replicaCount: 1`, свой `extraSecretEnv.SUPERSET_SECRET_KEY`, сабчарты Postgres+Redis **допустимы на тесте**. **Не** этот values в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Metadata DB | SQLite или bitnami Postgres чарта | Нет HA-цели |
| Redis | сабчарт или без (light) | Нет async-цели |
| Реплики web/worker | 1 | Нет HA |
| Alerts / beat | выкл | Не цель |
| SECRET_KEY | свой, даже на тесте | Иначе не-debug не стартует / известный Helm-ключ |
| SQL Lab | только синтетика | Боевое озеро не подключать |
| `AUTH_USER_REGISTRATION_ROLE=Admin` | **не** копировать из примера OAuth | Привычка уедет в бой |

Чего **не** упрощать: тег **6.1.0**; хотя бы один чарт не пустой; Export дашборда в git; 8088 не с мира; не superuser озера.

### Проверка Dev

1. `/health` → `OK` (это **не** проверка Postgres/склада).
2. Explore живой. Рестарт без volume — дашборды пропали, если SQLite в контейнере.
3. `runAsUser: 0` чарта не считать нормой прода.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты–час, официальный quickstart | Нет HA Postgres/Redis |
| Дешёво показывает, каких витрин нет | Успех на ноутбуке ≠ 5 реплик не убьют `max_connections` озера |
| | `admin`/`admin` и root bootstrap уедут в бой |

Препрод: внешний Postgres, Redis, 2 web, ≥1 worker, свой секрет, read-only datasource — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода web/worker внутри ЦОДа**. Отказ ЦОДа с Postgres метаданных = нет BI, пока DR. Склад — отдельный HA.

### Почему не stretch

Metadata DB и Redis — синхронные клиенты с web/worker. Stretch Postgres на 2–3 ЦОДа — история PostgreSQL, не фича Superset. Два beat = двойные письма. Results backend на диске пода = пользователь на web-A не заберёт результат worker-B.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один стек:

- web **≥ 2**, worker **≥ 2**, beat **= 1** (beat только если Alerts нужны);
- Postgres база `superset` HA **в ЦОД-1** (не та же, что Grafana/Camunda/озеро); TLS `verify-full`;
- Redis HA **в ЦОД-1**; logical DB как в чарте (0 Celery, 1 cache);
- `SECRET_KEY` свой (`openssl rand -base64 42` — рекомендация вендора) **до** datasources;
- образ **6.1.0 с драйвером склада**, собранный в CI, **non-root** (README чарта: `runAsUser: 0` в проде не рекомендуется);
- `postgresql.enabled: false`, `redis.enabled: false`;
- HTTPS LB на 8088, health `/health`; sticky для дефолтных cookie **не** обязателен;
- `ENABLE_PROXY_FIX = True`, `SESSION_COOKIE_SECURE = True`, `SUPERSET_ENV=production`;
- PDB на web; в values чарта PDB дефолт **выкл** — включить;
- чарт **0.21.1**, pin digest образа.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: весь стек. ЦОД-2: люди ходят в UI ЦОД-1 **или** независимый стек (своя БД/Redis, git-export) | Нет этого BI, пока DR/второй стек. Склад может быть жив — панели failed |
| **3 ЦОДа** | То же; три полных Superset — тройные права, обычно хуже | То же |

Не открывать SQL Lab на primary SoT. Читать **реплику** озера. Grafana — другой класс дашбордов (инфра), не замена.

### Предпосылки прода

- HA Postgres и HA Redis в ЦОД-1 (`PostgreSQL.install.md` и ваш Redis).
- Vault: SECRET_KEY, пароль БД, Redis, SSO, пароли складов.
- NetworkPolicy: 8088 от LB; 5432/6379 только от web/worker/beat/init; склад — только от web/worker.
- Драйвер склада в образе; с 6.1.0 в Docker пакеты через **`uv pip install`**, не старый bootstrap `pip` в каждом старте от root.

### Установка (Helm, ЦОД-1)

```bash
helm repo add superset https://apache.github.io/superset
helm upgrade --install superset superset/superset --version 0.21.1 \
  --namespace bi --create-namespace \
  -f superset-prod-values.yaml
```

Смысл values:

```yaml
image:
  repository: apache/superset   # не scarf.sh в закрытом контуре
  tag: "6.1.0"
extraSecretEnv:
  SUPERSET_SECRET_KEY: "<свой>"
supersetNode:
  replicas:
    replicaCount: 2
supersetWorker:
  replicas:
    replicaCount: 2
supersetCeleryBeat:
  enabled: false          # true только с webdriver/SMTP и beat=1
postgresql:
  enabled: false
redis:
  enabled: false
```

Init Job: `db upgrade` + `init` один раз; `createAdmin` затем выкл. Breaking чарта 0.21.1: labels `app.kubernetes.io/*` — upgrade со старого релиза без удаления Deployment может упасть на immutable selector (README).

Flower (5555) не включать без нужды. `supersetWebsockets` / community-образ `latest` — не тащить; `GLOBAL_ASYNC_QUERIES` в 6.1.0 помечен testing.

### Конфигурирование прода

1. Нет `admin`/`admin`. SSO + `AUTH_ROLES_SYNC_AT_LOGIN`; регистрация **не** в Admin.
2. `CACHE_CONFIG` / `DATA_CACHE_CONFIG` — Redis, не NullCache при «высокой нагрузке».
3. `RESULTS_BACKEND` — общий Redis, не FileSystemCache.
4. `TALISMAN_ENABLED` оставить. `PREVENT_UNSAFE_DB_CONNECTIONS` не выключать. `DATASET_IMPORT_ALLOWED_DATA_URLS` сузить (дефолт `.*` — SSRF).
5. `SCARF_ANALYTICS=false`. Public/anonymous на ПДн не включать. `ENABLE_TEMPLATE_PROCESSING` без модели угроз не включать.
6. SQL Lab — узкая роль; учётка БД **SELECT** на витрины. RLS Superset **плюс** GRANT на складе.
7. `RATELIMIT_STORAGE_URI` на Redis, иначе лимиты по процессу.
8. CSV export — не всем Gamma.

Смена SECRET_KEY потом: `PREVIOUS_SECRET_KEY` + `superset re-encrypt-secrets`.

### Масштабирование (когда появятся цифры)

1. Люди → реплики web + Gunicorn (пример доки `-w 10` gevent — **пример**, BigQuery SDK с gevent несовместим).
2. Тяжёлый SQL → worker + склад, не «ещё под».
3. Считать коннекты: (web+worker) × процессы × pool к метаданным **и** к озеру.
4. Не dashboard refresh 10s на тяжёлые агрегации. Агрегации — во витринах склада.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия 6.1.0, `/health`, логин SSO. SECRET_KEY не из констант 6.1.0 и не `thisISaSECRET_1234`.
2. Убить web-под: UI жив. Убить worker: sync-чарт жив, async нет.
3. Недоступен склад: UI жив, панели failed — ожидаемо.
4. Restore metadata Postgres на стенде. Git-export дашбордов существует.
5. Учение «ЦОД-1 выключен»: простой или второй независимый стек. Двух beat нет.

### Сильные / слабые стороны прод-схемы (стек в одном ЦОДе)

| Сильное | Слабое |
|---|---|
| Postgres/Redis не ходят по WAN | Падение ЦОД-1 = нет этого BI |
| Apache 2.0, SECRET_KEY без override не стартует (не-debug) | Admin = ключ ко всем datasource passwords и к SQL |
| Один набор RLS | `/health` слеп к БД/складу |
| | SQL Lab обходит «красивые» чарты |

**Не готов к проду**, если: одна реплика + SQLite/bitnami; `CHANGE_ME_…` / `thisISaSECRET_1234`; `admin`/`admin`; `latest`; `runAsUser: 0` + apt в bootstrap; Public на ПДн; NullCache и «готовы к нагрузке»; SQL Lab под superuser озера; два beat; stretch Postgres на 2–3 ЦОДа; ждут, что Superset хранит терабайты или заменяет Grafana.

---

## Источники

- Релиз 6.1.0: https://github.com/apache/superset/releases/tag/6.1.0
- Configuring: SECRET_KEY, Postgres 10–17, `/health`: https://superset.apache.org/docs/configuration/configuring-superset
- Kubernetes + Helm: https://superset.apache.org/docs/installation/kubernetes
- Helm 0.21.1: https://artifacthub.io/packages/helm/superset/superset/0.21.1
- Operator 0.2.0: https://github.com/apache/superset-kubernetes-operator
- Правила: `Apache Superset.md`
