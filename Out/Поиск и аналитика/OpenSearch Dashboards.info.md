# OpenSearch Dashboards 3.8.0 — термины и сокращения

Словарь к файлу `OpenSearch Dashboards.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа: память и открытые файлы. OpenSearch Dashboards — процесс Node.js (UI + прокси к REST 9200). Несколько реплик — несколько одинаковых процессов. Лидер не выбирается: достаточно любого живого процесса за балансировщиком.

**Файл / каталог на диске** — то, что остаётся после рестарта. У OSD на диске пода почти ничего: saved objects живут в индексах OpenSearch, не на PVC Dashboards. Конфиг — `opensearch_dashboards.yml`.

**TCP-порт** — номер в сети. OSD слушает **5601**. Исходящие запросы OSD → OpenSearch — **9200**. UDP нет. Порт **9600** — Performance Analyzer **ноды OpenSearch**, не Dashboards.

**ЦОД** — отдельная площадка. Реплики OSD размазывают по зонам Kubernetes. UI в «чужом» ЦОДе без HTTP 9200 до кластера бесполезен.

**RTT** — время туда-обратно. Для OSD важен путь **подов OSD → coordinating 9200**, не ping до Ingress. Высокий RTT → тормозит Discover, таймауты `opensearch.requestTimeout`.

**TLS** — шифрование байтов в сети. `server.ssl.*` — браузер ↔ OSD (5601). `opensearch.ssl.*` — OSD ↔ OpenSearch (9200). По умолчанию для простоты старта 5601 — **HTTP**. ГОСТ/СКЗИ не заявлено.

**Под (Pod)** — контейнер(ы) Kubernetes. OSD — **Deployment**: поды взаимозаменяемы. PVC под данные поиска **не нужен**. `emptyDir` для кэша Node допустим.

**Helm / оператор** — целевой путь: секция `spec.dashboards` в OpenSearch Kubernetes Operator, `version: 3.8.0` = версия кластера. Запасной: Helm-чарт `opensearch/opensearch-dashboards`. Дефолт чарта: `replicaCount: 1`, `updateStrategy: Recreate`, resources **100m CPU / 512M RAM**.

**PDB** — Kubernetes не имеет права снять сразу все реплики OSD при drain. При 2 репликах `minAvailable: 1`; при 3 — разумнее `minAvailable: 2`.

**Anti-affinity / topology spread** — не складывать все реплики OSD в одну зону/узел.

**Ingress / Service** — точка входа пользователей на 5601. TLS для людей обычно на Ingress.

**OpenSearch Dashboards (OSD)** — веб-приложение (Node.js): Discover, визуализации, дашборды, часть админки Security, Index Management. Порт **5601**. Кластер поиска живёт без UI; UI без живого кластера бесполезен.

**opensearch_dashboards.yml** — главный конфиг процесса. Без него контейнер возьмёт дефолты образа (часто demo). Невалидный ключ = процесс **не стартует**.

**Saved objects** — то, что нарисовали в UI: index patterns, visualizations, dashboards, saved searches. Живут **не на диске пода**, а в индексах OpenSearch. Удалили volume OSD — дашборды на месте; удалили индекс — нет.

**`.kibana` / `.opensearch_dashboards`** — индекс(ы), куда OSD пишет saved objects. Имя должно совпасть с `kibana.index` в Security config (`index: '.kibana'` по умолчанию).

**Tenant (тенант)** — пространство saved objects внутри одного кластера. Global — общий, Private — личный на пользователя, плюс именованные. Security plugin для каждого тенанта плодит **отдельный** индекс вида `.kibana_<hash>_<имя>`. Сотни пользователей × private = сюрприз для heap cluster manager. Multi-tenancy в `config.yml` Security по умолчанию **включена**; схема плагина OSD в коде имеет дефолт `false` — задать явно.

**kibanaserver** — служебный пользователь, которым **сам процесс** OSD ходит в OpenSearch обслуживать свои индексы. Это **не** учётка аналитика. Дефолт demo: пароль `kibanaserver`. Должен быть в роли `kibana_server`. Оператор без секрета генерирует случайный пароль в `<cluster>-dashboards-password`.

**kibana_server (роль)** — reserved-роль с правами на `.kibana*`, `.opensearch_dashboards*`, `.tasks` и служебные шаблоны. Если `server_username` ≠ `kibanaserver`, пользователя **надо явно** привязать к роли.

**Security Dashboards plugin** — плагин UI к Security plugin кластера: логин, роли, тенанты, cookie-сессия. Ключи — `opensearch_security.*`. Без плагина ключи вроде `opensearch_security.cookie.secure` **не существуют**, процесс падает с `Unknown configuration key`. В стандартном дистрибутиве плагин есть.

**Cookie-сессия** — после логина OSD кладёт в браузер cookie `security_authentication` (имя по умолчанию). Сессия **не** хранится в Redis и **не** в отдельной БД OSD. Любая реплика OSD может принять cookie, **если** у всех одинаковый `opensearch_security.cookie.password`.

**cookie.password** — секрет шифрования cookie. Минимум **32 символа**, дефолт **`security_cookie_default_password`** (ровно 32, общеизвестен). Кто знает дефолт — может подделать сессию. Один секрет на все реплики, в Secret.

**cookie.secure** — флаг «cookie только по HTTPS». Дефолт плагина **`false`**. Если включили HTTPS у OSD / браузер ходит HTTPS — ставить **`true`**. Классика поломки: Ingress HTTPS, OSD HTTP внутри, `cookie.secure: true` → логин не работает.

**session.ttl / cookie.ttl** — срок жизни сессии и cookie в **миллисекундах**. Дефолт обоих **3600000** (1 час). `session.keepalive` (дефолт `true`) продлевает TTL при активности.

**opensearch.hosts** — список URL **нод одного и того же** кластера OpenSearch (fallback, если одна не отвечает). Это **не** «два разных кластера в одном UI». Ходить на coordinating/ingest, **не** на dedicated cluster manager.

**Multiple data sources** — фича: `data_source.enabled: true`. Тогда в UI можно добавить **другие** кластеры. Выключено по умолчанию. Timeline-визуализации с этим режимом **не** поддерживаются.

**SSO (SAML / OpenID Connect)** — вход через корпоративный IdP. Настраивается **в двух местах**: `config.yml` Security plugin на кластере **и** `opensearch_dashboards.yml`. Для SAML ACS-URL надо в `server.xsrf.allowlist` (`/_opendistro/_security/saml/acs`). ACS URL — стабильное имя LB, не Pod IP.

**multiple_auth** — экран логина с несколькими кнопками. Допустимые типы: `basicauth`, `openid`, `saml`. Если типов больше одного — **`basicauth` обязателен** в массиве.

**default_redirect_auth_type** — авторедирект на SAML/OIDC, минуя форму. Обход для админа: `?auto_login=false`.

**Proxy auth** — OSD доверяет HTTP-заголовкам с reverse-proxy. Опасно, если 5601 доступен в обход proxy: заголовки можно подделать.

**server.ssl.*** — TLS **браузер ↔ OSD** (порт 5601).

**opensearch.ssl.*** — TLS **OSD ↔ OpenSearch** (9200). `verificationMode`: `full` (сертификат + hostname, дефолт и рекомендация), `certificate`, `none` (docker-примеры; для прода нет).

**alwaysPresentCertificate** — OSD предъявляет клиентский сертификат на 9200 (mTLS). Дефолт `false`.

**server.basePath / rewriteBasePath** — OSD за reverse-proxy на подпути (`/logs`). У оператора поле `basePath` само проставляет `rewriteBasePath: true`. Ingress должен совпадать.

**NODE_OPTIONS / max-old-space-size** — лимит кучи **V8** (это не JVM). Helm приводит пример `--max-old-space-size=1800`. Без этого Node упирается в дефолт кучи и умирает, не дойдя до лимита Kubernetes. Лимит Kubernetes должен быть **выше** этой кучи.

**updateStrategy: Recreate** — дефолт официального Helm-чарта: при выкате сначала убиваются все поды, потом встают новые. Это **простой UI**. Для HA меняют на `RollingUpdate`.

**Node.js** — в пакете OSD 3.5+ лежит **Node.js 22**. Свой runtime — `NODE_OSD_HOME` / `NODE_HOME`. Для контейнера обычно не трогают.

**Заголовок `securitytenant`** — если его нет в `opensearch.requestHeadersAllowlist` / `requestHeadersWhitelist` вместе с `Authorization`, документация: OSD стартует с **red** status.

**index pattern** — привязка UI к полям индекса OpenSearch. Без него Discover пустой.

**Discover** — экран поиска документов. Тяжёлый Discover бьёт data/coordinating OpenSearch, не «добавь пятый Dashboards».

**auth.anonymous_auth_enabled** — дефолт **false**. Включать анонима в проде = публичное чтение того, что разрешите роли anonymous.

**metricsPort 9601** — опция Helm: ServiceMonitor на `/_prometheus/metrics`. Не строка обязательных портов install-гайда OSD.

**HPA** — автомасштаб реплик OSD в чарте (`autoscaling.enabled`, CPU/memory 80%). Нужен Metrics Server. Узкое место часто 9200, а не 5601.

**NDJSON** — формат экспорта/импорта saved objects. Боевые дашборды желательно уметь накатить из файла, иначе единственная копия — в кластере.

**Snapshot `.kibana*`** — часть snapshot-политики кластера OpenSearch, не галочка в OSD. Документация multi-tenancy: бэкап — snapshot tenant indexes.

**IdP** — Keycloak / AD FS / иное. Падение IdP = нельзя войти, если нет `basicauth` в запасе.

**Wazuh indexer** — отдельный кластер/UI. Подключать как data source в боевой OSD = смешать роли ИБ и аналитиков.

**Grafana / PromQL** — другое ПО. OSD рисует OpenSearch DSL/PPL, не заменяет Prometheus.

**SoT** — карточка в Discover — проекция для поиска, не источник истины.

**Image tag `latest`** — вендор в гайде пишет `latest` для OpenSearch — **не копировать**. Тег **3.8.0**, как у кластера. Плагины OSD = major.minor.patch кластера.

**Probes** — в чарте TCP 5601. Это «порт открыт», не «логин и `.kibana` зелёные».

**client_secret OIDC** — из Secret, в yml через `${ENV}`. Смена секрета = рестарт подов OSD (оператор: без рестарта конфиг не подхватит).

**Браузеры** — Chrome, Firefox, Safari, Edge (Chromium). Internet Explorer и Edge Legacy — нет.

Источники формулировок: глоссарий и тело `OpenSearch Dashboards.md`. Новых порогов RTT и размеров диска здесь нет.
