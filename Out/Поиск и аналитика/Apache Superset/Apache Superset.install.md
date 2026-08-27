# Apache Superset 6.1.0 — установка (учебный контур)

Superset — веб-приложение: люди рисуют графики и дашборды **SQL-запросами** к базам, которые вы подключили. Само **не хранит** терабайты озера: цифры живут в складе (ClickHouse / PostgreSQL-витрина). Ставите **свою** копию **6.1.0**, не Preset Cloud.

**Допущение:** закрытая сеть, **одна** Linux-машина, учебный стенд. Боевой запуск сюда не копировать. Compose вендор **не** поддерживает как устойчивость и бой: [Docker Compose](https://superset.apache.org/docs/installation/docker-compose). Официальной поддержки **Windows нет**.

Официальный путь «пощупать»: **Docker Compose** + файл `docker-compose-image-tag.yml` и тег образа. Docker — программа, которая запускает **образ** (упакованная программа с зависимостями) как **контейнер**; Compose поднимает пачку контейнеров по YAML. Команды — на Linux-машине стенда (дока Compose допускает ещё Mac OSX), не в PowerShell Windows.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна Linux-машина рядом с учебным складом, не «кластер на три дата-центра». На ней — Docker Compose. UI слушает **8088** (дефолт `SUPERSET_PORT` и `ports` в `docker-compose-image-tag.yml` тега **6.1.0**). Привязать к `127.0.0.1`: в YAML вендора стоит `8088:8088` (это все интерфейсы хоста). Каталог метаданных — том Compose `db_home` (**не бэкапится**, так пишет вендор).

```mermaid
flowchart LR
  U["Браузер на хосте"] -->|"http :8088"| WEB["Compose: веб Gunicorn"]
  WEB --> PG["Postgres метаданных\nтом db_home"]
  WEB --> RD["Redis"]
  WEB -->|"SQL read-only"| LAKE["ClickHouse / витрина PG"]
```

Это **не** кворум: несколько процессов одного ПО с общей базой метаданных. Живой стек — **в одной площадке**. Порога RTT у проекта нет.

**Сколько.** Цифр «хватит N ядер / M ГБ» для Compose у вендора **нет**. Не путать с примером Gunicorn `-w 10` на [Configuring](https://superset.apache.org/docs/configuration/configuring-superset) — это не смета стенда. Замечание про **16 ГБ RAM** на странице Compose относится к **интерактивной** сборке фронта (`docker-compose.yml` + npm), не к `docker-compose-image-tag.yml`. Сметы боя здесь нет: нагрузки в контексте нет.

**Сильная сторона:** минуты, официальный «boot up an official release». **Слабая:** один хост; том метаданных без бэкапа; падение машины = нет UI и дашбордов.

**Критично:** **8088** не в интернет. Не `TAG=latest`. Не стартовать с `SUPERSET_SECRET_KEY=TEST_NON_DEV_SECRET` (так в `docker/.env` тега 6.1.0), не `CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET`, не `thisISaSECRET_1234`. `admin` / `admin` — только изоляция. SQLite как metadata URI при двух репликах — ловушка; этот Compose поднимает **Postgres** (`postgres:17`), не SQLite.

---

## Установка для новичка

Страница шагов: https://superset.apache.org/docs/installation/docker-compose  
Не брать `docker-compose.yml` (сборка фронта) и не `docker-compose-light.yml` (порт **9001**, это другой контур).

### Что должно быть до установки

**Есть:**

- Linux x86_64 (или Mac OSX по доке Compose), закрытая сеть, вход с jump-хоста / VPN
- Docker и git; команда **`docker compose`** (два слова, Compose v2), не устаревающий `docker-compose`
- свободны **8088** на localhost, **5432/6379** снаружи не публиковать
- `openssl` (ключ приложения)

**Нет** (и не должно появиться на этой машине):

- публикация 8088 в интернет
- Windows как целевая ОС стенда
- `latest` / прыжок на другой minor «на глаз»
- `.env-local` в git (файл в `.gitignore` репозитория Superset)
- боевое озеро учёткой superuser

### Этап 1. Проверка машины

**Что делаем:** убеждаемся, что Docker жив и 8088 свободен.

```bash
uname -s
docker version
docker compose version
ss -ltn | grep 8088 || true
openssl version
```

Успех: Linux (или Darwin); `docker compose` отвечает версией; 8088 не занят.

### Этап 2. Клон и тег 6.1.0

**Что делаем:** берём скрипты Compose **того же** тега, что образ. Пример доки всё ещё пишет `TAG=5.0.0` и оговорку про суффикс `-dev` (в `-dev` есть `psycopg2-binary`). Здесь пин **6.1.0**; драйверы — в `requirements-local.txt` (этап 4). Сверить digest образа на Docker Hub **до** `up`.

```bash
git clone --depth=1 https://github.com/apache/superset.git
cd superset
export TAG=6.1.0
git fetch --depth=1 origin tag "$TAG"
git checkout "$TAG"
```

Успех: `git describe --tags` (или `git rev-parse --abbrev-ref HEAD`) указывает на **6.1.0**.

### Этап 3. Порт только на localhost

**Что делаем:** в `docker-compose-image-tag.yml` у сервиса `superset` меняем публикацию порта.

Было: `8088:8088`  
Стало: `127.0.0.1:8088:8088`

Успех: `grep -n 8088 docker-compose-image-tag.yml` показывает `127.0.0.1:8088:8088`. Образ в YAML по умолчанию идёт через Scarf Gateway (`apachesuperset.docker.scarf.sh/apache/superset`). На закрытом контуре замените на `apache/superset:${TAG:-latest-dev}` — [тот же раздел Compose](https://superset.apache.org/docs/installation/docker-compose#docker-compose-tips--configuration).

### Этап 4. Секрет и драйверы **до** первого `up`

**Что делаем:** свой ключ приложения и пароль admin — в `docker/.env-local` (перекрывает `docker/.env`, в git не попадёт). Lean-образ **6.1.0** **не** содержит драйверы складов; bootstrap ставит пакеты из `docker/requirements-local.txt` (`uv pip`, с 6.1.0).

```bash
cd superset   # корень клона
openssl rand -base64 42
```

`docker/.env-local` (подставьте ключ и пароль, не примеры):

```bash
SUPERSET_SECRET_KEY=
ADMIN_PASSWORD=
SCARF_ANALYTICS=false
SUPERSET_LOAD_EXAMPLES=yes
```

`SUPERSET_LOAD_EXAMPLES=yes` — учебные дашборды вендора; на слабой машине поставьте `no` (дока: загрузка жрёт CPU). Не оставлять `SUPERSET_SECRET_KEY=TEST_NON_DEV_SECRET`.

```bash
printf '%s\n' 'psycopg2-binary' 'clickhouse-connect>=0.6.8' > docker/requirements-local.txt
```

Успех: `grep SUPERSET_SECRET_KEY docker/.env-local` — не `TEST_NON_DEV_SECRET` и не пусто; в `requirements-local.txt` есть оба пакета.

### Этап 5. Подъём стека

**Что делаем:** Compose скачивает образы, init гоняет миграцию, `superset init` и создаёт admin. Первый старт — минуты (примеры, если включены).

```bash
export TAG=6.1.0
docker compose -f docker-compose-image-tag.yml pull
docker compose -f docker-compose-image-tag.yml up -d
```

Успех: команда без ошибки. Смотреть init:

```bash
docker compose -f docker-compose-image-tag.yml ps
docker logs superset_init
```

Контейнер `superset_init` — **exited 0** / completed. В логе — создание admin, не `Refusing to start due to insecure SECRET_KEY`. Образы — тег **6.1.0**, не `latest`. Ожидаются: `superset_app`, `superset_db` (`postgres:17`), `superset_cache` (Redis 7), `superset_worker`, `superset_worker_beat`, плюс завершённый init.

### Этап 6. Стенд живой

**Что делаем:** проверка процесса веба, не склада.

```bash
curl -sS http://127.0.0.1:8088/health
```

Успех: HTTP 200, тело `OK`. Это **не** проверка Postgres, Redis и ClickHouse ([Configuring](https://superset.apache.org/docs/configuration/configuring-superset)).

**Чего этот стенд не доказывает:** отказ площадки; HA Postgres/Redis; нагрузка аналитиков; выборы лидера (их в продукте нет); что N веб-реплик не съедят `max_connections` озера; SQL Lab по боевому эталону; отчёты PNG / SMTP. Два Celery beat не запускать: в этом YAML beat **один**; два = двойные письма. Удалили том `db_home` — дашборды пропали.

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL:** `http://127.0.0.1:8088` — порт **8088**. Браузер часто уводит на `https` — нужен **http** ([Compose, § Log in](https://superset.apache.org/docs/installation/docker-compose)). Не с мира, с той же машины / SSH-туннеля.

**Учётка.** Дефолт вендора Compose / Helm init: логин **`admin`**, пароль **`admin`**. На этом стенде пароль — значение `ADMIN_PASSWORD` из `docker/.env-local`, заданное **до** первого `up` (`docker/docker-init.sh`: `ADMIN_PASSWORD` с запасным `admin`). Если `.env-local` не задали и init уже прошёл — это **`admin` / `admin`**.

**Смена пароля.** Сразу после входа сменить в UI (профиль пользователя Flask-AppBuilder). Отдельной страницы вендора «как сменить пароль» нет. Повторный `ADMIN_PASSWORD` в `.env-local` **после** успешного init пароль в базе **не** меняет. Учебный пароль в бой не копировать.

**SECRET_KEY.** Ключ Flask: подпись cookie **и** шифрование паролей источников в метаданных. С 2.1.0 дефолт на не-debug **запрещает старт**. Рекомендация вендора: `openssl rand -base64 42` ([Configuring](https://superset.apache.org/docs/configuration/configuring-superset), [Kubernetes](https://superset.apache.org/docs/installation/kubernetes)).

Не оставлять:

| Строка | Откуда |
|---|---|
| `TEST_NON_DEV_SECRET` | `docker/.env` 6.1.0 |
| `CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET` | `superset/constants.py` 6.1.0 |
| `thisISaSECRET_1234` | дефолт Helm, страница Kubernetes |
| `YOUR_OWN_RANDOM_GENERATED_SECRET_KEY` | пример Configuring |

Смена ключа **после** подключения источников: `PREVIOUS_SECRET_KEY` + `superset re-encrypt-secrets`. Иначе пароли складов в метаданных не читаются.

Роли: на стенде вы Admin. В бою Gamma на данные, не все Admin; `superset init` **перезаписывает** права встроенных ролей.

---

## Подключение к своей системе

Аналитики — браузер на **8088**. Источники — **SQLAlchemy URI** к витрине, не к Kafka как к таблице. Официальный образ **не** содержит диалекты: пакет уже в `requirements-local.txt` (этап 4); без него UI сохранится, Test Connection упадёт.

В UI: **Settings → Data: Database Connections → + DATABASE** → URI → **Test Connection** → **Connect** ([Connecting to Databases](https://superset.apache.org/docs/databases/)).

Из контейнера `localhost` — это **сам** контейнер, не хост. Хост: `host.docker.internal` (Mac / часть Linux) или шлюз моста (`docker network inspect bridge` → `Gateway`) — [Compose, § Connecting](https://superset.apache.org/docs/installation/docker-compose).

| Склад | Драйвер | URI (шаблон вендора) | Порт фактов |
|---|---|---|---|
| ClickHouse | `clickhouse-connect>=0.6.8` | `clickhousedb://<user>:<password>@<host>:<port>/<database>` | HTTP склада, обычно **8123** (см. `ClickHouse.install.md`) |
| PostgreSQL-витрина | `psycopg2` / `psycopg2-binary` | `postgresql://{username}:{password}@{host}:{port}/{database}` | **5432** |

Пример ClickHouse в доке (облачный стенд Altinity): `clickhousedb://demo:demo@github.demo.trial.altinity.cloud/default?secure=true`. Локально без пароля вендор показывает `clickhousedb://localhost/default` — на **вашем** складе без пароля не оставлять; `localhost` из контейнера заменить на хост/имя сервиса.

Учётка склада — **только SELECT** (лучше отдельная роль БД, лучше реплика, не primary эталона). Не операционные БД Camunda / интеграционного API. SQL Lab — консоль **от имени этой учётки**, не «безопасно, потому что UI».

Потом: один **dataset** → **Explore** → чарт не пустой. Это проверка витрины, не `/health`.

### Секреты (не в git)

| Секрет | Где живёт | Куда не класть |
|---|---|---|
| `SUPERSET_SECRET_KEY` | `docker/.env-local` | git, чат, образ |
| `ADMIN_PASSWORD` / пароль после смены | `.env-local` до init, потом сейф / Vault | git |
| Пароль Postgres метаданных (`DATABASE_PASSWORD` / `POSTGRES_PASSWORD`, дефолт compose **`superset`**) | `docker/.env` / `.env-local` | git; на стенде известен — в бой не тащить |
| Пароль ClickHouse / витрины PG | UI; в метаданных **зашифрован** `SECRET_KEY` | git, скриншоты URI |
| Дашборды | export YAML/ZIP + git | единственная копия только в томе `db_home` |

В git — процедура и имена переменных без значений.

### Чем продукт не является

| Сосед | Чем отличается |
|---|---|
| Grafana | Инфра-дашборды и алерты платформы. Superset — предметная область поверх SQL |
| Luxms BI | Платный российский BI, другой путь поставки (пакеты, лицензия), не этот образ |
| ClickHouse | Склад фактов. Superset — UI запросов, терабайты не хранит |
| Kafka | Шина событий, не таблица. Смотреть проекцию в SQL |
| Metadata Postgres | Пользователи и дашборды Superset, **не** озеро клиентов |

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Compose: не бой, не Windows, `TAG`, `admin`/`admin`, `:8088`, том метаданных без бэкапа, Scarf, `localhost` ≠ хост, 16 ГБ только для npm-dev | https://superset.apache.org/docs/installation/docker-compose |
| `SECRET_KEY` / `openssl rand -base64 42`, Postgres метаданных **10.X–16.X** (таблица), `/health` = `OK`, Gunicorn `-w 10` — пример, `ENABLE_PROXY_FIX`, `re-encrypt-secrets` | https://superset.apache.org/docs/configuration/configuring-superset |
| Helm, `thisISaSECRET_1234`, init `admin`/`admin`, драйверы в образе (не apt на старте) | https://superset.apache.org/docs/installation/kubernetes |
| Драйверы, URI ClickHouse `clickhousedb://…`, PostgreSQL `postgresql://…`, `requirements-local.txt` | https://superset.apache.org/docs/databases/ |
| Релиз **6.1.0** | https://github.com/apache/superset/releases/tag/6.1.0 |
| Helm-чарт **0.21.1**, `appVersion` 6.1.0 | https://artifacthub.io/packages/helm/superset/superset/0.21.1 |
| `docker-compose-image-tag.yml` 6.1.0: Scarf, `postgres:17`, `8088:8088`, свой SECRET в `.env` | https://github.com/apache/superset/blob/6.1.0/docker-compose-image-tag.yml |
| `docker/.env` 6.1.0: `SUPERSET_PORT=8088`, `SUPERSET_SECRET_KEY=TEST_NON_DEV_SECRET` | https://github.com/apache/superset/blob/6.1.0/docker/.env |
| Асинхронный SQL Lab / Celery | https://superset.apache.org/docs/configuration/async-queries-celery |
| Зачем продукт, порты, железо | `Apache Superset.md` |
| Словарь | `Apache Superset.info.md` |
| Схема стыковки с платформой | `Apache Superset.shema.md` |
| Роль консультанта | `Apache Superset.consultant.md` |
| Склад ClickHouse (порт 8123, учётка) | `ClickHouse.install.md` |
| PostgreSQL платформы | `../БД и хранилища/PostgreSQL.install.md` |

**В доке вендора нет (и здесь не выдумано):** CPU/RAM «хватит для учебного Compose»; порог RTT между дата-центрами; отдельная страница «сменить пароль admin в UI»; обещание, что lean-тег **6.1.0** без `requirements-local` подключится к Postgres метаданных (страница Compose для image-tag всё ещё советует суффикс `-dev` на примере **5.0.0**); PostgreSQL **17** в таблице Configuring (там **10.X–16.X**, при том что Compose 6.1.0 поднимает `postgres:17`); digest образа 6.1.0 — смотреть Docker Hub перед `pull`.
