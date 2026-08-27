# Apache Superset 6.1.0 — термины и сокращения

Словарь к файлу `Apache Superset.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа: память и открытые файлы. «Поставить Superset» — не один процесс, а несколько: процесс, который отдаёт HTTP UI/API; отдельные процессы, которые забирают долгие задачи из очереди; ровно один процесс-будильник расписаний. Несколько копий UI — одинаковые процессы с общей базой метаданных и общей очередью, без выборов «кто главный».

**Файл / каталог на диске** — то, что остаётся после рестарта. Метаданные (дашборды, пользователи) в проде — в PostgreSQL, не в файле пода. SQLite как metadata URI — файл на диске одного пода: вторая реплика видит другую «правду».

**TCP-порт** — номер в сети. UI/API web **8088** (дефолт Helm `service.port`). Redis **6379**, PostgreSQL **5432** / MySQL **3306** — не Superset. Celery Flower **5555** (Helm, дефолт выкл). Websockets чарта **8080** (дефолт выкл). Пример Gunicorn в доке `-b 0.0.0.0:6666` — иллюстрация, не дефолт образа.

**ЦОД** — отдельная площадка. Stretch = web/worker в нескольких ЦОДах, но **одна** metadata DB и **один** Redis.

**RTT** — время туда-обратно. Порога в мс нет. Мерить **5432** и **6379**. Metadata DB и Redis чувствуют задержку как любой синхронный клиент.

**TLS** — шифрование байтов в сети. Обычно на HAProxy/Ingress перед 8088. `ENABLE_PROXY_FIX` говорит Flask доверять `X-Forwarded-*`. ГОСТ/СКЗИ не заявлено. Официальной поддержки Windows нет.

**Под (Pod)** — контейнер(ы) Kubernetes. Web — Deployment. PVC SQLite на web при `replicas > 1` — ловушка.

**Helm / чарт** — `superset/superset` **0.21.1** (18 июля 2026), `appVersion` **6.1.0**. Зависимости чарта: Bitnami PostgreSQL и Redis — стендовые сабчарты, не прод. Operator **0.2.0** — другой инсталлятор, PostgreSQL и Redis не ставит.

**PDB** — в values чарта дефолт выключен. В проде не убивать все web сразу (`minAvailable: 1`).

**Superset** — веб-приложение: люди рисуют графики и дашборды **SQL-запросами** к базам, которые вы подключили. Само **не хранит** терабайты озера.

**Metadata database (метаданные)** — отдельная БД Superset: пользователи, дашборды, датасеты, сохранённые запросы, расписания алертов. Это **не** озеро клиентов. Для прода: PostgreSQL **10.x–17.x** или MySQL **5.7 / 8.x**. SQLite настоятельно не рекомендован.

**Data source / Database (в UI)** — подключение к *чужой* СУБД, откуда берутся факты. Пароль лежит в метаданных, зашифрованный `SECRET_KEY`.

**Dataset** — таблица или SQL-вид, из которого рисуют чарт. Права часто вешаются на датасет, не на отдельный график.

**Chart / Slice** — один график = один запрос + визуализация.

**Dashboard** — экран из чартов. Пока не **published** — его не видят остальные (правило документации Security).

**Explore** — конструктор чарта без SQL Lab.

**SQL Lab** — встроенный SQL-редактор. Консоль к вашей базе **от имени учётки, которой подключили datasource**. Не «безопасно, потому что внутри UI».

**Row Level Security (RLS)** — фильтры Superset вида «этому субъекту добавляй `WHERE client_id IN (...)`». Это **не** RLS PostgreSQL и **не** firewall. Обойти можно SQL Lab / другой datasource / прямой JDBC к складу.

**Subject (субъект)** — кому выдают доступ: пользователь, группа или роль. В 6.x пикеры по умолчанию предлагают **Users и Groups**.

**Admin / Alpha / Gamma / sql_lab / Public** — встроенные роли. `superset init` **перезаписывает** их права. Admin = полный контроль, включая CSS/Jinja; вендор: векторы «нужен Admin» **не** считаются CVE. Gamma смотрит датасеты, к которым выдан доступ. Public — роль **анонимов**.

**`SECRET_KEY` / `SUPERSET_SECRET_KEY`** — ключ Flask: подпись cookie сессии **и** шифрование секретов в метаданных. С 2.1.0 дефолт на не-debug **запрещает старт**. Смена ключа без `re-encrypt-secrets` = не читаются пароли источников. Рекомендация вендора: `openssl rand -base64 42`. Потом: `PREVIOUS_SECRET_KEY` + `superset re-encrypt-secrets`.

**`CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET`** — значение `CHANGE_ME_SECRET_KEY` в `superset/constants.py` тега **6.1.0**. Процесс отказывается считать это секретом.

**`thisISaSECRET_1234`** — дефолт, который документация Kubernetes называет ключом helm-инсталляции, если свой не задали. Другая известная строка.

**`superset_config.py`** — файл-оверрайд. Кладётся в `PYTHONPATH` или путь `SUPERSET_CONFIG_PATH`. Копировать весь `config.py` не нужно.

**WSGI** — договор, как отдельная программа принимает HTTP и вызывает функцию Python-приложения.

**Gunicorn** — эта отдельная программа для Python: слушает TCP, запускает несколько рабочих процессов, вызывает код Superset (написан на Flask). Вендор рекомендует режим async / gevent. Пример в доке: **10** воркеров (`-w`), `--worker-connections 1000`, `--timeout 120` — пример, не смета. `superset run` / `flask run` — встроенный однопроцессный запуск Flask, не прод. BigQuery Python SDK **несовместим с gevent**.

**gevent** — режим воркера Gunicorn: много соединений на процесс через кооперативную многозадачность, не отдельный OS-thread на каждый запрос.

**Celery worker** — отдельный процесс: забирает задачи из очереди (брокер Redis). Длинные SQL Lab («Run Async»), прогрев кэша, скриншоты алертов. Без воркеров асинхронные задачи **не выполняются**. Каждая реплика открывает свои коннекты к складу.

**Celery beat** — отдельный процесс-будильник расписаний (Alerts & Reports). **Должен быть один**. Два beat = двойные письма.

**Broker (Celery)** — очередь задач в Redis (в проде). Дефолт в `config.py` 6.1.0 — **SQLite**. Для нескольких подов SQLite ломает async.

**Results backend** — куда кладут результат длинного SQL Lab. Дефолт — SQLite / `None`. Для нескольких подов нужен **общий** Redis/S3, не диск пода. Дока приводит `flask_caching.backends.rediscache.RedisCache`.

**CACHE_CONFIG / DATA_CACHE_CONFIG** — кэш объектов и результатов чартов. Дефолт **`NullCache`**: каждый refresh бьёт в склад HTTP/SQL.

**FILTER_STATE_CACHE / EXPLORE_FORM_DATA_CACHE** — состояние фильтров дашборда / формы Explore. Дефолт — таблица в **метаданных** (`SupersetMetastoreCache`). HA web-реплик это переживает.

**NullCache** — «кэша нет». Дашборд с автообновлением 10 секунд при NullCache = постоянный шторм SQL.

**Talisman / CSP** — процесс web ставит HTTP-заголовок Content-Security-Policy (какие скрипты/iframe можно грузить). `TALISMAN_ENABLED` по умолчанию **True**. Выключать «чтобы заработало Mapbox/iframe» — снимать эту защиту XSS.

**CSRF (`WTF_CSRF_ENABLED`)** — защита форм: браузер должен прислать токен, который выдал этот же сайт. Дефолт **True**. Срок токена в 6.1.0 — **1 неделя** (`WTF_CSRF_TIME_LIMIT`).

**`ENABLE_PROXY_FIX`** — сказать Flask: доверяй `X-Forwarded-*` от балансировщика. Дефолт **False**. За HTTPS-прокси без этого ломаются cookie/`redirect_uri` OAuth.

**`/health`** — HTTP 200 и тело `OK`, если веб-процесс жив. Это **не** проверка Postgres/Redis/склада.

**Provisioning / import** — дашборды и datasources можно залить YAML/ZIP (GitOps), а не только кликами.

**Alerts & Reports** — расписание: выполни запрос / сними PNG дашборда, пошли email/Slack. Нужны: feature flag `ALERT_REPORTS`, **beat**, worker, **webdriver**, SMTP. Дефолт флага **False**.

**Webdriver** — отдельный процесс headless-браузера (Chrome/Firefox) в worker-поде. Рисует PNG для письма. `WEBDRIVER_BASEURL` — внутренний Service (`http://...:8088/`); `WEBDRIVER_BASEURL_USER_FRIENDLY` — внешний HTTPS.

**Embedded / Guest token** — встроить дашборд в чужой сайт по JWT. Feature flag `EMBEDDED_SUPERSET` дефолт **False**. Дефолтные JWT-секреты в `config.py` — тестовые строки `test-...-change-me`.

**Row limit (`ROW_LIMIT` / `SQL_MAX_ROW`)** — сколько строк чарт / SQL Lab могут вытащить. В 6.1.0: `ROW_LIMIT = 50000`, `SQL_MAX_ROW = 100000`. Защита UI, не склада.

**Jinja (`ENABLE_TEMPLATE_PROCESSING`)** — подставлять шаблоны в SQL датасета. Дефолт **False**. Включение без политики = пользователь влияет на SQL.

**Scarf** — телеметрия проекта: gateway на pull образа и пиксель в UI. Для закрытого контура: образ `apache/superset`, `SCARF_ANALYTICS=false`.

**Stretch** — одни и те же web/worker-поды в нескольких ЦОДах, одна metadata DB и один Redis.

**Redis** — отдельный процесс (порт 6379): кэш, broker Celery, results backend, опционально server-side session / rate-limit storage. Helm: `redis_celery_db` 0, `redis_cache_db` 1. Сабчарт Bitnami — стенд.

**Init Job** — `superset db upgrade`, `superset init`, создание admin. Разово на install/upgrade. Не часть runtime HA. `createAdmin` на проде — один раз, потом выключить.

**`admin` / `admin`** — первый admin Docker Compose / Helm init (`init.adminUser.password`).

**`SESSION_COOKIE_SECURE`** — дефолт `False`. В проде `True` при HTTPS.

**`SESSION_SERVER_SIDE`** — дефолт `False`: сессия в cookie, sticky на LB не требуется. Если включить + Redis — тоже общий Redis.

**`RATELIMIT_ENABLED`** — только если `SUPERSET_ENV=production`. `AUTH_RATE_LIMIT` = `5 per second` когда лимиты включены. Без `RATELIMIT_STORAGE_URI` на Redis счётчики **по процессу**.

**`PUBLIC_ROLE_LIKE`** — дефолт `None`. Включить «как Gamma» + датасеты клиентов = аноним читает озеро.

**`PREVENT_UNSAFE_DB_CONNECTIONS`** — дефолт `True`. Не выключать: sqlite/local files — вектор.

**`DATASET_IMPORT_ALLOWED_DATA_URLS`** — дефолт `[r".*"]`: импорт датасета может тянуть URL откуда угодно (SSRF). Сузить список.

**SSRF** — сервер по указанию клиента открывает HTTP на внутренний адрес. Импорт датасета с `.*` — такая точка.

**`FAB_ADD_SECURITY_API`** — в `config.py` 6.1.0 **`True`**. Страница Security говорит «включите, по умолчанию выкл» — для этого тега верьте файлу: Security REST включён.

**`runAsUser` Helm** — дефолт **`0` (root)**. README: в проде не рекомендуется; пример non-root **1000**, если bootstrap больше не нужен.

**`AUTH_USER_REGISTRATION_ROLE`** — в примере Helm OAuth стоит `"Admin"`. В прод не копировать.

**`AUTH_ROLES_MAPPING` / `AUTH_ROLES_SYNC_AT_LOGIN`** — маппинг групп IdP и синхронизация ролей при каждом входе. Иначе роли «залипнут» с первого входа.

**`DISALLOWED_SQL_FUNCTIONS`** — чёрный список SQL-функций. Документация: дополнение, не замена прав на складе.

**`VIEWER_PROMISCUOUS_MODE`** — позволяет зрителям дашборда обходить проверки датасета. Не включать.

**`SSH_TUNNELING`** — дефолт `False`. Туннель из пода в чужую сеть.

**`GLOBAL_ASYNC_QUERIES`** — в 6.1.0 `@lifecycle: testing`. Чартовый websocket — community-образ `oneacrefund/superset-websocket:latest`. Для первого прода выкл.

**`SQLLAB_TIMEOUT`** — дефолт **30s** для синка. `SUPERSET_WEBSERVER_TIMEOUT` дефолт **60s**. Согласовать со statement_timeout склада и timeout Ingress.

**`task_acks_late`** — в дефолтном CeleryConfig проекта **False**. Пример async-дока показывает `True`. Про повтор задач при убийстве worker.

**prune_query / prune_logs** — beat-задачи очистки истории SQL Lab. Пока не включены, metadata DB распухает запросами.

**`FAB_API_KEY_ENABLED`** — дефолт выкл. Включать только под CI с scopes.

**`ALERT_REPORTS_WEBHOOK_HTTPS_ONLY`** — дефолт `True`.

**uv pip** — с 6.1.0 в Docker пакеты ставят через `uv pip install`, не старый pip в bootstrap. Для прода: свой образ с драйвером склада в CI, не `apt` на каждом старте от root.

**Flask-AppBuilder** — библиотека ролей/OAuth/LDAP, на которой сидит Superset.

**Sticky session** — для дефолтных cookie-сессий **не** обязательна.

**SPOF** — падение metadata DB = нет логина и дашбордов, даже если поды зелёные. Beat = SPOF расписаний, не UI.

**Grafana** — дашборды инфраструктуры. Superset — предметная область. Не заменять одно другим.

**Image tag `latest`** — не брать. Прод: **6.1.0** digest.

**Apache License 2.0** — лицензия дистрибутива.

Источники формулировок: глоссарий и тело `Apache Superset.md`. Новых порогов RTT и размеров диска здесь нет.
