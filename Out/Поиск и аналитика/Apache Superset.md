# Apache Superset 6.1.0 — развёртывание и настройка

Версия ПО: **Apache Superset 6.1.0** (релизная дата 13 мая 2026; на дату подготовки документа это последний стабильный релиз линии 6.1).  
Документация линейки: https://superset.apache.org/  
Образ: `apache/superset:6.1.0` (Docker Hub). Helm-чарт по умолчанию тянет тот же тег через Scarf Gateway: `apachesuperset.docker.scarf.sh/apache/superset:6.1.0`.  
PyPI: `apache_superset==6.1.0`. Исходный релиз: https://downloads.apache.org/superset/6.1.0/  
Лицензия дистрибутива: **Apache License 2.0**.

На Kubernetes страница установки проекта (обновлялась 26 августа 2026) указывает **Helm-чарт** из `https://apache.github.io/superset`. На Artifact Hub актуальная связка: чарт **`superset/superset` 0.21.1** (18 июля 2026), `appVersion` **6.1.0**.  
Альтернатива: **Superset Kubernetes Operator 0.2.0** (11 августа 2026), репозиторий `apache/superset-kubernetes-operator`, Helm `oci://ghcr.io/apache/superset-kubernetes-operator/charts/superset-operator`. Оператор **не ставит** PostgreSQL и Redis (bring-your-own). CRD на момент README оператора — `superset.apache.org/v1alpha1`. Официальная страница «Installing on Kubernetes» **по-прежнему** описывает Helm, не оператор. В этом файле прод-путь — **Helm 0.21.1 + образ 6.1.0**; оператор — осознанная альтернатива, не «тот же инсталлятор».

Этот текст — не мануал «скопируй `helm install`», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

Apache Superset **не было** в исходном описании архитектуры (Kafka, Camunda, озеро данных, интеграционное API). Ниже — как поставить слой **бизнес-аналитики и дашбордов поверх SQL**. Это **не** Grafana (метрики/алерты), **не** озеро эталона, **не** Kafka и **не** «кнопка отчётность включена».

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **Superset** | Веб-приложение: люди рисуют графики и дашборды **SQL-запросами** к базам, которые вы подключили. Само **не хранит** терабайты озера. Оно *спрашивает* склад и показывает таблицу/картинку. |
| **Metadata database (метаданные)** | Отдельная БД Superset: пользователи, дашборды, датасеты, сохранённые запросы, расписания алертов. Это **не** озеро клиентов. |
| **Data source / Database (в UI)** | Подключение к *чужой* СУБД, откуда берутся факты: PostgreSQL озера, аналитический склад, иногда OpenSearch. Пароль этого подключения лежит в метаданных, зашифрованный `SECRET_KEY`. |
| **Dataset** | «Таблица или SQL-вид, из которого рисуют чарт». Права часто вешаются на датасет, не на отдельный график. |
| **Chart / Slice** | Один график = один запрос + визуализация. |
| **Dashboard** | Экран из чартов. Пока не **published** — его не видят остальные (правило документации Security). |
| **Explore** | Конструктор чарта без SQL Lab. |
| **SQL Lab** | Встроенный SQL-редактор. Это консоль к вашей базе **от имени учётки, которой подключили datasource**. Не «безопасно, потому что внутри UI». |
| **Row Level Security (RLS)** | Фильтры Superset вида «этому субъекту добавляй `WHERE client_id IN (...)`». Это **не** RLS PostgreSQL и **не** firewall. Обойти можно SQL Lab / другой datasource / прямой доступ к складу. |
| **Subject (субъект)** | Кому выдают доступ: пользователь, группа или роль. В 6.x пикеры по умолчанию предлагают **Users и Groups**; роли — скорее совместимость (в т.ч. старый RLS). |
| **Admin / Alpha / Gamma / sql_lab / Public** | Встроенные роли. `superset init` **перезаписывает** их права. Admin = полный контроль, включая CSS/Jinja; вендор прямо: векторы «нужен Admin» **не** считаются CVE. |
| **Gamma** | Смотрит то, к чему выдан доступ на датасеты (плюс может собирать чарты/дашборды). Не админ источников. |
| **Public** | Роль **анонимов**. Сама по себе не открывает интернет; открывает, если вы настроили `AUTH_ROLE_PUBLIC` / `PUBLIC_ROLE_LIKE` и выдали датасеты. |
| **`SECRET_KEY` / `SUPERSET_SECRET_KEY`** | Ключ Flask: подпись cookie сессии **и** шифрование секретов в метаданных. С 2.1.0 дефолт на не-debug **запрещает старт**. Смена ключа без `re-encrypt-secrets` = не читаются пароли источников. |
| **`CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET`** | Значение `CHANGE_ME_SECRET_KEY` в `superset/constants.py` тега **6.1.0**. Именно его процесс отказывается считать секретом. |
| **`thisISaSECRET_1234`** | Дефолт, который документация Kubernetes называет ключом helm-инсталляции, если свой не задали. Это **другая** известная строка, не `CHANGE_ME_...`. |
| **`superset_config.py`** | Ваш файл-оверрайд. Кладётся в `PYTHONPATH` или путь `SUPERSET_CONFIG_PATH`. Копировать весь `config.py` не нужно — только то, что меняете. |
| **Gunicorn** | WSGI-сервер, которым обычно крутят Flask-приложение. Вендор рекомендует **async / gevent**. `superset run` / `flask run` — **не** прод. |
| **Celery worker** | Отдельный процесс: длинные SQL Lab («Run Async»), прогрев кэша, скриншоты алертов. Без воркеров асинхронные задачи **не выполняются**. |
| **Celery beat** | Будильник расписаний (Alerts & Reports). Процесс **должен быть один**. Два beat = двойные письма. |
| **Broker (Celery)** | Очередь задач. Дефолт в `config.py` 6.1.0 — **SQLite**. В проде — Redis (или иной брокер из документации Celery). |
| **Results backend** | Куда кладут результат длинного SQL Lab. Дефолт — снова SQLite / `None` для `RESULTS_BACKEND`. Для нескольких подов нужен **общий** Redis/S3, не диск пода. |
| **CACHE_CONFIG / DATA_CACHE_CONFIG** | Кэш объектов и результатов чартов. Дефолт **`NullCache`**: каждый refresh бьёт в склад. |
| **FILTER_STATE_CACHE / EXPLORE_FORM_DATA_CACHE** | Состояние фильтров дашборда / формы Explore. Дефолт — таблица в **метаданных** (`SupersetMetastoreCache`). HA web-реплик это переживает. |
| **NullCache** | «Кэша нет». Дашборд с автообновлением 10 секунд при NullCache = постоянный шторм SQL. |
| **Talisman / CSP** | Заголовки Content-Security-Policy. `TALISMAN_ENABLED` по умолчанию **True**. Выключать «чтобы заработало Mapbox/iframe» — снимать защиту XSS. |
| **CSRF (`WTF_CSRF_ENABLED`)** | Защита форм. Дефолт **True**. Срок токена в 6.1.0 — **1 неделя** (`WTF_CSRF_TIME_LIMIT`). |
| **`ENABLE_PROXY_FIX`** | Сказать Flask: доверяй `X-Forwarded-*` от балансировщика. Дефолт **False**. За HTTPS-прокси без этого ломаются cookie/`redirect_uri` OAuth. |
| **`/health`** | Healthcheck: HTTP 200 и тело `OK`, если веб-процесс жив. Это **не** проверка Postgres/Redis/склада. |
| **Provisioning / import** | Дашборды и datasources можно залить YAML/ZIP (GitOps), а не только кликами. Клики без бэкапа метаданных = единственная копия правды в Postgres. |
| **Alerts & Reports** | Расписание: выполни запрос / сними PNG дашборда, пошли email/Slack. Нужны: feature flag `ALERT_REPORTS`, **beat**, worker, **webdriver** (Chrome/Firefox), SMTP. |
| **Webdriver** | Headless-браузер в worker-поде. Рисует PNG для письма. Отдельный Chromium, не часть «лёгкого» образа. |
| **Embedded / Guest token** | Встроить дашборд в чужой сайт по JWT. Feature flag `EMBEDDED_SUPERSET` дефолт **False**. Дефолтные JWT-секреты в `config.py` — тестовые строки. |
| **Row limit (`ROW_LIMIT` / `SQL_MAX_ROW`)** | Сколько строк чарт / SQL Lab могут вытащить. В 6.1.0: `ROW_LIMIT = 50000`, `SQL_MAX_ROW = 100000`. Это защита UI, не склада. |
| **Jinja (`ENABLE_TEMPLATE_PROCESSING`)** | Подставлять шаблоны в SQL датасета. Дефолт **False**. Включение без жёсткой политики = пользователь влияет на SQL. |
| **Scarf** | Телеметрия проекта: gateway на pull образа и пиксель в UI. Для закрытого контура оба канала **выключают**. |
| **Stretch** | Одни и те же web/worker-поды в нескольких ЦОДах, но **одна** metadata DB и **один** Redis. |

---

## Основные элементы системы и зависимости

### Что входит в «поставить Superset» (это несколько процессов одного ПО)

| Роль | Зачем | Как масштабируется |
|---|---|---|
| **Web (Gunicorn / `supersetNode`)** | UI, API, синхронные запросы чартов | Горизонтально: реплики Deployment. Вертикально: `-w` воркеров Gunicorn и RAM |
| **Celery worker (`supersetWorker`)** | Async SQL Lab, кэш-warmup, скриншоты отчётов | Горизонтально: реплики. Каждая открывает свои коннекты к складу |
| **Celery beat (`supersetCeleryBeat`)** | Тикает cron Alerts & Reports | **Ровно 1** реплика. Это не HA-кворум, это «не запустить два будильника» |
| **Metadata DB** | Состояние приложения | Это PostgreSQL/MySQL **кластер**, не «ещё один контейнер в чарте» |
| **Redis (типичный прод)** | Кэш, broker Celery, results backend, опционально server-side session / rate-limit storage | Отдельное ПО. Чартовый Redis в values — удобство стенда |
| **Init Job** | `superset db upgrade`, `superset init`, создание admin | Разово на install/upgrade. Не часть runtime HA |

Плюс *рядом*, но это уже другое ПО: **Ingress/HAProxy**, **IdP (LDAP/OAuth)**, **webdriver**, SMTP, драйверы складов, Grafana (другой класс дашбордов).

Это **не** кластер с кворумом вроде Kafka/etcd. Несколько web-реплик — одинаковые stateless процессы с **общей** БД и **общим** Redis.

### Чего в Apache Superset нет (частая путаница)

| Нужно системе | Это не Superset | Зачем помнить |
|---|---|---|
| Хранить терабайты клиентских данных | Озеро / склад / PostgreSQL SoT | «Поставили Superset» ≠ «аналитика выдержала объём». Ёмкость — у источника. |
| Метрики Kafka/K8s, алерт «под упал» | Prometheus + Grafana | Другой документ. Superset не скрейпит `/metrics` брокеров. |
| Шина событий / гарантии Kafka | Apache Kafka | Дашборд не доставляет событие в Camunda. |
| Исполнение BPMN | Camunda | Можно *нарисовать* статистику процессов, если она лежит в SQL. |
| Интеграции с госслужбами | Ваше интеграционное API | Superset не ходит в ведомства. Подключить ведомственный SQL «напрямую» = обойти интеграционный контур. |
| SIEM / FIM / WAF | Wazuh, Falco, SafeLine | BI-UI — ещё один HTTP-вход и ещё один SQL-клиент. |
| Отказоустойчивость **самой** metadata DB | Patroni / CloudNativePG / управляемый Postgres | Падение этой БД = нет логина, нет дашбордов, даже если поды зелёные. |
| Кворум «2 из 3 web-подов» | Нет | Жив ≥1 web **и** жива metadata DB — UI есть. |
| Database firewall | `DISALLOWED_SQL_FUNCTIONS` | Документация Security: это **дополнение**, не замена прав на складе. Обходы возможны. |
| ГОСТ TLS / СКЗИ | Не заявлено | Штатный TLS прокси — не криптография по требованиям КИИ, если ИБ их выдвинет. |
| Порог RTT между ЦОДами | Нет | Metadata DB и Redis чувствуют задержку как любой синхронный клиент. Миллисекунд в доке нет. |
| Официальный SLA 24/7 и таблица «Small/Medium/Large ядер» | Нет | Есть пример Gunicorn «известно, что работает» и рычаги (реплики, кэш, async). Сметы «N ядер на терабайт озера» у проекта **нет**. |
| Официальная поддержка Windows | Нет | Документация Docker Compose: официальной поддержки Windows нет. |

### Официальные порты (менять можно, но это контракт сети)

| Порт | Назначение |
|---|---|
| **8088/TCP** | UI и API web (дефолт сервиса Helm `service.port`). Пример Gunicorn в доке крутит `-b 0.0.0.0:6666` — это **иллюстрация**, не дефолт образа |
| **6379/TCP** | Не Superset, а Redis (кэш / Celery) |
| **5432 / 3306** | Не Superset, а PostgreSQL / MySQL метаданных |
| **5555/TCP** | Celery Flower (Helm, дефолт **выкл.**). Это UI очереди; в интернет не выставлять |
| **8080/TCP** | Helm `supersetWebsockets` (дефолт **выкл.**). Нужен только при `GLOBAL_ASYNC_QUERIES` в режиме `ws`. Официального образа websocket **нет**; чарт ставит community-образ |

Балансировщик перед 8088 официально входит в схему «за LB»: health — `/health`. У вас уже описан HAProxy — это этот слой, не часть образа Superset.

### Зависимости окружения (обязательны)

- **ОС.** Linux как основной контур (Docker/K8s). Windows для стенда/прода проектом **не** обещан.
- **Metadata DB для прода:** PostgreSQL **10.x–17.x** или MySQL **5.7 / 8.x** (таблица Configuring Superset). Драйверы: `psycopg2` / `mysqlclient`. SQLite в том же документе для прода **настоятельно не рекомендован** (безопасность, масштаб, целостность).
- **Рекомендация вендора по Gunicorn:** async (`-k gevent`), пример: **10** воркеров, `--worker-connections 1000`, `--timeout 120`. BigQuery Python SDK **несовместим с gevent** — тогда другой worker type.
- **Драйвер склада.** Официальный образ **не** содержит все диалекты. Для каждого datasource нужен DB-API + SQLAlchemy dialect. С 6.1.0 в Docker ставят пакеты через **`uv pip install`** (UPDATING.md, PR 31260), не «просто pip» в старых bootstrap-скриптах.
- **Redis** — не обязателен, чтобы процесс *стартанул*, но без него: NullCache, Celery на SQLite, нет общего results backend. Для нескольких реплик web/worker это ломает async и кэш.
- **Kubernetes:** чарт 0.21.1. Зависимости чарта: Bitnami PostgreSQL **16.7.27** и Redis **17.9.4** (`oci://registry-1.docker.io/bitnamicharts`), `condition: postgresql.enabled` / `redis.enabled`. Artifact Hub для 0.21.1 показывает образы `bitnamilegacy/postgresql:14.17.0-...` и `bitnamilegacy/redis:7.0.10-...`. Это **стендовые** сабчарты, не ваш прод-Postgres.
- **PKI / Ingress / HAProxy** для TLS. Сам процесс обычно за прокси; `ENABLE_PROXY_FIX = True`.
- Исходящая сеть до Scarf / Docker Hub — **по умолчанию** при pull через gateway. В закрытом контуре: образ `apache/superset`, `SCARF_ANALYTICS=false`.

### Как Superset стыкуется с вашей архитектурой

```
Озеро эталонных данных / склад (SoT)     ← терабайты живут ЗДЕСЬ
        ▲  SQL (лучше read-only учётка, лучше реплика)
        │
   Apache Superset × N web + workers
        ▲  метаданные, не факты
        │
   PostgreSQL (dashboards, users, encrypted passwords)
   Redis     (cache, celery)
        ▲
        │  HTTPS :443
   HAProxy / Ingress  (люди, SSO)

Kafka / Camunda / интеграционное API  ──не входят в Superset──
   (можно показать, только если их состояние уже лежит в SQL/складе)
```

Superset **не** участник event-sourcing и **не** ходит в госслужбы. Он может *показать* нормализованные данные озера и операционные витрины (лаг интеграций, инциденты Camunda), **если** эти витрины построены отдельно.

Пересечение с уже описанным:

- **Grafana** — дашборды *инфраструктуры*. Superset — дашборды *предметной области*. Два UI, два класса данных. Не заменять одно другим.
- **OpenSearch** — можно подключить как datasource для поиска/логов, это не замена озера.
- **HAProxy** — вход на 8088.
- **Vault** — `SECRET_KEY`, пароль metadata DB, пароли складов, OAuth client secret. Шифрование паролей *внутри* метаданных — не замена Vault.
- **Kafka** — не datasource «из коробки» как очередь; в BI обычно смотрят **проекцию** в складе.
- **Kubernetes** — оркестрация. PVC SQLite на web при `replicas > 1` — ловушка.

---

## Краткие вводные

### Зачем вам Superset в этой архитектуре

У вас озеро эталона (в том числе клиентские данные), 30+ интеграций, процессы Camunda и ожидание, что «руководство увидит картину». Типичные вопросы, на которые отвечает Superset **если** снизу жив SQL-источник:

1. «Сколько заявок в статусе X, какая воронка процесса» — витрина, не Operate Camunda.
2. «Какой профиль клиентов / срезы справочников» — чтение озера, не SoT.
3. «Сколько вызовов интеграционного API упало за сутки» — если это сложено в склад/БД, не если это только лог в stdout.
4. Self-service для аналитиков (Explore / SQL Lab) — это и ценность, и главный риск утечки ПДн.

Без витрин в SQL/складе Superset — пустая рамка. Он **не создаёт** озеро.

### Как устроена отказоустойчивость (идея, не магия)

Это **не** Raft. Независимые слои:

**1) Процессы web**

| Что падает | Что происходит |
|---|---|
| Один web-под при N≥2 и живых Postgres+Redis | Балансировщик уводит трафик (`/health`). UI жив. Сессия в cookie, подписанная общим `SECRET_KEY` — sticky **не** требуется для обычного логина (`SESSION_SERVER_SIDE = False` по умолчанию). |
| Все web-поды | Нет UI. Worker/beat могут ещё крутить отчёты, люди — нет. |
| `/health` зелёный, Postgres мёртв | LB счастлив, логин/дашборды нет. Health **не** проверяет метаданные. |

**2) Metadata DB (ядро состояния)**

| Что падает | Что происходит |
|---|---|
| SQLite на диске одного пода | Рестарт/другая реплика = другая «правда» или пусто. Прод-документация SQLite отговаривает. |
| Postgres primary без failover | Все web/worker не пишут. Дашборды и пользователи недоступны. |
| Потеря кворума Postgres, растянутого на 3 ЦОДа вслепую | Это история **PostgreSQL**, не Superset. |

**3) Redis / Celery**

| Что падает | Что происходит |
|---|---|
| Redis при NullCache и без async | UI чартов может жить (каждый раз SQL в склад). Async SQL Lab и общий кэш — нет. |
| Redis = broker, workers живы | Задачи не берутся. Отчёты молчат. |
| Два Celery beat | Дубли Alerts & Reports (два крона). |
| Results backend = файлы пода | Пользователь на web-A не заберёт результат, который worker записал на диск web-B / worker-B. |

**4) Склад (озеро)**

Падение склада / read-only реплики = пустые чарты и failed SQL Lab. Superset тут ни при чём. Для «система 24/7» BI считается **по самому слабому** из: web, metadata DB, Redis, склад, SMTP (если отчёты — часть дежурства).

### Как устроено масштабирование

Три оси, которые путают:

1. **Больше людей смотрят дашборды** → реплики web + воркеры Gunicorn + **кэш** (`DATA_CACHE_CONFIG` не NullCache) + не ставить автообновление дашборда в **10 секунд** на тяжёлые чарты (в UI такой интервал есть).
2. **Больше тяжёлых SQL / SQL Lab async** → реплики **Celery worker** + Redis results backend + таймауты (`SQLLAB_TIMEOUT` дефолт **30s** для синка; Gunicorn `--timeout 120` в примере; `SUPERSET_WEBSERVER_TIMEOUT` дефолт **60s**). Упереться можно в **склад**, не в «ещё один под Superset».
3. **Терабайты** → масштаб **озера/склада** (шарды, реплики, колоночное сжатие). Metadata DB остаётся маленькой (определения дашбордов, юзеры). `SQL_MAX_ROW` / `ROW_LIMIT` режут выборку в UI, они не индексируют озеро.

Нагрузка на склад с N web-подов ≈ N × (gunicorn-процессы) × (одновременные чарты) × (1 − cache hit). При NullCache cache hit = 0.

Пул соединений: в `config.py` 6.1.0 `SQLALCHEMY_ENGINE_OPTIONS = {}` (есть комментарий: для обычной работы рекомендуют isolation **READ COMMITTED**). Явного `pool_size` проект **не** нормирует. Считать сами: процессы web **и** worker, каждый со своим пулом к метаданным **и** к каждому складу. Иначе «добавили реплик для HA» съест `max_connections` Postgres озера.

Gunicorn `-w 10` из доки — **пример**, не формула под ваши терабайты. Валидировать нагрузкой с **вашими** дашбордами.

`supersetNode.autoscaling.maxReplicas: 100` в чарте — потолок HPA из values, не обещание, что 100 реплик разумны.

### Безопасность самой Superset

Компрометация Admin = компрометация **конфигурации UI**, Jinja/CSS и **всех паролей datasources** в метаданных (они зашифрованы `SECRET_KEY`). Gamma с доступом к клиентскому датасету видит строки озера в браузере — это штатно, это и есть BI.

Известные дефолты **6.1.0** (`superset/config.py` / `constants.py` тега `6.1.0` и Helm-документация):

| Параметр | Значение из коробки | Зачем это опасно |
|---|---|---|
| `SECRET_KEY` | `CHANGE_ME_TO_A_COMPLEX_RANDOM_SECRET` | Не-debug процесс **отказывается стартовать**. Если обойти debug'ом — один ключ на все наивные инсталлы |
| Helm fallback (дока k8s) | `thisISaSECRET_1234` | Ключ известен; дамп метаданных расшифровывается |
| Первый admin (Docker Compose / Helm init) | `admin` / `admin` (`init.adminUser.password` в чарте) | Первый вход с мира = чужой Admin |
| Metadata URI | SQLite файл | Нет HA, слабая модель доступа |
| Celery broker / result | SQLite файлы | Не для нескольких подов |
| `CACHE_CONFIG` / `DATA_CACHE_CONFIG` | `NullCache` | Нет кэша; плюс каждый смотрит свежие ПДн — и бьёт склад |
| `SESSION_COOKIE_SECURE` | `False` | Cookie сессии может уехать по HTTP |
| `ENABLE_PROXY_FIX` | `False` | За TLS-прокси OAuth/редиректы легко сломать; без фикса ещё и схема HTTP в ссылках |
| `RATELIMIT_ENABLED` | только если `SUPERSET_ENV=production` | На стенде с продом-профилем без этой env лимиты молчат |
| `AUTH_RATE_LIMIT` | `5 per second` (когда лимиты включены) | Имеет смысл; storage лимитов без Redis — по процессу |
| `PUBLIC_ROLE_LIKE` | `None` | Пока не включили анонимов — ок. Включить «как Gamma» + датасеты клиентов = интернет читает озеро |
| `ENABLE_TEMPLATE_PROCESSING` | `False` | Не включать «для удобства фильтров» без ревью SQL-инъекций |
| `SSH_TUNNELING` | `False` | Туннель из пода в чужую сеть — отдельный риск |
| `EMBEDDED_SUPERSET` | `False` | Ок. Дефолты `GUEST_TOKEN_JWT_SECRET` / `GLOBAL_ASYNC_QUERIES_JWT_SECRET` — строки `test-...-change-me` |
| `PREVENT_UNSAFE_DB_CONNECTIONS` | `True` | Не выключать: часть URI (sqlite, local files) — вектор |
| `DATASET_IMPORT_ALLOWED_DATA_URLS` | `[r".*"]` | Импорт датасета может тянуть URL **откуда угодно**, пока не сузите список |
| `FAB_ADD_SECURITY_API` | **`True` в config.py 6.1.0** | Страница Security говорит «включите, по умолчанию выкл» — для **этого тега** верьте файлу конфигурации: Security REST **включён**. Сузьте сетью/RBAC |
| `runAsUser` Helm | **`0` (root)** | README чарта: в проде так не рекомендуется; для bootstrap часто оставляют root «чтобы apt/pip» |
| `postgresql.postgresqlPassword` в доке k8s | `superset` | Пароль сабчарта из примера |
| Image pull | Scarf Gateway | Счётчик инсталляций; в закрытом контуре — прямой Docker Hub |
| Flower | выкл | Если включить без auth — карта очередей наружу |
| `ALERT_REPORTS` | `False` | Включение без webdriver/SMTP = мёртвая кнопка; с webdriver = Chromium в кластере |

«Выставили LoadBalancer в интернет с `admin`/`admin`» — это новый attack surface на **озеро**, не «удобные графики».

SQL Lab: учётка datasource с `INSERT`/`COPY`/`FILE` = аналитик пишет в SoT. Документация: least privilege, лучше read-only, лучше отдельные роли БД.

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Берём self-hosted Apache Superset OSS 6.1.0**, не Preset Cloud и не форк вендора. Облако в схеме с тремя своими ЦОДами не предполагалось.
2. **Прод крутится в Kubernetes**, инсталлятор — Helm **`superset/superset` 0.21.1**, образ **`apache/superset:6.1.0`** (pin по digest, не `latest`). Operator 0.2.0 — допустимая альтернатива с теми же правилами HA (общая metadata DB + Redis), но страница k8s проекта его не описывает как единственный путь.
3. **Metadata DB — PostgreSQL** (из поддерживаемых 10–17; практически — актуальный 16/17 вашего платформенного Postgres), **отдельная** база `superset`, не та же, что у Grafana/Camunda/озера. HA этой БД — отдельное ПО.
4. **Redis — отдельный кластер** (не single-master из bitnami-сабчарта). Разные logical DB: Celery broker и cache/results, как делает Helm (`redis_celery_db` 0, `redis_cache_db` 1).
5. **Три ЦОДа = три зоны отказа.** Web/worker можно размазать. **Синхронный Postgres метаданных и Redis** — нет, пока RTT неизвестен. Честный прод: не растягивать синхронную запись метаданных на три площадки вслепую.
6. **Цель отказа: пережить 1 ЦОД** для UI, **если** в живых зонах остались web-поды **и** доступны metadata DB + Redis. Пережить 2 из 3, если Postgres жил только в мёртвых двух — нельзя.
7. **Нагрузка неизвестна** — поэтому **нет** цифры «N реплик и M millicores». Есть рычаги (реплики web/worker, кэш, не умножать коннекты, не refresh 10s).
8. **Озеро/склад как SQL-источник будет.** Этот документ его не проектирует. Без витрины «готовность Superset» бессмысленна.
9. **SSO/IdP будет** (LDAP или OAuth через Flask-AppBuilder). Локальный `admin` в проде — break-glass. `AUTH_USER_REGISTRATION_ROLE = "Admin"` из примера Helm OAuth **не** копировать в прод.
10. **Закрытый контур.** Scarf выкл, образы из внутреннего registry, Mapbox/Slack SaaS — только если ИБ разрешит.
11. **Тестовый стенд изолирован.** На нём допустимы SQLite/сабчарты, `admin`/`admin`, examples. В прод не копируем.
12. **Camunda, Kafka, интеграционный API** — не datasources «на проде под Admin». Их операционные БД не открывать аналитикам. Смотреть витрины в озере.
13. **Alerts & Reports и Embedded в первом проде не обязательны.** Иначе лишний Chromium, SMTP наружу и guest JWT.
14. **SQL Lab в проде — отдельное решение ИБ**, не «включено, потому что кнопка есть». По умолчанию роль `sql_lab` выдаём узко.
15. **Формального SLO на «дашборд открылся за 2 секунды» нет.** Latency = склад + кэш + сеть, не обещание Superset.

---

## Критически важные условия, которых нет в исходном контексте

Их нужно закрыть **до** «поставили Helm и открыли :8088».

| Пробел | Почему это ломает решение |
|---|---|
| **Что такое «озеро» технически** (PostgreSQL? ClickHouse? Iceberg+Trino? Mongo?) | От этого зависят драйвер в образе, gevent vs sync, пулы, RLS. Без типа склада нет схемы. |
| **RTT и фильтрация между ЦОДами на 5432 и 6379** | Web в зоне A и Postgres в зоне B с большим RTT = медленный логин и тай-ауты чартов. Redis как broker ещё чувствительнее к потерям. |
| **Один K8s на 3 ЦОДа или три кластера** | Один Deployment + одна БД **или** три независимых Superset (три правды дашбордов — плохо для руководства). |
| **Где живёт Postgres метаданных** | Поды переживают зону. Primary в одном ЦОДе — смерть зоны = смерть BI везде, пока нет failover. |
| **Число зрителей, дашбордов, SQL Lab, refresh** | Без этого нельзя выбрать число реплик и размер склада. «Готовы к высокой нагрузке» без замера — не смета. |
| **152-ФЗ / КИИ / состав ПДн в чартах** | Gamma с датасетом клиентов = законный читатель ФИО/ИНН в браузере. Нужны RLS **и** права на складе **и** запрет выгрузки CSV тем, кому нельзя. |
| **Кто имеет SQL Lab и с какой учёткой БД** | Это главный внутренний вектор «вынесли SoT». |
| **Куда смотрит UI** (только внутренняя сеть? SSO?) | Публичный 443 с `admin`/`admin` — инцидент на клиентских данных, не на CPU. |
| **Уже есть Grafana** | Два «дашборд-инструмента» без разделения ролей = двойные права и путаница, кто алертит. |
| **Нужны ли письма с PNG дашбордов** | Да → webdriver, beat=1, исходящий SMTP, `WEBDRIVER_BASEURL` за Ingress. Нет → не включать `ALERT_REPORTS`. |
| **Юридический контур образов** | Scarf, Docker Hub, Bitnami/bitnamilegacy сабчарты — отдельные решения ИБ/закупок. |

---

## Краткое описание ключевых этапов и элементов развертывания

Общий каркас (и тест, и прод):

1. Зафиксировать **Superset 6.1.0** и способ: Docker Compose **или** Helm 0.21.1 **или** Operator 0.2.0. Не смешивать 5.x UI с 6.1 БД «на глаз» без UPDATING.md.
2. Сначала **metadata DB**, потом Redis (для всего, что не одноразовый ноутбук), потом web, потом worker. Init: `db upgrade` + `init` + admin.
3. Выпустить **свои** `SECRET_KEY` (`openssl rand -base64 42` — рекомендация вендора), пароль admin, пароль БД, TLS **до** вывода в прод. Смена ключа потом — `PREVIOUS_SECRET_KEY` + `superset re-encrypt-secrets`.
4. Подключить **один** datasource (не Kafka) read-only учёткой. Чарт «Hello, lake». Проверка: панель обновилась.
5. Включить `/health` на LB. Логи web/worker — в тот же контур, что и остальной аудит.
6. Только потом — SSO, роли Gamma vs sql_lab, RLS, запрет Public.

Дальше — два режима.

---

### 1 инстанс: тестовый стенд, 1 ЦОД, без нагрузки

**Цель стенда:** открыть UI, подключить *ваш* тестовый SQL, понять Explore vs SQL Lab vs Grafana. **Не** цель: доказать, что прод переживёт падение ЦОДа и 200 аналитиков по клиентам.

Два официальных минимальных пути (выберите один):

**Путь A — Docker Compose, тег релиза** (быстрее понять продукт). Документация: Compose **не** поддерживается как HA/прод; для одиночного хоста вендор скорее рекомендует minikube+Helm. Compose годится как «пощупать»:

```text
export TAG=6.1.0
git fetch --depth=1 origin tag $TAG && git checkout $TAG
docker compose -f docker-compose-image-tag.yml up
```

Вход: http://localhost:8088 , `admin` / `admin`. Метаданные — Postgres **в compose volume** (дока прямо: volume **не бэкапится**). Удалили volume — удалили дашборды.

Для `docker-compose-image-tag.yml` документация 2026 года всё ещё приводит пример `TAG=5.0.0` и оговорку про `-dev` суффикс из‑за `psycopg2-binary`. Для 6.1.0 сверяйте tag/digest в Docker Hub **перед** тем, как копировать пример 5.x буквально.

**Путь B — Helm в namespace `bi`** (ближе к вашему оркестратору):

- репозиторий `https://apache.github.io/superset`, чарт **0.21.1**;
- образ `apache/superset:6.1.0`;
- `extraSecretEnv.SUPERSET_SECRET_KEY` — свой, даже на тесте;
- `supersetNode.replicas.replicaCount: 1`;
- встроенные postgresql+redis сабчарты **допустимы на тесте**;
- Ingress/port-forward только во внутренней сети;
- `init.loadExamples: true` — по желанию, жрёт CPU на старте.

Режим «3 реплики web + SQLite + PVC RWO» для знакомства **не нужен**.

#### Что сознательно упрощаем

| Параметр | На тесте | Почему можно |
|---|---|---|
| Metadata DB | SQLite или bitnami Postgres чарта | Нет требования пережить выкат пода |
| Redis | сабчарт single или даже без (light compose) | Нет HA-цели |
| Реплики web/worker | 1 | Нет HA-цели |
| Celery beat / Alerts | выкл | Не цель |
| PKI | self-signed / port-forward | Иначе команда утонет в сертификатах раньше, чем в SQL |
| Пароль admin | сменить с `admin`, но не городить SSO | На тесте важнее контур данных |
| Scarf | можно оставить, **если** стенд без ПДн и с интернетом | В чек-лист прода не переносить |
| SQL Lab | можно потрогать на **синтетике** | Боевое озеро не подключать |
| CPU/memory | скромные requests | Не цель — ёмкость склада |

#### Чего на тесте **не** стоит упрощать

- Тег **6.1.0**, не `latest`.
- Свой `SECRET_KEY`, не `CHANGE_ME_...` и не `thisISaSECRET_1234`.
- Хотя бы один чарт, который показывает **не** пусто. Проверка: Explore тот же запрос живой.
- Понимание: дашборд только в UI **умрёт** вместе с volume. Имеет смысл сразу Export в git.
- Не публиковать 8088 в интернет «на минутку».
- Не подключать боевое озеро учёткой superuser «потому что так быстрее».
- Не ставить `AUTH_USER_REGISTRATION_ROLE = "Admin"` из примера OAuth.

#### Сильные стороны такой схемы

- Compose поднимается за минуты; Helm — за час.
- Совпадает с официальным quickstart / k8s getting-started.
- Дешево показывает, какие витрины у вас **уже есть**, а каких нет (часто нет нормализованного ответа интеграционного API в SQL).

#### Слабые стороны (обязательно понимать)

- Нет модели отказа ЦОДа, нет HA Postgres/Redis.
- Успешный график на ноутбуке **не** доказывает, что 5 web-реплик не убьют `max_connections` озера.
- Привычка `admin`/`admin`, root (`runAsUser: 0`) и LoadBalancer из примера уедет в прод, если не запретить явно.
- Examples-дашборды не имеют отношения к вашей доменной модели.

Практическая рекомендация: препрод = маленький **прод-профиль** (внешний Postgres, Redis, 2 web, ≥1 worker, свой секрет, SSO-заглушка, Scarf выкл, read-only datasource), даже без боевого трафика.

---

### Прод: 3 ЦОДа, нагрузка

Цифр «ядер Superset под терабайты озера» **нет** — нагрузки нет, и эти терабайты Superset не хранит. Ниже правила, без которых экземпляр не считается готовым.

#### Шаг 0. Макроархитектура (сделать до установки)

Superset плохо переживает **три независимых UI** (три набора дашбордов и прав) и плохо переживает **слепой stretch Postgres**.

**Вариант A — один Kubernetes на 3 ЦОДа (зоны), один Superset.**

- Deployment web **≥ 2**, worker **≥ 2**, beat **= 1**.
- topologySpread / anti-affinity: **не все web в одном ЦОДе**. Beat — без разницы зона, но он SPOF расписаний: падение зоны beat = не уходят отчёты, UI жив.
- Один Postgres-кластер метаданных (синхронная реплика — по замеренному RTT; третья площадка чаще **асинхронный** DR).
- Один Redis (или Redis Cluster/Sentinel — это уже документ по Redis; чарт умеет задать `cache.sentinel`).
- Перед 8088 — **HTTPS LB** (HAProxy/Ingress). Sticky для дефолтных cookie-сессий **не** обязателен. Если включите `SESSION_SERVER_SIDE` + Redis — тоже общий Redis, не sticky.
- `ENABLE_PROXY_FIX = True`, канонический публичный URL для OAuth `redirect_uri` (`https://<host>/oauth-authorized/<provider>`).
- NetworkPolicy: 8088 только от LB; 5432 только от web/worker/beat/init; 6379 только от них же. Склад — только от web/worker, **не** от Ingress.

**Вариант B — три изолированных Kubernetes.**

Не копировать полный Superset «в каждый ЦОД как Grafana B2». Получите три правды дашбордов.

Рабочие подварианты:

- **B1.** Мозг (web/worker/beat + Postgres + Redis) в **одном** кластере/ЦОДе; люди и SSO ходят туда. Склад в других ЦОДах доступен как SQL (учтите RTT и `SQLLAB_TIMEOUT`). Один UI.
- **B2.** Три полных Superset — только если готовы тройной GitOps дашбордов и тройные права. Обычно хуже.

**Пока топология K8s не зафиксирована, нельзя честно выбрать A или B1.** Дальше — общее.

##### Сильные / слабые стороны (A, один кластер на 3 зоны)

| Сильное | Слабое |
|---|---|
| Один набор дашбордов и RLS | Postgres и Redis чувствуют RTT |
| Реплики UI переживают смерть одной зоны | Смерть зоны с Postgres primary без failover = смерть BI везде |
| Официальная модель: N web за LB + shared DB | Ошибка в роли/датасете сразу везде |
| Worker в других зонах продолжают async | Каждый лишний под умножает коннекты к озеру |

##### Сильные / слабые стороны (B1, мозг в одном месте)

| Сильное | Слабое |
|---|---|
| Postgres/Redis не обязаны ходить по трём ЦОДам | Падение ЦОДа с Superset = нет BI для всех площадок |
| Проще кворум БД | Нужен явный DR: как поднять Postgres+Redis+Superset на другой площадке **с бэкапом метаданных** |
| Меньше сюрпризов Celery | Путь людей/SSO и путь SQL к озеру пересекают ЦОДы |

##### Слабое место любого stretch без замера RTT

Документация Superset **не задаёт** «можно при 5 мс, нельзя при 40». Утверждение «3 ЦОДа в одном городе, значит Postgres синхронно на три» — **непроверенное**. Измерить: RTT и потери на **5432** и **6379** до закупки схемы.

#### Web-поды (ядро отказоустойчивости *просмотра*)

- `supersetNode.replicas.replicaCount ≥ 2` (чарт дефолт **1** — это не прод).
- PDB: не убивать все web сразу (`minAvailable: 1` как минимум). В values чарта PDB дефолт **выключен**.
- Rolling update; `terminationGracePeriodSeconds` > таймаута Gunicorn (в values есть явная подсказка).
- **Не** SQLite. Состояние — в Postgres.
- Resources: пустые `resources: {}` чарта в прод не оставлять. Старт консервативный и смотреть latency чартов + CPU throttling + `max_connections` склада.
- Готовый образ **со своими драйверами**, не `bootstrapScript` с `apt`/`uv pip` на каждом старте от root. Документация k8s: для прода «build own image with this step done in CI».
- `runAsUser`: не `0`. README чарта предлагает например **1000**, если bootstrap больше не нужен.
- Пробы: liveness/readiness чарта бьют в `/health` (initialDelay 15s). Это не readiness «Postgres ответил».
- `SUPERSET_ENV=production` — чтобы включился `RATELIMIT_ENABLED`. Для лимитов на несколько подов задайте `RATELIMIT_STORAGE_URI` на Redis, иначе счётчики **по процессу**.

**Слабое место:** `helm install` из TL;DR с одной репликой, bitnami Postgres, root и без `SECRET_KEY` выглядит как BI, ведёт себя как один диск плюс известный ключ.

#### Worker и beat (ядро *длинных запросов и писем*)

- Worker `replicaCount ≥ 2` (дефолт чарта **1**).
- Beat: `supersetCeleryBeat.enabled = true` **только** если нужны Alerts & Reports; реплик beat **не** скейлить.
- Broker/result **не** SQLite. Redis, как в примере документации Async queries.
- `RESULTS_BACKEND` — общий RedisCache (дока приводит `flask_caching.backends.rediscache.RedisCache`), не `FileSystemCache` из docker-dev конфига.
- `task_acks_late` в дефолтном `CeleryConfig` проекта = **False**; пример async-дока показывает `True`. Не копировать вслепую: это про повтор задач при убийстве worker, не про «магия быстрее».
- Healthcheck worker в чарте по умолчанию — `celery inspect ping` с `initialDelaySeconds: 120`. File-based health — opt-in.
- Prune: история SQL Lab **не** чистится, пока не включите beat-задачи `prune_query` / `prune_logs` (дока Configuring). Иначе metadata DB распухнет запросами, не дашбордами.

Падение всех worker: UI жив, async и отчёты нет. Падение beat: UI жив, расписание нет.

#### Metadata DB

- Отдельная роль `superset`, отдельная database, TLS (`database.ssl.enabled` в чарте 0.21.1 есть, дефолт **false** — в проде включить, mode `verify-full` на стороне клиента, не только `require`, если доверяете своему CA).
- Бэкап БД = бэкап дашбордов, пользователей, RLS, **зашифрованных** паролей источников. Потеря при живом git-export дашбордов ещё переживаема; потеря без git — нет.
- `superset db upgrade` только через init Job / пайплайн, не руками с трёх подов сразу.
- Сабчарт `postgresql.enabled: true` в проде **выкл**; свой кластер.

#### Redis

- `redis.enabled: false` в проде, свой HA Redis.
- TLS (`cache.ssl` в чарте, дефолт **false**).
- Разделение DB 0/1 как в чарте — минимум, чтобы ключи Celery не дрались с кэшем чартов.
- Без Redis в HA: либо NullCache + нет async, либо файлы/SQLite и сюрпризы sticky.

#### Безопасность прода (без этого кластер не считается настроенным)

Три слоя:

1. **Секреты и идентичность**
   - Свой `SECRET_KEY` **до** заведения datasources. В Helm: `extraSecretEnv.SUPERSET_SECRET_KEY` и/или `configOverrides`; `secretEnv.create: false` + Vault/External Secrets.
   - Нет `admin`/`admin`. Break-glass в Vault. Init `createAdmin` на проде — один раз, потом выключить.
   - SSO (OAuth/LDAP) + `AUTH_ROLES_MAPPING` + `AUTH_ROLES_SYNC_AT_LOGIN = True` (иначе роли «залипнут» с первого входа).
   - Регистрация: `AUTH_USER_REGISTRATION_ROLE` **не** Admin. Типично Gamma или своя роль без sql_lab.
   - `FAB_API_KEY_ENABLED` дефолт выкл — включать только под CI с scopes.
2. **Сеть**
   - UI только из внутренней сети / через SSO-прокси. Не LoadBalancer в интернет.
   - 8088 не с мира; Flower не включать без нужды.
   - Datasource URI только на склады/реплики, которые разрешены NetworkPolicy. `PREVENT_UNSAFE_DB_CONNECTIONS` не трогать.
   - `DATASET_IMPORT_ALLOWED_DATA_URLS` сузить; дефолт `.*` — SSRF при импорте.
3. **Данные, SQL и утечки наружу**
   - `SESSION_COOKIE_SECURE = True`, HTTPS на LB, `ENABLE_PROXY_FIX = True`.
   - `TALISMAN_ENABLED` оставить True; iframe/embed — отдельное решение (`EMBEDDED_SUPERSET` не включать без guest-секрета и CSP).
   - Scarf: образ `apache/superset` (не scarf.sh), `SCARF_ANALYTICS=false`.
   - Public/anonymous не включать на контуре с ПДн.
   - `ENABLE_TEMPLATE_PROCESSING` не включать без письменной модели угроз.
   - SQL Lab: роль узкая; datasource user **SELECT** на витрины, не owner озера. RLS в Superset **плюс** RLS/GRANT в СУБД.
   - `DISALLOWED_SQL_FUNCTIONS` не вычищать «чтобы version() работал».
   - CSV export (`can csv` / `can export csv`) — не всем Gamma.
   - `VIEWER_PROMISCUOUS_MODE` не включать: он позволяет зрителям дашборда обходить проверки датасета.
   - Письма отчётов: `ALERT_REPORTS_WEBHOOK_HTTPS_ONLY = True` (уже дефолт); SMTP внутрь, не личный Gmail из примера Helm.

Плюс:

- не ГОСТ — если ИБ потребует СКЗИ, vanilla Superset это не закроет;
- аудит входов и SQL Lab в SIEM (Wazuh), не только stdout;
- ревью, что Gamma интеграционного namespace не видит payload гособмена;
- Admin — штучные учётки; вендор не считает Admin-XSS уязвимостью.

Webdriver: если отчёты всё же нужны — отдельный образ с Chrome, `WEBDRIVER_BASEURL` на **внутренний** Service (`http://...:8088/`), `WEBDRIVER_BASEURL_USER_FRIENDLY` — внешний HTTPS. `--no-sandbox` в примере Helm нужен из‑за root; если ушли с root — пересмотреть аргументы. Иначе **не включать** Alerts.

##### Сильные / слабые стороны выбранной ИБ-схемы (OSS 6.1.0 + Postgres + Redis + SSO + выключенный Scarf)

| Сильное | Слабое |
|---|---|
| Apache 2.0, не AGPL Grafana | Admin = ключ ко всем datasource passwords и к SQL |
| Дефолты известны; SECRET_KEY без override не стартует (не-debug) | Helm/k8s дефолт `thisISaSECRET_1234` может стартовать — цена «как в helm install» |
| Роли Admin/Alpha/Gamma/sql_lab из коробки | `superset init` затирает ручные правки встроенных ролей |
| Talisman и CSRF включены | CSP часто ломают «вставьте этот скрипт» — соблазн выключить |
| `PREVENT_UNSAFE_DB_CONNECTIONS` | SQL Lab всё равно выполняет SQL, который разрешила учётка БД |
| Не облако, данные в ваших ЦОДах | Нет ГОСТ; драйверы и Chromium расширяют CVE-поверхность |
| `/health` есть | `/health` слеп к Postgres/Redis/складу |
| RLS и subjects | RLS приложения ≠ безопасность склада; SQL Lab / другой BI / прямой JDBC обходят |

#### Kubernetes-специфика прода

- Pin чарта **0.21.1** и образа **6.1.0** (digest). На `latest` уедет минор без вашего решения.
- `postgresql.enabled: false`, `redis.enabled: false`, свои hosts в `database.*` / `cache.*` (в 0.21.1 старый `supersetNode.connections.*` ещё может читаться как legacy — не держать два источника правды).
- Breaking чарта: labels `app.kubernetes.io/*`. Upgrade со старого релиза без удаления Deployment/StatefulSet падает на immutable selector — это в README 0.21.1.
- Bitnami OCI + `bitnamilegacy` образы сабчартов — ещё один аргумент **не** тащить их в прод.
- Ingress `enabled: false` по дефолту; не включать с `chart-example.local` и без TLS.
- `supersetWebsockets`: community-образ `oneacrefund/superset-websocket:latest` — в прод-контур с ПДн **не** тащить `latest` с Docker Hub без своей сборки. Feature `GLOBAL_ASYNC_QUERIES` в 6.1.0 помечен `@lifecycle: testing`. Для первого прода оставить **выкл**.
- Не ставить web на те же ноды, что etcd/Kafka, без лимитов.

#### Склад и производительность (то, что люди называют «Superset тормозит»)

- Читать **реплику** озера, не primary SoT.
- Кэш: Redis для `CACHE_CONFIG` и `DATA_CACHE_CONFIG` (дефолт NullCache в проде неприемлем при «высокой нагрузке»).
- Не ставить dashboard auto-refresh 10s на тяжёлые агрегации (интервал 10s в UI **есть**).
- Агрегации — во вьюхах/материализованных витринах склада, не «SELECT * из сырого озера в чарте».
- `ROW_LIMIT` 50k / `SQL_MAX_ROW` 100k — потолки UI; аналитик упрётся в них раньше, чем в терабайты, если не уйдёт в SQL Lab.
- Timeout'ы согласовать: склад statement_timeout ≤ `SQLLAB_TIMEOUT` ≤ Gunicorn timeout ≤ Ingress timeout. Иначе LB рвёт, а запрос на складе живёт дальше.

#### Порядок вывода в прод (этапы, не команда за командой)

1. Измерить RTT/потери между ЦОДами на 5432 и 6379. Решить: stretch web или мозг в 1–2 площадках.
2. Зафиксировать A vs B1 и **ЦОД Postgres primary + Redis**.
3. Снять оценки: зрители, дашборды, refresh, SQL Lab да/нет, тип склада, ПДн.
4. Поднять HA Postgres (`superset`), HA Redis, бэкап, TLS.
5. Собрать образ 6.1.0 **с драйвером склада**, non-root, без Scarf-gateway в FROM.
6. Выпустить секреты (Vault): пароль БД, `SECRET_KEY`, admin break-glass, SSO, Redis.
7. Helm 0.21.1: ≥2 web, ≥2 worker, beat только если отчёты, без сабчартов, anti-affinity, PDB, `SUPERSET_ENV=production`.
8. Закрыть дефолты: cookie secure, proxy fix, Scarf, Public выкл, ssl к БД.
9. Datasource read-only. 2–3 дашборда из git (K8s-инфра **не** сюда — в Grafana). Учение: убит web-под — UI жив; убит worker — sync чарт жив, async нет; недоступен склад — UI жив, панели failed — и это **ожидаемо**.
10. SSO + Gamma/группы + RLS. Убрать ежедневный вход под Admin. SQL Lab — письменное решение.

Без пунктов 4 (HA БД и Redis!), 7 (не одна реплика!) и 9 у вас нет аналитики, есть namespace с зелёным подом.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | Не требуется | ≥2 web, ≥2 worker, beat≤1, **Postgres HA** (не SQLite), **Redis HA**, LB на `/health`, переживание **1** ЦОДа для UI *если* БД+Redis доступны; runbook на смерть ЦОДа с Postgres; склад — отдельный HA |
| Производительность / масштаб | Не требуется | Кэш не NullCache; не refresh 10s вслепую; терабайты — задача склада; считать коннекты (web+worker)×процессы×pool; драйвер в образе; смотреть latency чартов |
| Безопасность | `admin` только в изоляции; :8088 не с мира; свой `SECRET_KEY`; не боевое озеро | Свой `SECRET_KEY`; нет дефолт-пароля; SSO; non-root; Scarf выкл; TLS+`SESSION_COOKIE_SECURE`+`ENABLE_PROXY_FIX`; Public/embed выкл; SQL Lab ограничен; read-only учётка склада; pin 6.1.0; UI не публичный |

**Не готов к проду**, если: одна реплика с SQLite/bitnami Postgres на три ЦОДа; `CHANGE_ME_...` / `thisISaSECRET_1234`; `admin`/`admin`; `latest`; `runAsUser: 0` + `apt` в bootstrap; Superset в интернет; Public на ПДн; NullCache и «готовы к высокой нагрузке»; SQL Lab под superuser озера; ждут, что Superset *хранит* терабайты; ждут, что это Grafana; RTT не мерили, а Postgres растянули на 3 ЦОДа; два Celery beat; дашборды только в UI без git и без бэкапа Postgres; учение отказа пода/БД/склада не делали.

---

## Источники (чтобы не принимать на веру)

- Релиз 6.1.0 (13 May 2026): https://github.com/apache/superset/releases/tag/6.1.0  
- Source tarball: https://downloads.apache.org/superset/6.1.0/  
- UPDATING.md 6.1.0 (`uv pip`, отказ стартовать на дефолтном SECRET_KEY с 2.1.0): https://github.com/apache/superset/blob/6.1.0/UPDATING.md  
- `CHANGE_ME_SECRET_KEY`: https://raw.githubusercontent.com/apache/superset/6.1.0/superset/constants.py  
- `config.py` 6.1.0 (SQLite URI, NullCache, Celery SQLite, таймауты, флаги, cookie, Talisman, JWT test secrets, `FAB_ADD_SECURITY_API = True`, `DATASET_IMPORT_ALLOWED_DATA_URLS`, `PREVENT_UNSAFE_DB_CONNECTIONS`): https://raw.githubusercontent.com/apache/superset/6.1.0/superset/config.py  
- Configuring Superset: SECRET_KEY, metadata Postgres 10–17 / MySQL 5.7+8, Gunicorn gevent пример `-w 10`, `/health`, `ENABLE_PROXY_FIX`, OAuth redirect, LDAP, prune SQL Lab, лимит layout дашборда 65535: https://superset.apache.org/docs/configuration/configuring-superset  
- Security: роли, Public, subjects, ENABLE_VIEWERS, VIEWER_PROMISCUOUS_MODE, «не firewall», FAB security API (текст доки vs config.py), API keys выкл: https://superset.apache.org/docs/security/  
- Стандартные роли: https://raw.githubusercontent.com/apache/superset/6.1.0/RESOURCES/STANDARD_ROLES.md  
- Docker Compose: не прод, не HA, Windows не поддерживается, `admin`/`admin`, Scarf, volume метаданных без бэкапа: https://superset.apache.org/docs/installation/docker-compose  
- Kubernetes + Helm: SECRET_KEY, `thisISaSECRET_1234`, password `superset`, OAuth-пример с ролью Admin, Alerts+beat+webdriver, Scarf: https://superset.apache.org/docs/installation/kubernetes  
- Helm 0.21.1, app 6.1.0, значения (replicas 1, runAsUser 0, init admin/admin, Flower/beat/ws выкл, `/health`, Bitnami deps): https://artifacthub.io/packages/helm/superset/superset/0.21.1 · Chart.yaml тега `superset-helm-chart-0.21.1`  
- Async SQL Lab / Celery / RESULTS_BACKEND Redis: https://superset.apache.org/docs/configuration/async-queries-celery (и зеркало docs в репозитории)  
- Operator 0.2.0 (11 Aug 2026), BYO DB/cache, production secrets via `*From`: https://github.com/apache/superset-kubernetes-operator/releases · https://github.com/apache/superset-kubernetes-operator  
- SIP-149 (Helm deprecate after operator stable): https://github.com/apache/superset/issues/31408  

Утверждения вида «Superset переживёт два ЦОДа» или «N millicores на терабайт озера» в документации проекта **отсутствуют** — поэтому в этом файле их нет. RTT между ЦОДами на путь *браузер → web* влияет как на любой HTTPS; на Postgres/Redis — как на любой синхронный клиент. Порога в миллисекундах у Superset нет: его надо измерить у себя, а не взять из блога.
