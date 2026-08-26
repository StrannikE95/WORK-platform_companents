# Grafana 13.2.0 — термины и сокращения

Словарь к файлу `Grafana.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа в операционной системе: у неё есть память и открытые файлы. Grafana — один такой процесс `grafana` (HTTP UI и API). Несколько реплик Grafana — несколько одинаковых процессов, не выборы лидера.

**Файл / каталог на диске** — то, что остаётся после рестарта процесса. У Grafana на диске — конфиг `grafana.ini` / `custom.ini` и (если не вынесли) файл SQLite. Дашборды и пользователи в проде живут в отдельной СУБД, не в этом файле процесса.

**TCP-порт** — номер, по которому другие программы находят этот процесс в сети. У Grafana UI/API — **3000**. Служебный обмен встроенных Alertmanager — **9094** (и TCP, и UDP).

**ЦОД** — отдельная площадка с серверами (своё питание, своя сеть). «Три ЦОДа» в тексте — три такие площадки.

**RTT** — время туда-обратно по сети между двумя машинами. Документация Grafana порога в миллисекундах не даёт. Для обмена на 9094 есть `ha_peer_timeout` (дефолт **15s**).

**TLS** — шифрование байтов в сети. Grafana умеет `protocol = https` сама; типичный прод — TLS на reverse proxy перед 3000.

**HTTP** — протокол запрос-ответ по TCP. UI, API, `/metrics`, `/api/health` — это HTTP на 3000. WebSocket Live — upgrade того же HTTP.

**WebSocket** — длинное TCP-соединение, по которому сервер шлёт события в браузер без нового HTTP-запроса. Grafana Live так работает. На процесс дефолт **max_connections = 100**; ориентир памяти ~**50 КБ** на соединение.

**Под (Pod)** — минимальная единица запуска в Kubernetes: контейнер(ы), которые живут и умирают вместе. IP пода меняется; клиент ходит в DNS сервиса / Ingress.

**Helm / чарт** — шаблон установки манифестов Kubernetes. Для Grafana 13.2.0 в файле — чарт **`grafana-community/grafana` 12.11.2**. Grafana Operator **5.25.0** — другой инсталлятор, не тот же чарт.

**PVC** — заявка Kubernetes «дай том такого размера». **RWO** — том можно отдать только одному поду. SQLite на RWO при нескольких репликах Grafana не работает: либо Multi-Attach, либо три разных файла БД.

**PDB (PodDisruptionBudget)** — лимит, сколько подов можно снять сразу при плановом обслуживании ноды. Для Grafana в файле: не убивать все реплики (`minAvailable: 1` как минимум).

**NetworkPolicy** — правила Kubernetes, кто в кластере может открыть TCP/UDP на порты Grafana. Это не RBAC внутри Grafana.

**Grafana** — веб-приложение: рисует графики, дашборды, шлёт алерты. Само **не хранит** ряды метрик. Оно ходит HTTP/SQL к другим системам и показывает ответ.

**Data source (источник данных)** — запись подключения, откуда Grafana читает: Prometheus, Loki, OpenSearch, PostgreSQL, Tempo. Без источника дашборд пустой. Пароли источников лежат в БД Grafana, зашифрованные `secret_key`.

**Dashboard / panel** — экран из панелей. Панель = один запрос к источнику + картинка (график, таблица).

**Provisioning** — дашборды, источники и алерты кладут файлами YAML/JSON при старте процесса (каталог provisioning / ConfigMap sidecar), а не только кликами в UI.

**Explore** — экран «выполнить запрос к источнику» без сохранения дашборда.

**Unified Alerting** — штатный алертинг Grafana 13.x. Старый «alert на панели» выключен. Правило = запрос к источнику + условие.

**Alert rule** — правило «что сломалось»: PromQL/запрос + порог.

**Contact point** — куда писать при срабатывании: email, webhook, Telegram.

**Notification policy** — каким маршрутом правило попадает в contact point.

**Scheduler** — часть процесса Grafana, которая по расписанию **оценивает** правила (ходит в Prometheus и другие источники). По умолчанию **каждый** под Grafana считает **все** правила.

**Alertmanager (встроенный)** — часть того же процесса Grafana, которая **отправляет** уведомления. Это не отдельный бинарь Prometheus Alertmanager, пока вы его сами не подключите снаружи.

**HA Grafana** — несколько процессов Grafana за балансировщиком, все смотрят в **одну** общую БД (PostgreSQL 12+ или MySQL 8.0+). Любой живой узел обслуживает UI. Жив под, мертва БД — UI нет.

**SQLite** — встроенная файл-БД по умолчанию: один файл на одном диске. Для прода и для нескольких реплик документация установки запрещает.

**Shared database** — общая MySQL 8.0+ или PostgreSQL 12+, куда все реплики Grafana пишут пользователей, дашборды, состояние алертов. HA этой БД — отдельное ПО (Patroni / CloudNativePG), не Grafana.

**Memberlist / gossip** — обмен служебными сообщениями между встроенными Alertmanager, чтобы **не слать одно уведомление N раз**. Порты **9094/TCP и 9094/UDP**. Оба протокола обязательны. Закрытый UDP между ЦОДами ломает этот обмен.

**`ha_peers`** — список соседей для gossip (или DNS headless-сервиса в Kubernetes). Без этого несколько реплик = дубли писем при живом UI.

**`ha_listen_address` / `ha_advertise_address`** — на каком IP:9094 слушать и какой адрес объявлять соседям. В Kubernetes обычно `${POD_IP}:9094`.

**`ha_peer_timeout`** — сколько ждать соседа. Дефолт **15s**. Поднимать только после замера RTT, не «на всякий случай 2 минуты».

**Single-node evaluation** — режим (`ha_single_node_evaluation = true`): правила считает **один** узел, остальные только UI. В Grafana 13.x — **public preview**. Снижает нагрузку на источники в N раз. При смене primary есть короткая дыра оценки. Очередь broadcast дефолт **200**.

**Public preview** — статус функции: ломающие изменения возможны, поддержка Grafana Labs ограниченная. Не GA.

**Grafana Live** — WebSocket «события в браузер сразу» (правки дашборда, стримы). По умолчанию **in-memory** в процессе. В HA без Redis стримы не общие между подами. Проверка Origin завязана на `root_url`.

**Redis** — отдельный процесс-хранилище ключей в памяти (порт **6379**). Для Grafana: опция HA алертинга **или** Live HA engine. Это не часть образа Grafana. Sentinel для Live официально **не** поддерживается; рекомендуют Redis Cluster.

**Image Renderer** — **отдельный** процесс (Chromium): рисует PNG дашборда для письма. Не часть бинаря Grafana. Типичный порт **8081**. Дефолтный токен `-`. Исторически были уязвимости чтения файлов через render.

**`secret_key`** — ключ, которым Grafana шифрует пароли источников в своей БД (AES, envelope encryption, провайдер по умолчанию `secretKey.v1`). В `defaults.ini` 13.2.0 зашито `SW2YcwTIb9zpOOhoPsMm` на все установки. Смена после сохранения источников требует пересохранения паролей / штатной миграции encryption.

**Envelope encryption** — пароль источника шифруется ключом; сам ключ тоже завёрнут. KMS-провайдеры (`available_encryption_providers`) в OSS ограничены, в основном Enterprise.

**`admin` / `admin`** — логин и пароль первого администратора. Ставятся **один раз** при первом старте. Потом правка `grafana.ini` пароль в БД **не меняет**.

**Organization / folder / RBAC** — организация = арендатор внутри Grafana. Папки и роли (Viewer / Editor / Admin + finer RBAC) режут, кто какие дашборды видит.

**Viewer / Editor / Admin** — роли. Viewer с доступом к источнику видит данные запросов (метрики, лаги, иногда ПДн в labels). Admin = доступ к зашифрованным паролям источников, если есть `secret_key`.

**Service account** — машинный пользователь с токеном для API (GitOps, алерты). Старые API keys вендор выводит.

**Anonymous / externally shared dashboard** — анонимный вход без логина / публичная ссылка на дашборд. Для контура с ПДн в файле это дыра. RBAC-право `dashboards.public:write`.

**Data source proxy** — Grafana **с сервера** ходит в источник от имени пользователя. Grafana видит внутреннюю сеть. Это SSRF: процесс Grafana может быть заставлен сходить на внутренний HTTP. В проде — `data_source_proxy_whitelist`.

**SSRF** — сервер по указанию клиента открывает HTTP на адрес во внутренней сети. Data source proxy — такая точка.

**`/metrics`** — служебные метрики самого процесса Grafana (HTTP на 3000). По умолчанию **без** basic auth, пока не зададите `basic_auth_username` / `basic_auth_password`.

**`/api/health`** — HTTP-точка для балансировщика: процесс жив. Не проверяет Prometheus и не проверяет Postgres.

**`grafana_alertmanager_cluster_members`** — метрика числа соседей gossip. С Grafana **v12.4** префикс `grafana_`.

**`root_url`** — канонический URL, как пользователь открывает Grafana в браузере (`https://grafana.example.internal/`). Без него ломаются OAuth callback и проверка Origin у Live.

**`enforce_domain`** — после того как LB шлёт правильный `Host`, включить проверку домена.

**Sticky session** — балансировщик всегда шлёт одного пользователя на один под. Для Grafana **не обязательна**: сессии живут в общей БД.

**LGTM** — связка Grafana Labs: **L**oki (логи), **G**rafana, **T**empo (трейсы), **M**imir (метрики). Grafana — буква G, не весь стек.

**Prometheus / Mimir** — хранилища метрик. Prometheus — локальный TSDB на диске процесса. Mimir — долгий/большой объём. Grafana их **запрашивает**, не заменяет.

**Loki / Tempo** — логи и трейсы. Grafana их спрашивает как источники. Не хранит.

**kube-prometheus-stack** — Helm «Prometheus Operator + экспортёры + Grafana». Grafana там **вложенный** чарт. Дефолты стека ≠ прод-HA Grafana.

**Stretch** — один логический Grafana-кластер (одна БД, одни `ha_peers`), поды физически в нескольких ЦОДах. Gossip 9094 и Postgres чувствуют RTT.

**`grafana.ini` / `custom.ini` / `GF_*`** — конфиг процесса. Переменные окружения `GF_*` перекрывают ini.

**OSS / Enterprise** — образ `grafana/grafana:13.2.0` (AGPLv3) и `grafana/grafana-enterprise:13.2.0`. Без лицензии Enterprise = тот же OSS-функционал. Коммерческие плагины и KMS в файле не включаются.

**AGPLv3** — лицензия OSS Grafana. Модификация и сетевое предоставление — вопрос юротдела, не Helm.

**`reporting_enabled`** — дефолт `true`: счётчики на `stats.grafana.org` каждые 24 часа. В закрытом контуре выключают.

**`check_for_updates` / `check_for_plugin_updates`** — дефолт `true`: GET на `grafana.com`. Версию сами не ставят; контур «звонит домой».

**`disable_gravatar`** — дефолт `false`: браузер/сервер ходит на `secure.gravatar.com`. В закрытом контуре `true`.

**`snapshots.external_enabled`** — дефолт `true`: кнопка публикации на `https://snapshots.raintank.io`. В проде выкл; часто и `snapshots.enabled = false`.

**`cookie_secure`** — дефолт `false`: cookie сессии может уехать по HTTP. В проде `true` при HTTPS.

**`cookie_samesite`** — для OAuth документация: `lax`. Значение `strict` ломает OAuth.

**`allow_embedding`** — дефолт `false`: не вставлять Grafana в iframe без нужды.

**`allow_loading_unsigned_plugins`** — загрузка неподписанных плагинов. В проде не разворачивать.

**`min_refresh_interval`** — не дать панели refresh чаще **5s** (дефолт).

**`min_tls_version`** — если Grafana сама терминирует TLS: дефолт TLS 1.2.

**Image tag `latest`** — ярлык, который завтра укажет на другой патч. В проде пинят `13.2.0` (digest).

**Headless Service** — DNS Kubernetes без ClusterIP: имена подов для `ha_peers`. В чарте `headlessService: true`.

**Anti-affinity / topology spread** — правило планировщика: не класть все реплики Grafana в один ЦОД/на одну машину.

**SSO / Generic OAuth / LDAP** — вход через IdP. Локальный `admin` в проде — break-glass. Client secret — в Vault.

**Vault** — хранилище секретов. Туда: пароль БД Grafana, `secret_key`, client secret SSO, токены service account. Шифрование паролей в БД Grafana Vault не заменяет.

**HAProxy / Ingress** — reverse proxy перед 3000. Нужна поддержка WebSocket (`Upgrade`/`Connection`), если Live включён. Readiness — `/api/health`, не «TCP 3000 открыт».

**SPOF** — единственная точка, падение которой гасит сервис. Для Grafana это общая БД: падение Postgres = падение всех реплик Grafana.

**Remote evaluation** — вынос оценки алертов на отдельные инстансы, когда правил >1000 или interval < 1 мин. Отдельная схема, не `replicas: 5` на том же Prometheus.

**Внешний Alertmanager** — отдельный процесс экосистемы Prometheus. Опция, если встроенный gossip не справляется; не часть дистрибутива Grafana.

**SIEM / Wazuh / Falco / WAF** — другой контур (security). Дашборд Grafana их не заменяет.

**ГОСТ TLS / СКЗИ** — в Grafana не заявлено. Штатный TLS 1.2/1.3 — не криптография по требованиям КИИ.

**Availability over consistency** — формулировка встроенного Alertmanager: возможны редкие дубли или перестановка уведомлений; вендор считает это лучше, чем тишина.

**InitChownData** — init-контейнер чарта, меняет владельца файлов на диске. Для HA без PVC часто выключают.

**Sidecar provisioning** — контейнер рядом кладёт дашборды из ConfigMap. UI-правки будут перезатёрты. Источник правды: git **или** клики.

Источники формулировок: глоссарий и тело `Grafana.md`. Новых порогов RTT и размеров диска здесь нет.
