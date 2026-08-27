# Backstage 1.54.0 — термины и сокращения

Словарь к файлу `Backstage.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа Node.js. Backend Backstage слушает HTTP **7007**. Несколько подов **не выбирают лидера**; состояние — в PostgreSQL.

**Файл / каталог на диске** — каталог сущностей живёт в PostgreSQL, не в PVC пода Backstage. SQLite `:memory:` при рестарте даёт пустой каталог. TechDocs `builder: local` пишет на диск **этого** пода: реплики сгенерированные доки не делят.

**TCP-порт** — **7007** backend (и UI, если frontend в том же процессе). **3000** — dev-сервер frontend при `yarn start`; в проде этого порта нет, если UI раздаёт backend. UDP нет. HTTPS на 7007 обычно заканчивается на Ingress.

**ЦОД** — отдельная площадка (`topology.kubernetes.io/zone`). Цель отказа: пережить **1 ЦОД**, не 2 из 3.

**RTT** — время туда-обратно. Для Node-подов важен RTT до **PostgreSQL** и SCM, не «магия Backstage». Порога в документации проекта нет. Мерить TCP до 5432 и HTTPS до SCM, не только ping ICMP.

**TLS** — на Ingress, обычный. ГОСТ/СКЗИ vanilla Backstage не закрывает. Backstage token — свой JWT, это не замена TLS до пользователя.

**Developer portal / IDP** — внутренний сайт для разработчиков: какие сервисы, кто владелец, где репозиторий, как поднять новый. Не портал для граждан и не UI озера ПДн. (IDP здесь — Internal Developer Platform, не Identity Provider; IdP для входа — отдельно.)

**App** — *ваш* экземпляр Backstage: монорепозиторий из `create-app`, дальше сами наполняете плагинами. Проект Backstage — **фреймворк**, не готовый бинарь. Официальный Docker-гайд собирает образ из репозитория app; базовый runtime шаблона 1.54 — `node:24-trixie-slim`.

**Core** — база платформы (конфиг, плагины, сервисы backend), которую развивает open-source проект.

**Plugin** — кусок функциональности (каталог, шаблоны, документация, Kubernetes-вкладка). Frontend-плагин рисует UI; backend-плагин — API и работу на сервере. Путь API: `/api/<pluginId>/…`.

**Software Catalog** — реестр сущностей: сервисы, API, сайты, библиотеки, люди, группы. Источник правды *о ПО организации*, не о клиентах.

**Entity** — одна запись каталога (`Component`, `API`, `System`, `User`, `Group`, `Template`, …). Описывается YAML `catalog-info.yaml`.

**Location** — откуда каталог берёт YAML: URL в Git, файл, discovery по организации. В проде `file:` для боевых данных не рекомендуют.

**Entity provider** — модуль, который *сам* кладёт сущности в каталог по расписанию или webhook (организация в GitLab, LDAP, …). `createBackendModule` живёт **в том же** процессе, что плагин, который расширяют. Нельзя вынести GitHub entity provider в другой Deployment отдельно от catalog.

**Processor** — шаг конвейера каталога: прочитать YAML, проверить, обогатить, вытащить связанные сущности.

**Processing loop** — фоновый цикл: каталог периодически заново обрабатывает все сущности. Интервал — `catalog.processingInterval` (желаемый минимум, фактически может быть длиннее). Слишком короткий интервал сожрёт rate limit GitLab. Метрика: `catalog.processing.duration`.

**Stitching** — финальная сборка сущности (связи, поиск по полям) после обработки. Отдельная очередь.

**catalog.rules** — какие kind можно регистрировать. `User`/`Group` — только из доверенного provider, не из произвольного PR. `Template` — узкий allow-list локаций.

**catalog.readonly** — каталог зеркало Git, UI-регистрация локаций запрещена.

**Scaffolder / Software Templates** — мастер создания: шаблон генерирует репозиторий/файлы и выполняет **действия на хосте backend**. Это не Camunda и не процесс заявки в госорган. Job выполняется на хосте пода. Часто широкие права на Git — главный attack surface портала.

**TechDocs** — документация рядом с кодом (Markdown → MkDocs). `builder: local` считает MkDocs *в поде*; `external` только читает из S3/GCS/Azure/MinIO. Recommended setup — **external** + объектное хранилище. Для MinIO: `endpoint` + `s3ForcePathStyle: true`. Кэш TechDocs требует ещё `backend.cache`.

**Search** — поиск по каталогу/докам. Сам Backstage поисковиком не является: подключают Lunr, PostgreSQL или Elasticsearch/OpenSearch.

**Lunr** — поиск **в памяти процесса**. В create-app это дефолт, пока нет Postgres. В проде официально **крайне не рекомендован**. На нескольких репликах у каждого процесса свой индекс в RAM.

**search-backend-module-pg** — поиск в PostgreSQL (дока: десятки тысяч документов). Модуль требует Postgres **≥ 12**.

**search-backend-module-elasticsearch** — поиск во внешнем ES/OS (`provider: opensearch`). Индекс портала не смешивать с бизнес-логами/ПДн.

**Auth provider** — способ войти: GitLab, GitHub, OIDC, SAML, proxy (ALB/IAP/Cloudflare). Один обычно для sign-in, остальные — чтобы плагины ходили во внешние API от имени пользователя.

**Guest** — вход «гостем» без IdP. В доке: удобно не в проде; в Docker-гайде — **не для контейнеров**. В production SignInPage нет.

**Sign-in resolver** — код/настройка: «этот аккаунт IdP = вот этот `User` в каталоге». Ошибка resolver = чужая учётка или вход постороннего. В 1.54.0 allowlist OAuth **ужесточили** (breaking): redirect URI по компонентам URL.

**IdP** — сервер учётных записей (GitLab, Keycloak, Entra). SSO в проде, не Guest.

**Backstage token** — JWT, которым UI доказывает backend «я вот этот пользователь». Ключи подписи **уникальны для инсталляции**, короткоживущие, по умолчанию **не переживают рестарт** процесса, если не задать свой keystore. После rolling update возможен «неизвестный kid».

**auth.keyStore.provider: static** — свои файлы ключей, чтобы ключи переживали рестарт.

**Limited user token / UIP** — урезанный токен: доказывает *кто* пользователь, но большинство API его не принимают. Им прокидывают identity между плагинами.

**Service-to-service auth** — плагины backend ходят друг к другу со своими JWT; публичные ключи — `/.backstage/auth/v1/jwks.json`.

**JWKS** — набор публичных ключей для проверки JWT.

**Permission framework** — слой «можно ли это действие». **По умолчанию выключен**: все залогиненные видят и делают почти всё. Включается `permission.enabled: true` + политика.

**Permission policy** — ваша функция: разрешить / запретить / условно разрешить. В create-app часто стоит allow-all модуль — это не RBAC «из коробки».

**Proxy plugin** — прокси из браузера на внешний HTTP (обход CORS). Опасно, если в него вшивают секреты: любой, кто дошёл до Backstage, бьёт ими во внешний API. Не инжектить Authorization во upstream; резать `allowedMethods`.

**UrlReader** — сервис backend: «скачай этот URL». Список доверенных хостов — `backend.reading.allow`. Ошибка списка = чтение metadata облака (`169.254.169.254`) или внутренних админок.

**`$text` в YAML** — placeholder: backend может прочитать то, до чего у пользователя нет прав в Git. Если это проблема — отключить/заменить placeholder resolvers.

**DiscoveryService** — «по какому URL жить плагину `catalog` / `auth`». При **нескольких** backend-деплоях его надо настраивать самим: из коробки мультидеплой «сам разъедется» не обещан. Frontend `DiscoveryApi` тоже, если не закрыли маршрутизацию прокси.

**app.baseUrl / backend.baseUrl** — публичные URL UI и API. Если не совпадают с тем, что видит браузер (Ingress, TLS, путь) — логин и CORS ломаются.

**CORS** — политика браузера, какие origin могут вызывать API.

**app-config.yaml** — главный конфиг. Секреты — через `${ENV}` / Secret, не в git. Слои: `app-config.yaml` + `app-config.production.yaml`.

**Knex** — библиотека SQL. Каждый backend-плагин получает **свою логическую БД** (или схему). Плагины на одном хосте с доступом к **чужой** логической БД считаются имеющими полный доступ к тому плагину.

**pluginDivisionMode** — `database` (дефолт: отдельная БД на плагин, `backstage_plugin_*`) или `schema` (одна БД PostgreSQL, схемы на плагин).

**SQLite / better-sqlite3** — файловая/in-memory БД для эксперимента. Дефолт create-app — `:memory:`.

**PostgreSQL** — предпочтительная БД прода. Политика: поддерживаются **последние 5 major**, тестируют самый новый и самый старый. На август 2026 ориентир **14…18**. Пример k8s-гайда `postgres:13.2-alpine` — **учебный YAML**, не официальная версия для прода. HA БД — отдельный контур. Реплики каталога данные друг другу не копируют.

**Cache store** — кэш плагинов (Keyv): `memory` / `redis` / `valkey` / `memcache` / `infinispan`. `memory` — для локальной разработки.

**New backend system** — текущий способ собирать backend: `createBackend()` + `backend.add(import('плагин'))`. Старый `src/plugins/*.ts` — миграция.

**Health (с 1.29)** — `/.backstage/health/v1/liveness` и `/.backstage/health/v1/readiness` для проб Kubernetes. Пример k8s-гайда с `/healthcheck` — «если включили в app». `/metrics` в свежем app **нет**, пока сами не настроите (Helm README).

**Host build** — сборка JS на CI (`yarn tsc` + `yarn build:backend`), в Docker кладут уже bundle. Рекомендованный путь.

**Node.js** — ровно **две соседние чётные** версии. С релиза **1.46.0** это **22 и 24**. Шаблон Docker 1.54 — **Node 24**. yarn и runtime — **одной** major: нативные модули иначе падают.

**Yarn 4.4.1** — Getting started на дату доки (`corepack` + `yarn set version 4.4.1`).

**CNCF Incubating** — проект в CNCF с 15 марта 2022 (принят 8 сентября 2020). Лицензия **Apache 2.0**. License-server нет. Плагины из npm — vetting как любого пакета.

**Release line** — Main раз в месяц (вторник перед третьей средой). Next — еженедельно, меньше гарантий. Версия `1.54.0` **не semver продукта**: ломающие изменения бывают в minor. Релиз 1.54.0 содержит **critical security fixes** плагина Kubernetes.

**Skew policy** — ядро app не старше плагинов «наоборот»; frontend-плагин и его backend обновляют вместе, backend **раньше или одновременно**.

**Stretch** — один логический сервис, поды/БД в нескольких ЦОДах. Для Backstage «растягивается» в первую очередь **PostgreSQL**, не Node-процесс.

**RPO / RTO** — сколько данных готовы потерять / как быстро восстановиться. RPO портала ≈ RPO PostgreSQL + то, что ещё не успел обработать catalog из Git. Формального SLA нет.

**Helm chart `backstage/backstage` 2.8.2** — 17 июня 2026. Дефолтный образ — vanilla demo, **для прода скорее всего не подходит**. `postgresql.enabled` чарта — не прод-БД (standalone Bitnami, workaround-образ `bitnamilegacy/postgresql`). PDB в чарте выключен по умолчанию (`create: false`). HPA чарта: `targetCPUUtilizationPercentage: 80`, `maxReplicas: 100` — дефолты, не расчёт нагрузки. Prerequisites: K8s **1.25+** (в таблице requirements ещё `>= 1.19` — расхождение в README).

**plugin-app-backend** — по умолчанию frontend раздаётся **тем же** backend. Вынос в NGINX/CDN: конфиг **запекается на сборке**.

**`backend.auth.dangerouslyDisableDefaultAuthPolicy`** — выключает защиту API от анонимов (есть с backend ≥ 1.24). Не включать. Frontend-бандл по умолчанию **всё равно скачивается**; закрыть — experimental public entry.

**oauth2Proxy** — провайдер по заголовкам. **Не проверяет** подпись заголовков — внутренний злоумышленник может подделать identity. `awsAlb` / `cfAccess` / `gcpIap` — проверяют.

**Аутентифицирующий reverse proxy** — AWS ALB, GCP IAP, Cloudflare Access *плюс* нормальный sign-in. Рекомендуемый вход. Threat model: рассчитан на **защищённый контур**, не на публичный интернет. DoS — зачаточная защита; WAF/лимиты ставит оператор.

**split backend** — отдельные Deployment: auth (своя БД, обязательно для high-security), catalog+search, scaffolder, остальное. Нужен свой DiscoveryService. Начинать со split «на всякий случай» без требования ИБ — лишняя сложность.

**sessionAffinity: None** — дефолт чарта. Sticky не требуется: токены живут у клиента.

**replicas: 3** — пример golden-path, не расчёт под вашу нагрузку. Минимум прода: ≥ 2 для rolling update.

**`automountServiceAccountToken`** — в чарте по умолчанию true; для плагина Kubernetes может быть нужно. Иначе сузить RBAC SA. `skipTLSVerify: true` пишет warning в 1.54 — в проде не оставлять.

**OpenTelemetry** — метрики/трейсы golden-path. Не встроенный `/metrics`.

**isolated-vm** — нативный модуль scaffolder. Лимиты памяти с запасом под Node heap.

**emptyDir** — том жизни пода. Не хранить TechDocs и SQLite, если от этого зависит прод.

**SoT клиентских ПДн** — не Catalog. Catalog не ACID-модель ПДн, не ищет по ИНН терабайты.

**Red Hat Developer Hub / SaaS** — другие дистрибутивы. Файл — self-hosted 1.54.0 из create-app.

**Вариант A** — один Kubernetes на 3 ЦОДа, один Deployment, одна PostgreSQL.

**Вариант B1** — активный портал в одном ЦОДе.

**Вариант B2** — поды в нескольких кластерах на **одну** PostgreSQL. Решение по БД, не по Node. Риск двух processing loop.

**Вариант B3** — три независимых app. Три каталога. Обычно ошибка.

Источники формулировок: глоссарий и тело `Backstage.md`. Новых порогов RTT и размеров диска здесь нет.
