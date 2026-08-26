# OpenSearch Dashboards 3.8.0 — развёртывание и настройка

Версия ПО: **OpenSearch Dashboards 3.8.0** (тот же релиз, что кластер OpenSearch 3.8.0 от 4 августа 2026).  
Документация: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/  
Образ: `opensearchproject/opensearch-dashboards:3.8.0`.  
На Kubernetes официальные пути: секция `spec.dashboards` в **OpenSearch Kubernetes Operator** и отдельно Helm-чарт `opensearch/opensearch-dashboards` (appVersion 3.8.0, репозиторий `https://opensearch-project.github.io/helm-charts/`). Ниже для прода целевой путь — **оператор вместе с кластером OpenSearch**; Helm-чарт — запасной, более «ручной».

Этот текст — не мануал «скопируй `opensearch_dashboards.yml`», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

Dashboards — **другое ПО**, чем OpenSearch: это веб-UI и прокси к REST 9200. Кластер поиска живёт без UI; UI без живого кластера бесполезен. Документ по самому кластеру: `Out/Платформенная инфра/OpenSearch.md`. Здесь — только слой Dashboards.

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **OpenSearch Dashboards (OSD)** | Веб-приложение (Node.js): Discover, визуализации, дашборды, часть админки Security, Index Management и другие плагины. Порт **5601**. |
| **opensearch_dashboards.yml** | Главный конфиг процесса. Без него контейнер возьмёт дефолты образа (часто demo). |
| **Saved objects** | То, что вы «нарисовали» в UI: index patterns, visualizations, dashboards, saved searches. Живут **не на диске пода**, а в индексах OpenSearch. |
| **`.kibana` / `.opensearch_dashboards`** | Индекс(ы), куда OSD пишет saved objects. Имя должно совпасть с `kibana.index` в Security config (`index: '.kibana'` по умолчанию). |
| **Tenant (тенант)** | «Папка» saved objects внутри одного кластера. Global — общий, Private — личный на пользователя, плюс именованные. Security plugin для каждого тенанта плодит **отдельный** индекс вида `.kibana_<hash>_<имя>`. |
| **kibanaserver** | Служебный пользователь, которым **сам процесс** OSD ходит в OpenSearch обслуживать свои индексы. Это **не** учётка аналитика. Дефолт demo: пароль `kibanaserver`. Должен быть в роли `kibana_server`. |
| **kibana_server (роль)** | Reserved-роль с правами на `.kibana*`, `.opensearch_dashboards*`, `.tasks` и служебные шаблоны. Если `server_username` ≠ `kibanaserver`, этого пользователя **надо явно** привязать к роли. |
| **Security Dashboards plugin** | Плагин UI к Security plugin кластера: логин, роли, тенанты, cookie-сессия. Ключи конфига — `opensearch_security.*`. Без этого плагина ключи вроде `opensearch_security.cookie.secure` **не существуют**, процесс падает с `Unknown configuration key`. В стандартном дистрибутиве плагин есть. |
| **Cookie-сессия** | После логина OSD кладёт в браузер cookie `security_authentication` (имя по умолчанию). Сессия **не** хранится в Redis и **не** в отдельной БД OSD. Любая реплика OSD может принять cookie, **если** у всех одинаковый `opensearch_security.cookie.password`. |
| **cookie.password** | Секрет шифрования cookie. В схеме плагина: **минимум 32 символа**, дефолт **`security_cookie_default_password`** (ровно 32, общеизвестен). Кто знает дефолт — может подделать сессию. |
| **cookie.secure** | Флаг «cookie только по HTTPS». Дефолт плагина **`false`**. Документация TLS: если включили HTTPS у OSD — ставить **`true`**. |
| **session.ttl / cookie.ttl** | Срок жизни сессии и cookie в **миллисекундах**. Дефолт обоих **3600000** (1 час). `session.keepalive` (дефолт `true`) продлевает TTL при активности. |
| **opensearch.hosts** | Список URL **нод одного и того же** кластера OpenSearch (fallback, если одна не отвечает). Это **не** «два разных кластера в одном UI». |
| **Multiple data sources** | Отдельная фича: `data_source.enabled: true`. Тогда в UI можно добавить **другие** кластеры/источники. Выключено по умолчанию. Timeline-визуализации с этим режимом **не** поддерживаются. |
| **SSO (SAML / OpenID Connect)** | Вход через корпоративный IdP. Настраивается **в двух местах**: `config.yml` Security plugin на кластере **и** `opensearch_dashboards.yml`. Для SAML ACS-URL надо в `server.xsrf.allowlist`. |
| **multiple_auth** | Экран логина с несколькими кнопками. Допустимые типы в массиве: `basicauth`, `openid`, `saml`. Если типов больше одного — **`basicauth` обязателен** в массиве. |
| **default_redirect_auth_type** | Авторедирект на SAML/OIDC, минуя форму. Обход для админа: `?auto_login=false`. |
| **Proxy auth** | OSD доверяет заголовкам с reverse-proxy (IdP уже аутентифицировал). Опасно, если 5601 доступен в обход proxy. |
| **server.ssl.*** | TLS **браузер ↔ OSD** (порт 5601). По умолчанию для простоты старта — **HTTP**. |
| **opensearch.ssl.*** | TLS **OSD ↔ OpenSearch** (9200). `verificationMode`: `full` (сертификат + hostname, дефолт и рекомендация), `certificate`, `none` (как в docker-примерах; для прода нет). |
| **alwaysPresentCertificate** | OSD предъявляет клиентский сертификат на 9200 (mTLS). Дефолт `false`. |
| **server.basePath / rewriteBasePath** | OSD за reverse-proxy на подпути (`/logs`). У оператора поле `basePath` само проставляет `rewriteBasePath: true`. Ingress должен совпадать. |
| **NODE_OPTIONS / max-old-space-size** | Лимит кучи **V8** (это не JVM). Helm-чарт в комментарии приводит пример `--max-old-space-size=1800`. Без этого Node может упереться в дефолт кучи и умереть OOM при тяжёлых сохранённых объектах, не дойдя до лимита Kubernetes. |
| **Deployment vs StatefulSet** | OSD — **Deployment**: поды взаимозаменяемы. PVC под данные поиска **не нужен**. |
| **PDB** | PodDisruptionBudget: Kubernetes не имеет права снять сразу все реплики OSD при drain ноды. |
| **Anti-affinity / topology spread** | Не складывать все реплики OSD в одну зону/узел. |
| **updateStrategy: Recreate** | Дефолт **официального Helm-чарта**: при выкате сначала убиваются все поды, потом встают новые. Это **простой UI**. Для HA его меняют на `RollingUpdate`. |
| **Ingress / Service** | Точка входа пользователей на 5601. TLS для людей обычно на Ingress; внутри кластера оператор может сгенерировать свой cert. |

---

## Основные элементы системы и зависимости

### Что входит в «поставить Dashboards» (это не один файл)

| Элемент | Зачем | Как масштабируется |
|---|---|---|
| **Процесс OSD** | UI + прокси запросов пользователя в OpenSearch | Горизонтально: реплики Deployment. Вертикально: CPU/RAM + `NODE_OPTIONS` |
| **Индексы `.kibana*`** | Saved objects и тенанты | Это данные **кластера OpenSearch**. Replica/awareness/snapshot — правила OpenSearch, не OSD |
| **Пользователь `kibanaserver`** | Обслуживание индексов OSD | Один на кластер; пароль в Secret, не в Git |
| **Security plugin на кластере** | Кто вошёл, какие индексы видит | Без него OSD — открытое окно в 9200 |
| **Плагины OSD** (Security, ISM UI, Alerting, Observability, …) | Экраны поверх возможностей кластера | Версия плагина = версия OSD = версия OpenSearch (**major.minor.patch**) |
| **IdP (опционально)** | SSO людей | HA IdP — отдельная задача; падение IdP = нельзя войти (если нет `basicauth` в запасе) |
| **Ingress / LB** | Доступ людей и распределение по репликам | Минимум 2 реплики OSD + живой LB, иначе SPOF на входе |

OSD **не** содержит: шину событий, озеро эталона, Camunda, Wazuh, бэкап кластера, кворум cluster manager.

### Официальные порты и контракт сети

| Порт | Назначение |
|---|---|
| **5601/TCP** | UI и API самого Dashboards (браузер, иногда автоматизация saved objects) |
| **9200/TCP** | Не слушает OSD. Это **исходящие** запросы OSD → OpenSearch (и запросы, которые OSD проксирует от имени пользователя) |

UDP нет. Порт **9600** — Performance Analyzer **ноды OpenSearch**, не Dashboards.  
Helm-чарт дополнительно описывает `metricsPort: 9601` и ServiceMonitor на `/_prometheus/metrics` — это **опция чарта**, не строка «обязательных портов» в install-гайде OSD. В проде включать только если реально подняли экспорт и ограничили, кто скрейпит.

Браузеры, которые вендор **заявляет**: Chrome, Firefox, Safari, Edge (Chromium). Internet Explorer и Microsoft Edge Legacy — **нет**.

### Чего в Dashboards нет (частая путаница)

| Нужно системе | Это не Dashboards | Зачем помнить |
|---|---|---|
| Хранение логов / поиск / шарды | OpenSearch | Падение OSD не удаляет индекс. Падение OpenSearch делает OSD красным экраном |
| Источник истины клиентских данных | Озеро / СУБД | Карточка в Discover — проекция для поиска, не SoT |
| SIEM | Wazuh (у вас отдельный документ) | Подключать Wazuh indexer как data source в «боевой» OSD = смешать роли ИБ и аналитиков в одном UI |
| Grafana / PromQL | Другое ПО | OSD рисует OpenSearch DSL/PPL, не заменяет Prometheus |
| Кластерная сессионная БД | Нет | Сессия в cookie. Redis «для HA OSD» официально не требуется и не описан как компонент |
| ГОСТ TLS / СКЗИ | Не заявлено | Штатный TLS Node.js/JDK — не криптография КИИ, если ИБ её потребует |
| Гарантия «N одновременных аналитиков» | Нет замеров | Нагрузки у вас нет. Масштаб OSD ≠ масштаб data-нод |
| Бэкап дашбордов отдельно от кластера | Snapshot `.kibana*` **внутри OpenSearch** | Удалили индекс / нет snapshot — визуализации нет, даже если поды OSD живы |

### Зависимости окружения (обязательны)

- **Живой OpenSearch той же линии 3.8.0.** Документация плагинов: OSD 2.3.0 работает с OpenSearch 2.3.0; плагины — major.minor.patch. Смешивать 3.8 Dashboards с 2.x кластером — неподдерживаемая лотерея.
- **Node.js.** В пакете OSD 3.5+ лежит **Node.js 22**. Свой runtime — через `NODE_OSD_HOME` / `NODE_HOME`. Для контейнера обычно не трогают.
- **Сеть OSD → 9200.** Список `opensearch.hosts` должен резолвиться на coordinating/ingest (или Service перед ними), **не** на dedicated cluster manager. HTTPS, если на кластере включён HTTP TLS (в проде — да, см. документ OpenSearch).
- **Учётка процесса** (`kibanaserver` или свой `server_username`) с ролью `kibana_server`. Без неё OSD не создаст/не починит `.kibana` и стартует «красным».
- **Заголовок `securitytenant`.** Если его нет в `opensearch.requestHeadersAllowlist` / `requestHeadersWhitelist` вместе с `Authorization`, документация: OSD стартует с **red** status.
- **Kubernetes:** Deployment, без RWO под данные поиска. Секреты: пароль kibanaserver, cookie.password, client secret OIDC, TLS key. Ingress, если люди ходят снаружи namespace.
- **PKI.** Demo/self-signed — только стенд. Для браузера удобнее валидный cert на Ingress; оператор прямо предлагает: внутренний cert оператора + публичный на Ingress.
- **Снимок `.kibana*`.** Это часть snapshot-политики кластера OpenSearch, не «галочка в OSD».

### Как Dashboards стыкуется с вашей архитектурой

```
Люди (аналитики, сопровождение интеграций, админы)
        │  HTTPS 5601  (только внутренняя сеть / SSO)
        ▼
   Ingress / Service  ──►  реплики OpenSearch Dashboards
                                │
                                │  HTTPS 9200 от имени пользователя
                                │  + служебные запросы kibanaserver
                                ▼
                     OpenSearch cluster (см. отдельный документ)
                                ▲
   Kafka / Data Prepper / микросервисы ──► пишут документы сюда, не в OSD

Озеро эталона ──остаётся SoT──►  в OSD можно *смотреть* поисковую проекцию
Wazuh indexer ──отдельный кластер──►  не этот UI
```

Честные роли OSD у вас:

1. Смотреть логи/метрики-в-индексах сервисов, Camunda, интеграционного API.
2. Искать проекцию справочников/клиентов (если вы её вообще кладёте в OpenSearch).
3. Админка Security и ISM — **только** выделенным ролям, не всем, у кого есть закладка.

Нечестная роль: «откроем 5601 в интернет с admin, чтобы руководство видело графики». Это attack surface на ПДн и ответы госслужб.

---

## Краткие вводные

### Зачем вам Dashboards, если уже есть OpenSearch

Кластер даёт API 9200. Люди так не работают. Dashboards закрывает Discover, дашборды, часть эксплуатации (ISM, алерты плагинов). Он **не** ускоряет индексацию Kafka и **не** заменяет кворум manager.

Падение OSD = «не видим картинки». Поиск API и запись ingest при этом могут быть живы. Это важно для честного SLO: «поиск 99.9%» ≠ «картинка в браузере 99.9%».

### Как устроена отказоустойчивость (идея, не магия)

OSD **не выбирает кворум**. Отказоустойчивость — это «несколько одинаковых процессов + общие saved objects в OpenSearch + одинаковый секрет cookie».

| Что падает | Что происходит |
|---|---|
| Один под OSD из ≥2 за LB | LB уводит трафик. Сессия в cookie — пользователь не обязан перелогиниваться, **если** cookie.password одинаковый |
| Единственный под OSD | Нет UI. Кластер OpenSearch может быть green |
| Все поды OSD в одной зоне, зона умерла | Нет UI, даже если кластер в других ЦОДах жив |
| OpenSearch yellow/red, нет `.kibana` primary | OSD не отрисует saved objects / не стартанёт нормально |
| OpenSearch недоступен с подов OSD (сеть 9200) | UI «красный», люди думают «упал поиск», хотя data-ноды живы в другом ЦОДе |
| Сменили cookie.password на части реплик | Часть подов не принимает чужие cookie → «то логинит, то выкидывает» |
| Helm `updateStrategy: Recreate` | На время выката **ноль** подов — простой UI гарантирован чартом |
| Нет snapshot `.kibana*` | После `DELETE .kibana*` или гибели всех копий визуализации нет |

Три ЦОДа **сами по себе** OSD не спасают. Спасает раскладка реплик по зонам **и** то, что 9200 с этих реплик всё ещё достаёт кластер (см. stretch vs «мозг в одном ЦОДе» в документе OpenSearch).

### Как устроено масштабирование

Две разные оси — их путают чаще всего.

1. **Больше одновременных людей / вкладок** → больше **реплик OSD** (и CPU/RAM каждой). Это горизонталь UI.
2. **Тяжелее Discover/aggregations / больше данных** → это **data/coordinating OpenSearch**, не «добавь пятый Dashboards». OSD только проксирует запрос.

Ориентиры, которые **есть** у проекта, и которые **нельзя** принять за смету вашего прода:

- Helm README: для деплоя рекомендуется **8 ГиБ** RAM **доступно**, минимум **4 ГиБ**, иначе «deployment is expected to fail». Это про запас хоста/ноды, не про ваш QPS.
- Дефолт **values.yaml того же чарта**: `resources.requests/limits` = **100m CPU / 512M RAM**, `replicaCount: 1`. Это стендовый минимум, он **противоречит** абзацу «4–8 ГиБ» — в прод 512M не копировать.
- Куча V8 задаётся отдельно (`NODE_OPTIONS=--max-old-space-size=…`). Лимит Kubernetes должен быть **выше** этой кучи, иначе OOM killer, а не «Node сам пожмёт».
- HPA в чарте есть (`autoscaling.enabled`, CPU/memory 80%), нужен Metrics Server. Это не замена пониманию, что узкое место часто 9200, а не 5601.
- Числа «терабайты озера» на OSD **не влияют**, пока вы не открыли Discover по этим терабайтам: тогда страдает кластер.

Нет официальной формулы «N аналитиков = M реплик».

### Безопасность самого Dashboards

OSD видит то же, что видит пользователь через Security plugin: логи интеграций, клиентский поиск, карты индексов. Украденная сессия `admin` или demo `kibanaserver` = работа от чужого имени в UI.

Официально / из схемы плагина:

- Старт «для простоты» — **HTTP** на 5601 (`server.ssl.enabled` в docker-примерах `false`, `opensearch.ssl.verificationMode: none`).
- Включить HTTPS: `server.ssl.enabled: true` + cert/key; при этом `opensearch_security.cookie.secure: true`.
- Дефолт `cookie.password` = `security_cookie_default_password` — **сменить** на свой ≥32 символов, один на все реплики, хранить в Secret.
- Demo-пароль `kibanaserver`/`kibanaserver` в docker-гайде. Оператор, если секрет не дали, **генерирует случайный** пароль в `<cluster>-dashboards-password` — это лучше demo, но Secret всё равно надо ротировать и не светить в логи.
- SSO: SAML/OIDC в `config.yml` **и** в OSD. Для API 9200 без браузера вендор по SAML рекомендует **ещё один** domain (internal/LDAP/basic), иначе автоматизация разъедется с людьми.
- `auth.anonymous_auth_enabled` дефолт **false**. Включать анонима в проде = публичное чтение того, что разрешите роли anonymous.
- Публичный 5601 без SSO и без сети = инцидент, не «витрина для руководства».

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Берём self-hosted OSD 3.8.0** к self-hosted OpenSearch 3.8.0, не Amazon OpenSearch Service (там другой URL, IAM, нет ваших манифестов).
2. **Прод в Kubernetes**, OSD ставится **оператором** (`spec.dashboards.enable: true`, `version: 3.8.0` = версия кластера). Чарты оператора пинить тегом релиза. Helm `opensearch-dashboards` — запасной путь, если оператор Dashboards не используете.
3. **Один логический UI на один поисковый кластер** (тот, что в `OpenSearch.md`). Wazuh indexer — **отдельный** UI/кластер, не data source «для удобства».
4. **Три ЦОДа = три зоны Kubernetes** (`topology.kubernetes.io/zone` или ваш аналог). Реплики OSD размазываем по зонам. Если «три ЦОДа» без отдельных зон в scheduler — anti-affinity не сработает.
5. **Цель отказа для UI: пережить 1 ЦОД** (как у кластера). Два мёртвых ЦОДа из трёх при по одной реплике OSD в зоне = один живой под; если живой ЦОД без 9200 до кластера — UI всё равно мёртв.
6. **Нагрузки (число аналитиков, тяжесть Discover) нет** — нет цифры реплик. Есть правило «не меньше 2» и сигналы (latency 5601, heap Node, latency 9200, rejected на кластере).
7. **Формального SLO на UI нет.** Green OpenSearch ≠ «дашборд открылся за 2 секунды».
8. **Люди в проде входят через SSO** (OIDC или SAML) + запасной `basicauth` для аварийного входа (`?auto_login=false`, если включён redirect). Общая учётка `admin` в UI запрещена.
9. **Корпоративный PKI / Ingress-cert есть или будет.** HTTP 5601 в проде только за mesh/Ingress, который уже терминалит TLS, и тогда cookie.secure/заголовки надо согласовать (иначе cookie не выставится).
10. **Шифрование канала — обычный TLS.** ГОСТ/СКЗИ не озвучены.
11. **Тестовый стенд изолирован.** Demo-пароли, `verificationMode: none`, одна реплика, HTTP 5601 — только там.
12. **Saved objects — часть данных кластера.** Их судьба = replica/awareness/snapshot OpenSearch. Отдельного «бэкапа Dashboards» нет.
13. **Лицензия Apache 2.0** у дистрибутива. Сторонние плагины OSD (`plugins.installList` в Helm) проверять отдельно.

---

## Критически важные условия, которых нет в исходном контексте

Их нужно закрыть **до** «выкатили 5601 и скинули ссылку в чат».

| Пробел | Почему это ломает решение |
|---|---|
| **Кто пользователь UI** (10 админов? 300 аналитиков? подрядчики?) | От этого зависят реплики, SSO, тенанты, нужен ли вообще публичный Ingress |
| **Куда смотрит 5601** (только VPN/internal DNS? есть ли WAF?) | Публичный Ingress с demo cookie.password — инцидент. SafeLine/WAF у вас в другом документе — его надо явно включить в цепочку, если UI торчит наружу |
| **Есть ли IdP (Keycloak / AD FS / иное)** | Без IdP вы либо раздадите internal users (плохо масштабируется и аудитится), либо оставите `admin` |
| **OSD в том же Kubernetes, что OpenSearch, или отдельно** | От этого зависит DNS `opensearch.hosts`, NetworkPolicy, и переживает ли UI падение namespace кластера |
| **Вариант A/B кластера OpenSearch** (stretch на 3 ЦОДа vs мозг в одном vs CCR) | Реплики OSD в «чужом» ЦОДе без 9200 до leader бесполезны. UI latency = RTT до 9200 |
| **Нужны ли multiple data sources** | Второй кластер в одном UI — удобно и опасно (RBAC двух Security realm в одной сессии оператора) |
| **Состав тенантов** (один Global на всех vs тенант на команду) | Каждый private tenant = ещё индекс `.kibana_*` в cluster state. Сотни пользователей × private = сюрприз для heap manager |
| **Кто владеет saved objects** (Git/NDJSON vs «накликали в UI») | Без экспорта после аварии `.kibana` вы восстанавливаете только то, что попало в snapshot |
| **Согласование TLS: Ingress vs `server.ssl` vs cookie.secure** | Классика: Ingress HTTPS, OSD HTTP внутри, `cookie.secure: true` → логин «не работает». Или наоборот, cookie без Secure на HTTP |
| **Совместимость оператора: поле `dashboards` вашей версии CR** | Документация оператора в примерах пишет `version: 3.0.0`. Перед продом — прогон **3.8.0** на стенде |

---

## Краткое описание ключевых этапов и элементов развертывания

Общий каркас (и тест, и прод):

1. Зафиксировать **OSD 3.8.0 = OpenSearch 3.8.0**. Не смешивать major/minor/patch плагинов.
2. Сначала **здоровый кластер** (хотя бы green/yellow осознанный), пользователь `kibanaserver`, TLS 9200 как решили для кластера.
3. Выпустить **свои** пароли (kibanaserver, cookie.password ≥32) **до** боевых пользователей. Не оставлять demo.
4. Поднять OSD, проверить логин, создание index pattern, что документ **виден**.
5. Включить снимок, покрывающий `.kibana*` (и прогнать restore — это процедура кластера).
6. Метрики: процесс жив (probe 5601), latency UI, ошибки 5xx Ingress, куча Node, с другой стороны — здоровье 9200.
7. Только потом — SSO, тенанты, multiple data sources, Ingress «для всех».

Дальше — два режима.

---

### 1 инстанс: тестовый стенд, 1 ЦОД, без нагрузки

**Цель стенда:** чтобы разработчики открыли Discover, настроили mapping/index pattern, потыкали Security UI. **Не** цель: доказать отказ ЦОДа и N одновременных сессий.

Официальный минимальный путь (Docker, после уже запущенного single-node OpenSearch на сети `os-net`):

1. Файл `opensearch_dashboards.yml` как в install-гайде: `server.ssl.enabled: false`, `opensearch.ssl.verificationMode: none`, `opensearch.username/password: kibanaserver`, hosts на имя контейнера OpenSearch.
2. Запуск:

```text
docker run -d --name osd \
  --network os-net \
  -p 5601:5601 \
  -v ./opensearch_dashboards.yml:/usr/share/opensearch-dashboards/config/opensearch_dashboards.yml \
  opensearchproject/opensearch-dashboards:3.8.0
```

Вендор в том же гайде для OpenSearch пишет `opensearchproject/opensearch:latest` — **не копируйте latest в прод и лучше не на стенд**: тег **3.8.0**, как у кластера.

Альтернатива: docker compose из install-гайда OpenSearch (2 ноды + Dashboards). Примеры compose вендор помечает как **не для production**.

На Kubernetes: `spec.dashboards.enable: true`, `replicas: 1`, HTTP, demo/self-signed допустимы **в изолированном namespace**. Либо Helm с `replicaCount: 1`. Не копировать этот values в прод.

#### Что сознательно упрощаем

| Параметр | На тесте | Почему можно |
|---|---|---|
| Реплики OSD | 1 | Нет требования пережить выкат |
| TLS 5601 | HTTP | Иначе команда утонет в cert раньше, чем в Discover |
| `verificationMode` | `none` как в docker-гайде | Self-signed кластера |
| Пароли | demo `kibanaserver` **только в изолированной сети** | С 2.12 кластер всё равно требует свой admin-пароль; kibanaserver в гайде ещё demo |
| cookie.password | дефолт | Одна реплика, нет интернета |
| SSO / тенанты | можно не трогать | Сначала увидеть свои JSON |
| CPU/RAM | 512M как дефолт Helm | Не цель — ёмкость UI |
| Snapshot `.kibana*` | можно отложить на день | Нельзя забыть на препроде |

#### Чего на тесте **не** стоит упрощать

- Версия **3.8.0** = кластеру.
- Понимание: картинки живут в `.kibana*`, а не в контейнере OSD. Удалили volume OSD — дашборды на месте; удалили индекс — нет.
- Привычку **не** публиковать 5601 в интернет «на минутку».
- Запомнить: `verificationMode: none` и Recreate из Helm **не** есть «почти прод».

#### Сильные стороны такой схемы

- Совпадает с официальным Docker-гайдом; поднимается за часы после кластера.
- Дешево учит index patterns на *ваших* полях из Kafka/интеграций.
- Один процесс — меньше движущихся частей.

#### Слабые стороны (обязательно понимать)

- Нет модели отказа UI, нет доказательства, что cookie разъедется по репликам.
- Demo `kibanaserver` и дефолтный cookie.password приучают «так и оставим».
- Успешный клик на одном поде **не** доказывает Ingress+SSO+три зоны.

Практическая рекомендация: препрод = маленькая **копия прод-топологии OSD** (≥2 реплики, свои пароли, TLS на Ingress, cookie.secure согласован, RollingUpdate, snapshot `.kibana*`), даже без боевого числа пользователей.

---

### Прод: 3 ЦОДа, нагрузка

Цифр «реплик под терабайты озера» **нет** — нагрузки нет. Ниже правила, без которых экземпляр не считается готовым.

#### Шаг 0. Макроархитектура (сделать до установки)

OSD ходит на **9200**. Топология UI следует за топологией кластера из документа OpenSearch.

**Вариант A — stretch-кластер OpenSearch на 3 зонах, один OSD Deployment.**

- `replicas ≥ 2` (лучше **3**, по одной в зоне, если цель — пережить 1 ЦОД без потери UI).
- Topology spread / anti-affinity по зоне. PDB: не снять все реплики сразу (`minAvailable: 1` как минимум при 2 репликах; при 3 — разумнее `minAvailable: 2`).
- `opensearch.hosts`: Service кластера (coordinating), несколько URL **того же** кластера допустимы как fallback.
- Ingress/LB в каждой зоне или один Ingress, который переживает 1 ЦОД (это уже ваш HAProxy/контроллер, не OSD).
- Одинаковый `cookie.password` на всех подах (один Secret).

**Вариант B1 — кластер OpenSearch только в одном ЦОДе.**

- OSD имеет смысл держать **рядом с 9200** (тот же ЦОД): иначе каждый клик Discover едет через город.
- Реплики OSD в «пустых» ЦОДах дают ложное чувство HA: UI поднимется, а 9200 мёртв вместе с кластером.
- Честный DR UI = DR кластера (restore snapshot, включая `.kibana*`) + выкатить OSD к новому 9200.

**Вариант B2 — три кластера + CCR.**

- На leader — OSD для записи/админки.
- На follower — отдельный OSD **или** multiple data sources с read-only ожиданием. Follower read-only: админка Security «на follower как на leader» — ошибка эксплуатации.
- `opensearch.hosts` **не** смешивает URL leader и follower как «один кластер».

**Пока не выбран A/B для OpenSearch, нельзя честно разложить OSD.** Дальше — общее для A как целевого.

##### Сильные / слабые стороны (A, OSD на stretch)

| Сильное | Слабое |
|---|---|
| Один URL, один набор saved objects, один RBAC | Latency UI = худший 9200 из тех, куда попал запрос |
| Переживание 1 зоны на уровне подов OSD | Если stretch OpenSearch уже «плывёт» по RTT, OSD это **усиливает** жалобами людей |
| Cookie-сессия не требует Redis | Утечка cookie.password = подделка любой сессии |
| | Ошибка mapping/дашборда сразу у всех трёх ЦОДов |

##### Сильные / слабые стороны (B1)

| Сильное | Слабое |
|---|---|
| Короткий путь до 9200 | Падение ЦОДа кластера = нет UI ни у кого |
| Проще cookie/Ingress | Реплики OSD в других ЦОДах почти не помогают |

##### Сильные / слабые стороны (B2 / data sources)

| Сильное | Слабое |
|---|---|
| Можно смотреть follower «рядом» | Два Security realm, два набора `.kibana`, легко выдать лишние права |
| Официальная фича `data_source.enabled` | Timeline не работает; часть плагинов ведёт себя иначе |
| | Это не failover записи и не «один кластер» |

##### Слабое место: RTT на 9200, не на 5601

Между ЦОДами не измеряли даже 9300. Для OSD дополнительно важен **9200 с подов OSD до coordinating**. Высокий RTT → «тормозит Discover», таймауты (`opensearch.requestTimeout` в yml, если зададите), ложный «у нас упал OpenSearch». Мерить тот путь, которым ходит OSD, не ping до Ingress.

#### Реплики, выкат, ресурсы

- Минимум **2** реплики в проде; для 3 зон цель отказа 1 ЦОД — **3**.
- Оператор: `spec.dashboards.replicas`. Helm: сменить `replicaCount` **и** `updateStrategy.type` с **`Recreate` на `RollingUpdate`**. Иначе HA на бумаге, простой на каждом выкате.
- Probes в чарте — TCP 5601 (startup/liveness/readiness). Это «порт открыт», не «логин и `.kibana` зелёные». Дополнительно смотреть статус OSD и `_cluster/health`.
- Память: не оставлять 512M. Выставить request/limit по стенду; `max-old-space-size` **ниже** limit. Конкретного «официального Xmx для OSD» в install-гайде нет — только эмпирически и абзац Helm про 4–8 ГиБ **доступно**.
- HPA включать только после появления профиля: иначе реплики вырастут из-за тяжёлого aggregation, который надо лечить шардами OpenSearch.

#### Saved objects, тенанты, бэкап

- Snapshot кластера должен включать паттерн **`.kibana*`** (документация multi-tenancy прямо: бэкап — snapshot tenant indexes).
- Проверять **restore** этих индексов, не только «под OSD жив».
- Multi-tenancy по умолчанию в `config.yml` Security **включена** (`multitenancy_enabled: true`). На стороне OSD docker-гайд тоже ставит `opensearch_security.multitenancy.enabled: true`. Схема плагина в коде имеет свой дефолт `false` — **не полагаться на дефолт**, задать явно и согласовать с `config.yml`.
- Private tenant на каждого пользователя плодит индексы. Если пользователей сотни, а тенанты не нужны — сознательно выключить private/global по политике ИБ, а не «как в demo».
- Боевые дашборды желательно уметь накатить из **NDJSON** (Saved objects import), иначе единственная копия — в кластере.

#### Безопасность прода

| Слой | Правило |
|---|---|
| Сеть | 5601 не в интернет без SSO и периметра. NetworkPolicy: Ingress → OSD; OSD → 9200 coordinating; метрики — только Prometheus |
| TLS браузер | HTTPS на Ingress (доверенный CA). Если TLS терминалится на OSD — `server.ssl.enabled: true` |
| Cookie | Свой `cookie.password` ≥32; `cookie.secure: true` если браузер ходит HTTPS; одинаковый секрет на репликах |
| OSD ↔ OpenSearch | `verificationMode: full`, свои CA; demo `none` запрещён |
| kibanaserver | Не demo-пароль. Оператор: свой Secret или сгенерированный, ротация, не логировать. Пользователей UI этим логином **не** пускать |
| Люди | SSO (OIDC/SAML). Backend roles → FGAC индексов. `admin` только break-glass |
| Запасной вход | `basicauth` в multiple_auth + `?auto_login=false`, если включён redirect. Иначе при падении IdP нет даже админки |
| SAML | ACS в IdP **и** в `server.xsrf.allowlist` (`/_opendistro/_security/saml/acs`). Несколько реплик: ACS URL должен быть **стабильным** именем LB, не Pod IP |
| OIDC | `client_secret` из Secret, в yml через `${ENV}` (оператор это документирует). Смена секрета = рестарт подов OSD |
| Proxy auth | Только если OSD **недоступен** в обход proxy. Иначе заголовки можно подделать |
| Аноним | Не включать, пока ИБ явно не сказала «публичный read-only дашборд» |
| Секреты | Не в Git, не в ConfigMap plaintext |
| Аудит | Аудит Security plugin на кластере; отдельно — access log Ingress |
| At-rest | Диск OSD почти пустой; важны PVC OpenSearch и шифрование snapshot |

##### Сильные / слабые стороны (TLS + SSO + cookie-сессия)

| Сильное | Слабое |
|---|---|
| Нет отдельного Redis для сессий — меньше SPOF | Дефолтный cookie.password общеизвестен; забыли сменить = подделка сессий |
| Горизонталь OSD дешёвая | Cookie уезжает с каждым запросом; крупные SAML/OIDC assertion режут на extra cookies (документировано) |
| FGAC как у API 9200 | Ошибка роли выглядит как «пустой Discover»; нужна процедура выдачи |
| Оператор генерит пароль kibanaserver | Secret надо включить в эксплуатацию (кто знает пароль, как ротировать) |
| | Не ГОСТ; mTLS браузера (`server.ssl.clientAuthentication: required`) тяжелее SSO и редко нужен |

#### Kubernetes-специфика прода

- OSD **рядом** с кластером в том же операторном CR (`dashboards.enable: true`), чтобы не разъехались версии и TLS.
- `version` OSD = version OpenSearch в CR.
- TLS секция `spec.dashboards.tls`: для людей лучше валидный cert на Ingress, внутри — generate или свой Secret (`tls.crt`/`tls.key`).
- `additionalConfig` — любой ключ yml; невалидный ключ = OSD **не стартует**.
- Секреты OIDC: env + `${VAR}` в additionalConfig; смена Secret **без** рестарта подов конфиг не подхватит (документация оператора).
- Не монтировать данные поиска. `emptyDir` для OSD допустим (кэш Node), в отличие от data-нод OpenSearch.
- ResourceQuota: не поставить limit = heap V8.
- Не отдавать Service OSD типом NodePort/LoadBalancer в мир «чтобы проверить».

#### Порядок вывода в прод (этапы, не команда за командой)

1. Иметь выбранную схему OpenSearch (A/B) и рабочий 9200 с TLS.
2. Создать Secret: kibanaserver, cookie.password, OIDC/SAML secrets.
3. Включить `dashboards` в CR: версия 3.8.0, replicas ≥ 2, topology spread, PDB, RollingUpdate (если Helm).
4. Согласовать HTTPS Ingress и `cookie.secure`.
5. Проверить логин, index pattern, что падение **одного** пода не разлогинивает (cookie общий) и LB не 502.
6. Убедиться, что `.kibana*` в политике snapshot и restore прогнан.
7. Подключить SSO на препроде, break-glass `basicauth`.
8. Прогон «зона OSD недоступна»: UI жив с других зон; «кластер 9200 недоступен» — честно ожидаем красный UI.
9. Нагрузить хотя бы синтетикой числом вкладок. Смотреть heap Node **и** rejected/latency OpenSearch. По результату — реплики OSD vs ноды кластера.
10. Запретить demo-пароли в этом namespace политикой (Kyverno/OPA — уже ваша платформа).

Без пунктов 5 и 8 у вас нет отказоустойчивости UI, есть надежда.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | Не требуется | ≥2 (лучше 3) реплики в разных зонах; PDB; RollingUpdate; cookie.password общий; цель — 1 ЦОД; UI переживает смерть пода, **не** смерть OpenSearch |
| Производительность / масштаб | Не требуется | Реплики от числа сессий; куча V8 < limit; узкое место Discover — кластер; HPA только с профилем |
| Безопасность | HTTP+demo только в изоляции | Нет demo kibanaserver/cookie; HTTPS до браузера; `verificationMode: full`; SSO; 5601 не в интернет; `.kibana*` в snapshot |

**Не готов к проду**, если: одна реплика, Helm `Recreate`, `verificationMode: none`, `cookie.password` дефолтный, пароль `kibanaserver`/`kibanaserver`, 5601 опубликован, версии OSD ≠ OpenSearch, все поды в одной зоне, нет snapshot `.kibana*`, SSO нет и все сидят под `admin`, Wazuh и бизнес-поиск в одном OSD, схема stretch OpenSearch ещё не выбрана, а OSD уже размазан «для HA».

---

## Источники (чтобы не принимать на веру)

- Установка OSD, браузеры, Node.js 22 в 3.5+: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/
- Docker, demo yml, `kibanaserver`, `verificationMode: none`, HTTP 5601: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/
- TLS OSD (`server.ssl.*`, `opensearch.ssl.*`, `cookie.secure`, session ttl): https://docs.opensearch.org/latest/install-and-configure/install-dashboards/tls/
- Совместимость плагинов OSD = версия OpenSearch: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/plugins/
- Helm OSD: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/helm/  
  values (replicaCount 1, Recreate, 100m/512M, NODE_OPTIONS, metrics 9601): https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/values.yaml  
  README чарта (4–8 ГиБ available): https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/README.md
- Оператор, `spec.dashboards`, TLS, basePath, secrets: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/operator/operator-dashboards-config/  
  пароль kibanaserver: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/operator/operator-security/
- Multi-tenancy, `.kibana*`, роль `kibana_server`, snapshot `.kibana*`: https://docs.opensearch.org/latest/security/multi-tenancy/multi-tenancy-config/
- Multiple auth / SSO-экран: https://docs.opensearch.org/latest/security/configuration/multi-auth/
- SAML + `server.xsrf.allowlist`: https://docs.opensearch.org/latest/security/authentication-backends/saml/
- OpenID Connect: https://docs.opensearch.org/latest/security/authentication-backends/openid-connect/
- Multiple data sources (`data_source.enabled`, ограничения): https://docs.opensearch.org/latest/dashboards/management/multi-data-sources/
- Схема cookie.password дефолт и minLength 32 (исходник плагина): https://github.com/opensearch-project/security-dashboards-plugin/blob/main/server/index.ts
- `opensearch.hosts` = ноды **одного** кластера, не двух (разъяснение maintainer на форуме проекта): https://forum.opensearch.org/t/connecting-two-opensearch-instances-to-one-opensearch-dashboard/18154

Утверждения вида «N реплик хватит на терабайты» или «OSD переживёт падение OpenSearch» в документации проекта **отсутствуют** — поэтому в этом файле их нет. RTT на пути OSD→9200 надо измерить у себя. Дефолты Helm (512M, Recreate, 1 реплика) **не** являются прод-нормой, даже если чарт официальный.
