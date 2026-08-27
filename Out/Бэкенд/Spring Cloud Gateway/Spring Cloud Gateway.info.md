# Spring Cloud Gateway 5.0.3 — термины и сокращения

Словарь к файлу `Spring Cloud Gateway.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс / JVM** — запущенное Spring Boot-приложение шлюза. Несколько подов = несколько независимых JVM. Согласованность конфига — Git и объект Kubernetes с текстом YAML, не общий выбор лидера между подами. Кластера с кворумом маршрутов нет.

**Файл / конфиг** — маршруты в YAML или Java `RouteLocator`, обычно ConfigMap (запись в API Kubernetes с текстом) / Helm values из CI. Живой `POST /actuator/gateway/routes` в бою не источник истины.

**TCP / HTTP(S)** — шлюз принимает HTTP(S) и сам устанавливает исходящие HTTP-соединения к сервису-бэкенду. Это не прокси для Kafka, Postgres или gRPC Camunda `:26500`.

**Origin** — сервис, на который шлюз проксирует запрос. В проде — локальный той же площадки. «Шлюз в ЦОД-1, единственный HTTP-сервис к ведомствам только в ЦОД-3» превращает сеть в деградацию всего входа.

**ЦОД** — независимая площадка. Реплики шлюза раскладываются по зонам. Stretch «один под, но диск в другом зале» не рассматривается. Поды в других ЦОДах лидера не выбирают: городской failover — слой HAProxy/DNS/Anycast.

**RTT** — задержка до origin. На локальный сервис почти не влияет. Влияет, если шлюз ЦОД-1 ходит в сервис ЦОД-2, и если Redis лимитера «на город». Порог проектом не задан.

**TLS** — шифрование. На входе часто терминирует Ingress/HAProxy; исходящий TLS к origin / mTLS — у Netty HttpClient. `use-insecure-trust-manager: true` в проде не настройка. Штатный TLS JDK/Netty, не ГОСТ/СКЗИ.

**HA** — несколько живых подов в разных зонах + живой Ingress/HAProxy + (если лимиты общие) живой Redis. Один под на три ЦОДа отказоустойчивости не даёт. Упал под — Kubernetes поднимает другой; клиент на сломанном TCP получит обрыв — норма для HTTP-прокси.

**RPO / RTO** — у шлюза «данные» — это конфиг маршрутов и (если есть) счётчики лимитов, не клиентская база.

**SoT** — досье клиента. Шлюз не хранит карточки и не ищет по ИНН.

**Под (Pod) / Deployment** — оркестрация процесса. Не StatefulSet: стабильной личности нет. IP меняется; трафик идёт в Service/Ingress при живом readiness.

**HPA (Horizontal Pod Autoscaler)** — добавить/убрать реплики по CPU / RPS / latency. Включать после базового замера. Масштаб без потолка пула HttpClient даёт много подов, которые вместе сносят origin. Не по «терабайтам озера».

**PDB (PodDisruptionBudget)** — не выселять все реплики сразу; не допускать `replicas=0` при drain узла.

**Helm / GitOps** — поставка манифестов и YAML маршрутов из git, не ручная правка на живую.

**NetworkPolicy** — Ingress/HAProxy → шлюз; шлюз → только нужные Service; запрет шлюзу → интернет (исходящие вызовы к ведомствам делает интеграционное API). Origin принимает HTTP только из CIDR шлюза (и mesh), не NodePort «на всякий случай».

**Anti-affinity / topologySpreadConstraints** — зона = ЦОД.

**Ingress / Gateway API** — вход Kubernetes до подов шлюза. SCG — обычный Deployment+Service, не замена Ingress-объектов.

**HAProxy / городской VIP / Anycast / ADC** — край площадки. Реплики шлюза сами IP между площадками не таскают. Документ HAProxy отдельно.

**PKI** — сертификаты входа и исходящего TLS. Секреты — Secret/Vault, не Git.

**CVE-2026-47879** — SSRF и доступ к локальным файлам через gRPC-путь Gateway. Закрыт в **5.0.3**. В прод не тащить 5.0.2 и ниже этой линии без патча.

**SSRF** — шлюз ходит по адресу, который подсунул клиент (кривой маршрут, Actuator записи, фильтр из заголовка).

**ПДн / 152-ФЗ** — access log содержит URL и часто токены в query. Wiretap Netty пишет трафик в лог — в проде утечка. `httpclient.wiretap` / `httpserver.wiretap` в проде **false**.

**ГОСТ / СКЗИ / КИИ** — vanilla Gateway не закрывает.

**BOM** — bill of materials: один номер поезда зависимостей. Здесь `org.springframework.cloud:spring-cloud-dependencies:2025.1.3`. Не подтягивать случайные артефакты «как в блоге 2019».

**Release train / Oakwood / Spring Cloud 2025.1.3** — поезд библиотек. Анонс 20 августа 2026. Официально основан на **Spring Boot 4.0.8**. С **2025.1.2** Oakwood совместим также с **Spring Boot 4.1.x** — отдельное решение совместимости всех облачных модулей, не «молча latest Boot». Поддержка поезда следует за поддержкой Boot.

**JDK** — Spring Boot 4.0: минимум **Java 17**, first-class **Java 25**, до Java 25. На шлюзе цель файла — 21 или 25 LTS. Boot 4.1.x (если возьмёте): минимум 17, до Java 26 (system requirements текущей 4.1.1). Maven ≥ 3.6.3 или Gradle 8.14+ / 9.x — сверять страницу той минорной линейки Boot перед пином.

**API Gateway (шлюз)** — приложение, которое принимает HTTP(S) снаружи (или от внутреннего клиента) и проксирует запрос на нужный сервис: маршруты, заголовки, лимиты, аутентификация, обрыв при аварии бэкенда.

**Spring Cloud Gateway Server** — полноценный шлюз: отдельное Spring Boot-приложение (или встроенный модуль), которое живёт маршрутами. Не путать с Proxy Exchange.

**Proxy Exchange** — не отдельный шлюз, а объект `ProxyExchange` в уже существующем `@Controller` / WebFlux-хендлере. Проксировать из своего кода. Другая поставка, другой операционный контур.

**WebFlux** — реактивный стек Spring (Netty, неблокирующий I/O). Стартер: `spring-cloud-starter-gateway-server-webflux`. В 5.x это полнофункциональный Server. Целевой прод файла. Не класть WebFlux-шлюз во внешний Tomcat как WAR.

**WebMVC** — классический Servlet-стек (обычно Tomcat). Стартер: `spring-cloud-starter-gateway-server-webmvc`. Другой префикс настроек, другой набор фильтров. Нельзя смешать оба стартера в одном процессе. Старый артефакт `spring-cloud-starter-gateway` без `-server-webflux` из гайдов — не этот файл.

**Netty** — неблокирующий HTTP-сервер Boot для WebFlux и исходящий HttpClient к бэкенду.

**Маршрут (route)** — правило: если запрос подходит под предикаты — отправить на URI, прогнав фильтры. У маршрута есть `id` и `order` (кто победит, если подошло несколько). `fail-on-route-definition-error: true` (дефолт) — не глушить ошибки определения.

**Предикат (predicate)** — условие совпадения: путь `/api/**`, хост, метод, заголовок, вес (canary) и т.д. Несколько предикатов в YAML склеиваются через AND. Java `RouteLocatorBuilder` — когда условие сложнее AND.

**Фильтр (GatewayFilter)** — действие на этом маршруте: срезать префикс, добавить заголовок, rate limit, retry, circuit breaker. Часть логики — до вызова бэкенда, часть — после ответа.

**Глобальный фильтр (GlobalFilter)** — то же по смыслу, но на все маршруты (маршрутизация Netty, `lb://`, запись ответа).

**URI маршрута** — куда проксировать. `https://host:port` — фиксированный адрес. `lb://service-id` — через Spring Cloud LoadBalancer. `forward:/local` — внутрь этого же приложения. Путь в URI маршрута официально игнорируется (берётся путь входящего запроса + фильтры вроде StripPrefix). Не писать путь в `uri:`.

**StripPrefix / RewritePath** — срезать или заменить префикс пути перед origin. Реальные публичные пути отлаживают на стенде, иначе в проде 404.

**`lb://`** — схема «найди инстансы сервиса и выбери один». Нужен `spring-cloud-starter-loadbalancer` и DiscoveryClient (Eureka/Consul/K8s…) или явное перечисление инстансов. Без этого не «магия Kubernetes». `loadbalancer.use404` — 404 вместо 503, если инстансов нет.

**DiscoveryClient / Spring Cloud Kubernetes** — клиент реестра сервисов. На Kubernetes: либо DNS Service (`http://name.ns.svc`), либо Spring Cloud Kubernetes + RBAC на API. Смешивать Eureka, K8s и хардкод — путь к ночным 503. Если discovery нет — шлюзу не нужны list/watch всех Service кластера.

**Discovery locator** — автосоздание маршрутов вида `/имяСервиса/**` по всем сервисам из реестра. Дефолт выключен. Включить «чтобы не писать YAML» в проде = выставить в интернет все найденные Service. В допущениях прода — выкл.

**Netty HttpClient** — чем WebFlux-шлюз ходит на бэкенд. Свой пул соединений, TLS, таймауты. Это не Tomcat и не Ingress.

**Пул соединений (pool)** — сколько исходящих HTTP-коннектов к бэкендам держать. Дефолт типа пула в 5.0.3 — **`elastic`**. Режим **`fixed`** ограничивает `max-connections`, чтобы elastic не размножил коннекты до OOM/нехватки дескрипторов при шторме. `max-idle-time` / `max-life-time` — не держать вечно полумёртвые keep-alive за NAT.

**connect-timeout / response-timeout** — сколько ждать установления TCP к бэкенду / полного ответа. Глобально и на маршрут (в metadata). Дефолт connect-timeout в appendix — **30 с**, если не задан; для внутреннего Service обычно слишком много. Глобальный response-timeout — Duration; на маршруте metadata — миллисекунды. Отрицательный per-route response-timeout отключает глобальный. Для ведомств — из замера, не «как у REST 5s».

**Token bucket** — алгоритм rate limit: у ключа есть запас разрешений, каждый запрос тратит разрешения, запас пополняется со временем. Штатная Redis-реализация Gateway так устроена.

**KeyResolver** — по какому ключу считать лимит: пользователь, IP, API-ключ. Дефолт — `Principal.getName()`. Нет ключа → запрос отклоняется (ослаблять в проде вслепую нельзя). Пример из доки с query-параметром `user` — не для прода. При JWT — `sub`/client_id из уже проверенного токена, не из заголовка, которому верит клиент.

**Circuit breaker (автомат защиты)** — если бэкенд сыплет ошибками/таймаутами — перестать долбить его и сразу отдавать fallback (или 5xx). В Gateway это фильтр, реализация — Spring Cloud CircuitBreaker (обычно Resilience4j). Без него шлюз копит таймауты и жжёт пул. Spring Cloud CircuitBreaker 5.0.3 едет в том же BOM.

**Fallback** — запасной ответ/URI, когда автомат разомкнут или бэкенд мёртв. Без него клиент получает ошибку шлюза. Проверять, что fallback не отдаёт чужие данные.

**Retry** — повторить запрос к бэкенду. На GET иногда уместно. На POST в ведомство без идемпотентности = двойное списание/двойная заявка. По умолчанию на POST в Integration API — нет.

**TimeLimiter (Resilience4j)** — ограничение времени вызова. В поезде 2025.1.3 дефолт TimeLimiterConfig в factory больше не используется — не копировать старые гайды 2024 года.

**Actuator `/actuator/gateway`** — пульт маршрутов: список, создание, удаление, refresh. В 5.0.3 доступ по умолчанию выключен. `unrestricted` / `enabled=true` = можно менять маршруты на живую (классический вектор SSRF). Прод: `read-only` или не экспонировать. YAML-маршруты через DELETE не удаляются (404). `/env`, `/heapdump` не публиковать. Health в probes: не валить под из-за «Redis на секунду моргнул», если лимитер не критичен для жизни процесса — продуктовое решение.

**trusted-proxies** — Java-regex доверенных прокси. Без него фильтры Forwarded / X-Forwarded в 5.x не активируются. Нужен, чтобы реальный IP клиента дошёл до сервисов и чтобы клиент не подделал `X-Forwarded-For`. Значение = regex IP края (Ingress/HAProxy/WAF), не `.*`.

**SecureHeaders** — фильтр типовых заголовков браузерной защиты (HSTS, `X-Frame-Options: DENY`, CSP). Это не WAF. Жёсткий CSP ломает портал; HSTS — только если TLS на этом имени стабилен.

**RequestHeaderToRequestUri** — фильтр: URI бэкенда взять из заголовка запроса. Официально только в доверенной среде. С мира это готовый SSRF. Заголовок срезать с клиентских запросов до шлюза. На публичных маршрутах не включать «для удобства Postman».

**Local response cache** — кэш ответов в памяти пода (Caffeine). Не Redis, не CDN, не общее на три ЦОДа. GET без тела, при разрешающем `Cache-Control`. Каждый под кэширует своё. Не кэшировать ПДн «на всякий».

**BFF (Backend for Frontend)** — шлюз, заточенный под конкретный UI (мобилка / портал). Не обязан быть единственной точкой входа всей системы.

**Старый префикс YAML** — в Gateway 4.x: `spring.cloud.gateway.routes`. В 5.x WebFlux: `spring.cloud.gateway.server.webflux.routes`. Старый блок молча не биндится — маршрутов нет. На страницах TLS доки ещё встречается старый `spring.cloud.gateway.httpclient`; для 5.0.3 биндинг — `spring.cloud.gateway.server.webflux.httpclient` по appendix.

**Northbound / southbound** — вход людей и систем-потребителей в ваши HTTP API vs исходящие вызовы к ведомствам. Шлюз — northbound. Southbound делает интеграционное API. Иначе второй невидимый контур интеграций плюс SSRF.

**WAF (SafeLine)** — детектор семантики атаки. SecureHeaders и rate limit — не детектор SQLi. Документ SafeLine отдельно.

**CORS** — правила браузера, с каких Origin можно дергать API. Явные `allowedOrigins`, не `*` вместе с cookie. `add-to-simple-url-handler-mapping=true`, если preflight OPTIONS не попадает в предикат маршрута.

**OIDC / JWT / resource server / Spring Security** — проверка токена IdP **до** проксирования внутренних API. Origin всё равно должен проверять JWT: иначе обход шлюза = обход auth. Camunda Operate/Tasklist, OpenSearch Dashboards, Grafana — отдельные входы с отдельным IdP, не тот же публичный шлюз без жёсткого ACL.

**mTLS партнёров** — клиентский сертификат вместо (или вместе с) JWT.

**NTP** — часы для JWT, TLS, метрик, Redis TTL.

**Redis (лимитер)** — общий `RequestRateLimiter` на Redis. HA Redis проектируется отдельно (документ Redis 7). Caffeine/Bucket4j без распределённого бэкенда считает лимит на под — при 10 репликах квота ×10. Нет Redis — не имитировать кластерный лимит тремя подами: WAF/HAProxy или честно локально. Redis шлюза ≠ Redis сессий «на одной базе SELECT 2», если цель Cluster: в Redis Cluster живёт только DB 0. Заголовки `X-RateLimit-*` включены по дефолту. Стартер: `spring-boot-starter-data-redis-reactive`. Учение 429.

**Caffeine / Bucket4j** — лимит или кэш в памяти пода, не городской.

**Weight (canary)** — предикат доли трафика на маршрут. Веса считаются на шлюзе, это не service mesh.

**Метрики** — `spring.cloud.gateway.server.webflux.metrics.enabled` включить явно (дефолт appendix: выкл.). Path-tag без понимания кардинальности раздует ряды Prometheus. Трейсы — Grafana Tempo.

**Probes / liveness / readiness** — не через тяжёлый origin. Не готов — нет трафика. Liveness, который пингует origin как обязательный — каскад рестартов при медленном бэкенде.

**Graceful shutdown** — `server.shutdown=graceful` + `terminationGracePeriodSeconds` длиннее, чем самый длинный легитимный запрос интеграций. Иначе Kubernetes режет mid-flight. Rolling: `maxUnavailable` с учётом PDB.

**SpEL** — выражения в YAML (`key-resolver: "#{@bean}"`). `restrictive-property-accessor.enabled=true` (дефолт) не выключать. Только свои бины, не пользовательский ввод в SpEL.

**ModifyRequestBody** — чтение/смена тела в шлюзе: память и ПДн.

**JSON-to-gRPC / WebSocket** — отдельные маршруты и таймауты. WebFlux умеет ws/wss. gRPC-путь в 5.0.3 как раз патчили. Не включать «поиграть» без нужды.

**SBOM** — перечень зависимостей. Шлюз тянет пол-Spring Cloud транзитивно; скан обязателен. Pin: Boot + BOM 2025.1.3 + digest образа.

**kube-proxy / IPVS** — балансировка Service Kubernetes на поды origin при `http://svc.ns.svc.cluster.local:port`. Предпочтительнее `lb://`, если не приняли DiscoveryClient.

**RPS** — запросов в секунду. Официальных бенчмарков проекта нет. Влияют RPS, размер тела, число одновременных долгих запросов, доля TLS, retry (умножает нагрузку на origin). «Терабайты данных» на размер шлюза почти не влияют.

**FD / прямой буфер Netty** — дескрипторы и off-heap буферы. Без limits JVM+буферы легко съедают ноду.

**Tanzu / Spring Enterprise / Gateway 4.3.x** — другие дистрибутив, Boot и YAML. Файл про OSS 5.0.3 через BOM 2025.1.3.

**Integration API** — единственный исходящий контур к госслужбам. Gateway умеет дойти до этого API по HTTP. Он не решает, какие ведомства дергать, и сам в интернет к ведомствам не ходит.

**Service / Endpoints Kubernetes** — DNS и список живых подов origin. Readiness снимает под с Endpoints. Предпочесть `http://svc.ns.svc.cluster.local:port`.

**ConfigMap** — объект Kubernetes с YAML маршрутов. Согласованность между ЦОДами — один Git-артефакт, иначе в одном ЦОДе путь есть, в другом 404.

**Principal** — уже установленная личность запроса (после Spring Security). Дефолтный KeyResolver берёт `Principal.getName()`.

**429** — ответ «лимит исчерпан». Учение прода для Redis-лимитера. Если Redis упал — либо 429/ошибка лимитера, либо (если криво настроен fallback) дыра в лимите.

**5xx** — ошибка шлюза или origin. Circuit breaker на критичных синхронных маршрутах, когда origin умеет умирать; учение «origin 5xx / timeout».

**Long-poll** — долгий открытый HTTP-запрос. Слишком короткий `response-timeout` рвёт легитимный long-poll ведомства; слишком длинный забивает пул. Дефолт connect-timeout 30 с — не «как у REST».

**Digest образа** — пин не тегом `latest`, а хешем слоёв. Вместе с BOM 2025.1.3 и Boot 4.0.8 (или явно принятым 4.1.x).

**Vault / Secret** — TLS и Redis ACL. Не в Git.

Источники формулировок: глоссарий и тело `Spring Cloud Gateway.md`. Новых порогов RPS и RTT здесь нет. Дефолт connect-timeout 30 с — из appendix 5.0.3.
