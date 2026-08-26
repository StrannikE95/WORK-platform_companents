# Spring Cloud Gateway 5.0.3 — развёртывание и настройка

Версия ПО: **Spring Cloud Gateway 5.0.3** (релизная дата **20 августа 2026**; на дату подготовки документа это последний стабильный патч линии **5.0**, входит в release train **Spring Cloud 2025.1.3**, aka **Oakwood**).  
Документация линейки: https://docs.spring.io/spring-cloud-gateway/reference/ (страницы Server WebFlux помечены **5.0.3**)  
BOM: `org.springframework.cloud:spring-cloud-dependencies:2025.1.3`  
Анонс поезда: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released  
Матрица поездов: https://spring.io/projects/spring-cloud  

Поезд **2025.1.3** официально **основан на Spring Boot 4.0.8**. С **2025.1.2** линейка Oakwood совместима также с **Spring Boot 4.1.x**. Это библиотека внутри **вашего** Spring Boot-приложения, не готовый «кластер как Kafka», который ставят Helm-чартом вендора.

В **5.0.3** закрыт **CVE-2026-47879** (SSRF и доступ к локальным файлам через gRPC-путь Gateway). В прод не тащить 5.0.2 и ниже этой линии без патча.

Этот текст — не мануал «скопируй `application.yml`», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

Spring Cloud Gateway **не было** в исходном описании архитектуры как отдельный продукт (там — Kafka, Camunda, озеро эталона, интеграционное API). Ниже — как поставить **входной HTTP-шлюз** на ту же платформу: Kubernetes, 3 ЦОДа, микросервисы, интеграционное API как единственный выход к госслужбам. Это **не** замена Kafka, **не** Camunda, **не** озеро клиентских данных и **не** сам интеграционный оркестратор «какие 30+ ведомств дергать».

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **API Gateway (шлюз)** | Приложение, которое принимает HTTP(S) снаружи (или от внутреннего клиента) и **проксирует** запрос на нужный сервис: маршруты, заголовки, лимиты, аутентификация, обрыв при аварии бэкенда. |
| **Spring Cloud Gateway Server** | Полноценный шлюз: отдельное Spring Boot-приложение (или встроенный модуль), которое живёт маршрутами. Не путать с **Proxy Exchange**. |
| **Proxy Exchange** | Не отдельный шлюз, а объект `ProxyExchange` в **уже существующем** `@Controller` / WebFlux-хендлере. «Проксировать из своего кода». Другая поставка, другой операционный контур. |
| **WebFlux** | Реактивный стек Spring (Netty, неблокирующий I/O). Стартер: `spring-cloud-starter-gateway-server-webflux`. В 5.x это **полнофункциональный** Server. |
| **WebMVC** | Классический Servlet-стек (обычно Tomcat). Стартер: `spring-cloud-starter-gateway-server-webmvc`. Другой префикс настроек, другой набор фильтров. **Нельзя** смешать оба стартера в одном процессе «на всякий случай». |
| **Маршрут (route)** | Правило: «если запрос подходит под **предикаты** — отправить на **URI**, прогнав **фильтры**». У маршрута есть `id` и `order` (кто победит, если подошло несколько). |
| **Предикат (predicate)** | Условие совпадения: путь `/api/**`, хост, метод, заголовок, вес (canary) и т.д. Несколько предикатов в YAML **склеиваются через AND**. |
| **Фильтр (GatewayFilter)** | Действие на **этом** маршруте: срезать префикс, добавить заголовок, rate limit, retry, circuit breaker. Часть логики — **до** вызова бэкенда, часть — **после** ответа. |
| **Глобальный фильтр (GlobalFilter)** | То же по смыслу, но вешается на **все** маршруты (маршрутизация Netty, `lb://`, запись ответа). |
| **URI маршрута** | Куда проксировать. `https://host:port` — фиксированный адрес. `lb://service-id` — через Spring Cloud LoadBalancer. `forward:/local` — внутрь этого же приложения. **Путь в URI маршрута официально игнорируется** (берётся путь входящего запроса + фильтры вроде StripPrefix). |
| **`lb://`** | Схема «найди инстансы сервиса и выбери один». Нужен `spring-cloud-starter-loadbalancer` и DiscoveryClient (Eureka/Consul/K8s…) **или** явное перечисление инстансов. Без этого `lb://` не «магия Kubernetes». |
| **Discovery locator** | Автосоздание маршрутов вида `/имяСервиса/**` по всем сервисам из реестра. Дефолт **выключен**. Включить «чтобы не писать YAML» в проде = выставить в интернет **все** найденные Service. |
| **Netty HttpClient** | То, чем WebFlux-шлюз ходит **на бэкенд**. У него свой пул соединений, TLS, таймауты. Это не Tomcat и не Ingress. |
| **Пул соединений (pool)** | Сколько исходящих HTTP-коннектов к бэкендам держать. Дефолт типа пула в 5.0.3 — **`elastic`**. Режим **`fixed`** ограничивает `max-connections`. |
| **connect-timeout / response-timeout** | Сколько ждать **установления** TCP к бэкенду / **полного ответа**. Глобально и **на маршрут** (в metadata). Для интеграций с лагами ведомств это критично. |
| **Token bucket** | Алгоритм rate limit: ведро токенов, запрос забирает токены, ведро пополняется. Штатная Redis-реализация Gateway так и устроена. |
| **KeyResolver** | «По какому ключу считать лимит»: пользователь, IP, API-ключ. Дефолт — `Principal.getName()`. **Нет ключа → запрос отклоняется** (это можно ослабить, в проде так делать нельзя вслепую). |
| **Circuit breaker (автомат защиты)** | Если бэкенд сыплет ошибками/таймаутами — перестать долбить его и сразу отдавать fallback (или 5xx). В Gateway это **фильтр**, реализация — Spring Cloud CircuitBreaker (обычно Resilience4j). |
| **Fallback** | Запасной ответ/URI, когда автомат разомкнут или бэкенд мёртв. Без него клиент просто получает ошибку шлюза. |
| **Retry** | Повторить запрос к бэкенду. На **GET** иногда уместно. На **POST в ведомство** без идемпотентности = двойное списание/двойная заявка. |
| **Actuator `/actuator/gateway`** | Пульт маршрутов: список, создание, удаление, refresh. В 5.0.3 доступ **по умолчанию выключен**. `unrestricted` = можно **менять маршруты на живую** (классический вектор SSRF). |
| **trusted-proxies** | Java-regex доверенных прокси. **Без него фильтры Forwarded / X-Forwarded в 5.x не активируются.** Нужен, чтобы реальный IP клиента дошёл до сервисов и чтобы клиент **не подделал** `X-Forwarded-For`. |
| **SecureHeaders** | Фильтр, который навешивает типовые заголовки браузерной защиты (HSTS, `X-Frame-Options: DENY`, CSP и т.д.). Это не WAF. |
| **RequestHeaderToRequestUri** | Фильтр: «URI бэкенда взять из заголовка запроса». Официально **только в доверенной среде**. С мира это готовый SSRF. |
| **Local response cache** | Кэш ответов **в памяти пода** (Caffeine). Не Redis, не CDN, не общее на три ЦОДа. GET без тела, при разрешающем `Cache-Control`. |
| **BFF (Backend for Frontend)** | Шлюз, заточенный под конкретный UI (мобилка / портал). Не обязан быть единственной точкой входа всей системы. |
| **HPA / PDB** | Horizontal Pod Autoscaler — добавить/убрать реплики по метрике. PodDisruptionBudget — «не выселяй все реплики сразу». |
| **RPO / RTO** | Сколько данных готовы потерять / как быстро восстановиться. У шлюза «данные» — это **конфиг маршрутов и (если есть) счётчики лимитов**, не клиентская база. |
| **Старый префикс YAML** | В Gateway **4.x**: `spring.cloud.gateway.routes`. В **5.x WebFlux**: `spring.cloud.gateway.server.webflux.routes`. Старый блок **молча не биндится** — маршрутов нет. |

---

## Основные элементы системы и зависимости

### Что входит в Spring Cloud Gateway 5.0.3 (это одно ПО из нескольких «вкусов»)

| Артефакт | Зачем | Когда брать |
|---|---|---|
| **`spring-cloud-starter-gateway-server-webflux`** | Полноценный реактивный шлюз на Netty | Целевой прод этого файла |
| **`spring-cloud-starter-gateway-server-webmvc`** | Шлюз на Servlet | Если команда принципиально без WebFlux; фичи и YAML **другие** |
| **Proxy Exchange (webflux / webmvc)** | Прокси из своего контроллера | Не «платформенный вход», а кусок приложения |
| **Маршруты YAML / Java `RouteLocator`** | Контракт входа | YAML для GitOps; Java — когда предикат сложнее AND |
| **Netty HttpClient** | Исходящие вызовы к origin | Таймауты, TLS, пул |
| **Фильтры + предикаты** | Политика на маршруте | Явный список, не «все дефолты подряд» |

Рядом, но это **другое ПО** (часть у вас уже описана отдельно): **Kubernetes Ingress / Gateway API**, **HAProxy**, **SafeLine WAF**, **Redis** (для общего rate limit), **Spring Cloud LoadBalancer / Spring Cloud Kubernetes**, **IdP (OIDC)**, **Prometheus/Grafana Tempo**.

### Чего в Spring Cloud Gateway нет (частая путаница)

| Нужно системе | Это не Spring Cloud Gateway | Зачем помнить |
|---|---|---|
| Шина событий, `acks=all`, consumer group | Apache Kafka | Шлюз — HTTP. Kafka-клиенты ходят на брокеры напрямую |
| Исполнение BPMN | Camunda 8 | Процесс не «маршрут Path=» |
| Озеро / SoT клиентских данных | Ваше озеро / БД | Шлюз не хранит досье и не ищет по ИНН |
| Оркестрация 30+ госслужб с лагами, парсингом, нормализацией | Ваше **интеграционное API** | Gateway умеет **дойти** до этого API. Он **не** решает «какие ведомства дергать» |
| WAF (семантика атаки) | SafeLine / аналог | SecureHeaders и rate limit — не детектор SQLi |
| Ingress-объекты Kubernetes | Ingress-контроллер / Gateway API | SCG — обычный Deployment+Service |
| Плавающий городской VIP на 3 ЦОДа | HAProxy + DNS/Anycast (см. документ HAProxy) | Реплики шлюза сами IP между площадками не таскают |
| Балансировка Kafka / Postgres / gRPC Camunda `:26500` | Не этот HTTP-прокси | Иначе ломается протокол или таймауты стримов |
| Кластер с кворумом и репликацией маршрутов | Нет | Несколько подов = несколько независимых JVM. Согласованность конфига — Git/ConfigMap, не Raft |
| ГОСТ TLS / СКЗИ | Не заявлено | Штатный TLS JDK / Netty (обычные алгоритмы) |
| Общий кэш ответов на 3 ЦОДа | Нет (LocalResponseCache — RAM пода) | Иначе «кэш» врёт после выката и между зонами |
| Готовая цифра «хватит N RPS» | Нет в доке проекта | Ёмкость — замер **вашего** маршрута и TLS |

### Зависимости окружения (обязательны)

- **JDK.** Spring Boot 4.0 официально: минимум **Java 17**, first-class **Java 25**, совместимость линейки 4.0 — до **Java 25**. Spring Boot 4.1.x (если возьмёте её под Oakwood): минимум 17, до **Java 26** (system requirements текущей 4.1.1). Сборщики: Maven **≥ 3.6.3** или Gradle **8.14+ / 9.x** (требования текущей доки Boot 4.1.1; для 4.0.8 смотреть system requirements **той** минорной линейки перед пином).
- **Не класть WebFlux-шлюз в внешний Tomcat как WAR «как старый Spring MVC»** — модель Server WebFlux завязана на Netty-сервер Boot.
- **Kubernetes (у вас заявлен):** Deployment + Service + (обычно) Ingress. Это оркестрация процесса, не замена маршрутов.
- **Сеть до origin:** шлюз устанавливает **свои** исходящие соединения. NetworkPolicy: куда ему **можно** ходить (внутренние Service), а куда **нельзя** (произвольный интернет, metadata-серверы облака, брокер Kafka).
- **Redis** — только если включаете **общий** `RequestRateLimiter` на Redis. Тогда HA Redis проектируется **отдельно** (у вас есть документ Redis 7). Caffeine/Bucket4j **без** распределённого бэкенда считает лимит **на под** — при 10 репликах квота ×10.
- **DiscoveryClient** — только если используете `lb://` или discovery locator. На Kubernetes это либо **DNS Service** (`http://name.ns.svc`), либо Spring Cloud Kubernetes + RBAC на API. Оба пути валидны; смешивать «и Eureka, и K8s, и хардкод» — путь к ночным 503.
- **Часы (NTP):** JWT, TLS, метрики, Redis TTL.
- **PKI** для TLS на входе (часто терминирует Ingress/HAProxy) и **для исходящего** TLS к origin / mTLS.

Официальных бенчмарков «RPS на наше железо» у проекта Gateway **нет**. Не подставляйте цифры из блогов.

### Как шлюз стыкуется с вашей архитектурой

Два разных контура. Их нельзя склеить в один «ферма на всё».

```
Клиенты портала / партнёры
        │
   HAProxy / городской VIP     ← документ HAProxy
        │
   SafeLine WAF (вход HTTP)    ← документ SafeLine
        │
   Ingress / Gateway API
        │
   Spring Cloud Gateway        ← этот документ  (northbound)
        │
   ┌────┴─────────────────────┐
   ▼                          ▼
микросервисы (HTTP)     Integration API (HTTP)
                               │
                               ▼
                    30+ госслужб (наружу)  — не через Gateway
```

- **Northbound:** люди и системы-потребители ходят в ваши HTTP API. Здесь шлюз уместен: маршруты, JWT/OIDC, лимиты, единый CORS, обрыв больного сервиса.
- **Southbound:** исходящие вызовы **к ведомствам** делает **интеграционное API**. Gateway туда **не** должен ходить сам (иначе второй, невидимый контур интеграций, плюс SSRF при кривом маршруте).
- Kafka, Zeebe gRPC, диск озера — **мимо** этого шлюза.

---

## Краткие вводные

### Зачем вам шлюз в этой архитектуре

Микросервисы на событиях **не отменяют** синхронный HTTP: портал, Tasklist-обвязка, «дай статус заявки», вход в Integration API. Без шлюза каждый сервис сам торчит в Ingress — 30 политик CORS, 30 способов протащить JWT, 30 дыр Actuator.

Gateway даёт одно место для:

1. **Маршрутизации** по пути/хосту без правки Ingress на каждый сервис.
2. **Поперечных политик:** заголовки, размер запроса, rate limit, circuit breaker.
3. **Смены origin без смены клиента** (переезд Service, canary через `Weight`).

Он **не** даёт: согласованность данных, очередь событий, «пережить два ЦОДа».

### Как устроена отказоустойчивость (идея, не магия)

Процесс шлюза **почти stateless**. Упал под — Kubernetes поднимает другой. Клиент, который уже висел на сломанном TCP, получит обрыв: это норма для HTTP-прокси.

| Что падает | Что происходит |
|---|---|
| Один origin | Без circuit breaker шлюз будет копить таймауты и жечь пул. С автоматом — быстрый отказ/fallback на **этом** маршруте |
| Один под Gateway | Service/Ingress перестаёт слать сюда (при живом probe). Нужны **≥2** пода |
| Redis rate limiter | Либо 429/ошибка лимитера, либо (если криво настроен fallback) **дыра в лимите**. Это зависимость HA |
| Конфиг маршрутов | Живёт в приложении/ConfigMap. «Кластер шлюзов» сам его не реплицирует |
| Один ЦОД | Поды в других ЦОДах **не выбирают лидера**. Нужен тот же городской слой, что и для HAProxy |
| Неверный `trusted-proxies` | Либо нет X-Forwarded (сервисы видят IP шлюза), либо доверяете миру — подмена IP |

Следствие: HA входа = **несколько живых подов в разных зонах** + живой Ingress/HAProxy + (если лимиты общие) **живой Redis**. Один под на три ЦОДа отказоустойчивости **не даёт**.

### Как устроено масштабирование

Два независимых рычага:

1. **Вертикально** — CPU/RAM пода, размер кучи JVM, пул Netty HttpClient (`fixed` + `max-connections`), `response-timeout` (слишком большой = больше «висящих» запросов). TLS handshake на **входе**, если терминируете TLS **на шлюзе**, жрёт CPU; если TLS снял HAProxy/Ingress — шлюз дешевле.
2. **Горизонтально** — реплики Deployment. Это умножает ёмкость **и** даёт HA. HPA имеет смысл по CPU / RPS / latency, не по «терабайтам озера».

«Терабайты данных» на размер шлюза почти не влияют. Влияют: **RPS, размер тела, число одновременных долгих запросов**, доля TLS, retry (умножает нагрузку на origin).

Локальный кэш ответов **не** масштабируется горизонтально как общая память: каждый под кэширует своё.

### Безопасность самого шлюза

Шлюз видит URL, заголовки, часто токены, при буферизации тела — **содержимое** запросов к внутренним API. Actuator с записью маршрутов historically использовали как **SSRF «на заказ»**. Wiretap Netty пишет трафик в лог — в проде это утечка.

Фильтр `RequestHeaderToRequestUri` официально помечен: только доверенная среда; заголовок должен быть **срезán** с клиентских запросов до шлюза.

В 5.0.3 закрыт CVE про SSRF/файлы на gRPC-пути — ещё одна причина не сидеть на 5.0.2.

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Берём OSS Spring Cloud Gateway 5.0.3 через BOM 2025.1.3**, не коммерческий Tanzu/Spring Enterprise fork и не Spring Cloud Gateway **4.3.x** (другой Boot, другой YAML).
2. **Целевой вкус прода — Server WebFlux.** MVC не запрещён, но дальше текст и префиксы — про WebFlux. Два вкуса в одном JAR не смешиваем.
3. **Spring Boot — 4.0.8** как версия, на которой **явно основан** поезд 2025.1.3. Переход на 4.1.x допустим матрицей Oakwood с 2025.1.2, но это **отдельное** решение совместимости всех облачных модулей, не «молча latest Boot».
4. **Java на шлюзе — 21 или 25 LTS** (оба в зоне поддержки Boot 4.0). 17 — легальный минимум, не лучшая цель для greenfield 2026.
5. **Роль прода — northbound HTTP** перед микросервисами и Integration API. Исходящие вызовы к госслужбам шлюз **не** выполняет.
6. **Перед шлюзом есть слой края:** HAProxy и/или WAF и/или Ingress. Шлюз **не** единственный процесс с публичным IP на три ЦОДа.
7. **Три ЦОДа = три домена отказа.** Реплики шлюза раскладываются по зонам (один K8s на три зоны) **или** по три отдельных деплоя (три кластера). Stretch «один под, но диск в другом зале» не рассматривается.
8. **Нагрузки HTTP нет** — поэтому **нет** числа реплик и ядер «хватит для прода». Есть минимум размещения и правило роста.
9. **Формального SLA нет.** Схема «поды в трёх зонах + городской failover» переживает отказ 1 ЦОДа **по HTTP-входу**, если DNS/Anycast/Ingress это умеют. Это не юридический 24/7.
10. **Общий rate limit в проде — Redis** (у вас линейка Redis 7 описана отдельно), не Caffeine на поде. Если Redis не будет — лимиты либо на WAF/HAProxy, либо честно локальные.
11. **Маршруты в Git** (ConfigMap/Helm values из CI), не `POST /actuator/gateway/routes` в бою. Actuator записи — `read-only` или выключен.
12. **Корпоративный PKI есть или будет.** `use-insecure-trust-manager: true` к origin в проде не считаем настройкой.
13. **Шифрование канала — обычный TLS JDK/Netty.** ГОСТ не озвучен; vanilla Gateway его не закрывает.
14. **Тестовый стенд изолирован.** На нём допустим HTTP и открытый actuator **внутри** VPN. Боевые маршруты и таймауты интеграций лучше не упрощать в коде.
15. **Discovery locator в проде выключен.** Маршруты явные. `lb://` — только если приняли Spring Cloud Kubernetes (и RBAC) вместо DNS Service.
16. **Camunda Operate/Tasklist, OpenSearch Dashboards, Grafana** — отдельные входы с отдельным IdP; не тащить их через тот же публичный шлюз, что портал, без жёсткого ACL.

---

## Критически важные условия, которых нет в исходном контексте

Их нужно закрыть **до** «поставили шлюз в прод».

| Пробел | Почему это ломает решение |
|---|---|
| **Где именно стоит шлюз** относительно HAProxy, WAF, Ingress | Меняются TLS, `trusted-proxies`, кто видит тело запроса, health check, реальный IP |
| **Шлюз = Integration API или только вход к нему?** | Если «шлюз сам оркестрирует ведомства» — вы получаете второй интеграционный контур без парсинга/нормализации из постановки |
| **RTT и потери между ЦОДами** | На локальный origin почти не влияет. Влияет, если шлюз ЦОД-1 ходит в сервис ЦОД-2; и если Redis лимитера «на город» |
| **Профиль нагрузки** (RPS, p99 latency, размер тела, доля долгих запросов, пик) | Без этого нельзя сказать ни HPA, ни `max-connections`, ни кучу |
| **Максимальное ожидание ведомства** | Дефолтный connect timeout в appendix — **30 с**, если не задан. Слишком короткий рвёт легитимный long-poll; слишком длинный забивает пул |
| **Топология Kubernetes** (1 кластер × 3 зоны или 3 кластера) | От этого зависит, один Deployment или три независимых выката |
| **IdP / модель токена** (JWT resource server vs BFF-cookie vs mTLS партнёров) | Иначе KeyResolver и Security «как получится» |
| **Нужен ли общий rate limit** | Тогда Redis HA обязателен; иначе лимит на WAF или честный per-pod |
| **Требования ИБ: ГОСТ, 152-ФЗ, КИИ, запрет привилегированных подов** | Меняют TLS, логи (PII в URL), куда можно писать access log |
| **Какие URL публичные, какие только из VPN** | Один Ingress на «всё» = Actuator и внутренние API рядом с порталом |
| **gRPC / WebSocket / файловые upload** | WebFlux умеет ws/wss; gRPC-путь в 5.0.3 как раз патчили. Это отдельные маршруты и таймауты, не «как REST» |

---

## Краткое описание ключевых этапов и элементов развертывания

Общий каркас (и тест, и прод):

1. Зафиксировать **BOM 2025.1.3** + артефакт **Gateway 5.0.3** + Boot **4.0.8** (или явно принятый 4.1.x). Не `spring-cloud-starter-gateway` без `-server-webflux` из старых гайдов.
2. YAML только под `spring.cloud.gateway.server.webflux.*`. Прогнать `/actuator/gateway/routes` на стенде: если пусто — почти всегда старый префикс.
3. Явные маршруты (`id`, `uri`, предикаты, `order`). `fail-on-route-definition-error: true` (дефолт) — не глушить.
4. Таймауты: глобальный `httpclient.connect-timeout` / `response-timeout` **и** длинные metadata на маршруте Integration API. Отрицательный per-route `response-timeout` **отключает** глобальный — знать, что делаете.
5. Probes: liveness/readiness **не** через тяжёлый origin; Actuator `health` с аккуратным набором индикаторов (не валить под из-за «Redis на секунду моргнул», если лимитер не критичен для жизни процесса — это продуктовое решение).
6. Actuator: `/gateway` не `unrestricted` с мира; `/env`, `/heapdump` не публиковать.
7. TLS: на входе — слой края; к origin — доверенные сертификаты, **не** `use-insecure-trust-manager`.
8. Мониторинг: метрики шлюза **включить явно** (`spring.cloud.gateway.server.webflux.metrics.enabled`), Prometheus, **не** включать path-tag без понимания кардинальности. Трейсы (у вас есть Grafana Tempo).
9. Учение: убить под, убить origin, открыть circuit breaker, исчерпать rate limit, rolling update.

Дальше — два режима.

---

### 1 инстанс: тестовый стенд, 1 ЦОД, без нагрузки

**Цель стенда:** чтобы разработчики ходили в сервисы через **те же пути**, что в проде, отладили StripPrefix/JWT/CORS и таймауты Integration API. **Не** цель: доказать отказ ЦОДа и RPS.

#### Топология

Один Deployment, `replicas: 1` (допустимо 1 на тесте):

- образ вашего приложения с `spring-cloud-starter-gateway-server-webflux` **5.0.3**;
- 2–3 маршрута на тестовые Service (`http://svc.namespace.svc:8080`);
- HTTP внутри кластера/VPN;
- Actuator `/gateway` в режиме **read-only**, только из namespace мониторинга/VPN;
- без Redis-лимитера (или встроенный Redis в compose — понимать, что это не прод);
- discovery locator **выключен**;
- `trusted-proxies` можно не мучить, если нет цепочки прокси; как только появляется Ingress — задать regex IP Ingress.

Java-маршруты через `RouteLocatorBuilder` на тесте допустимы, но тогда прод должен брать **тот же** механизм (не «на тесте Java, в проде YAML» без дисциплины).

#### Что сознательно упрощаем

| Параметр | На тесте | Почему можно |
|---|---|---|
| Одна реплика | да | Некому делить трафик |
| PLAINTEXT до шлюза | да, если сеть закрыта | Иначе утонете в PKI до маршрутов |
| Redis rate limit | нет | Нет атаки и нет HA Redis |
| Circuit breaker | можно отложить | Нет хаоса origin, но таймауты лучше сразу |
| TLS к origin | по ситуации | Если сервисы HTTP внутри mesh |

#### Чего на тесте **не** стоит упрощать

- Префикс **`spring.cloud.gateway.server.webflux`**.
- Реальные **пути** публичного API и StripPrefix — иначе в проде «внезапно» 404.
- **Таймауты** на маршруте интеграций — иначе HAProxy уже ждёт минуты, а шлюз рвёт через 30 с.
- Клиенты ходят **на шлюз**, не в обход на под сервиса (иначе тесты врут).
- Версии BOM/Gateway **pinned**, не `latest`.
- Не включать `RequestHeaderToRequestUri` «для удобства Postman».
- `httpclient.wiretap: false` даже на тесте, если в запросах ПДн.

#### Сильные стороны такой схемы

- Поднимается за часы; совпадает с моделью проекта: Boot-приложение + YAML.
- Дешёвая.
- Ловит ошибку префикса 5.x и кривой RewritePath до прода.

#### Слабые стороны (обязательно понимать)

- Падение пода = нет входа, если DNS указывает только сюда.
- Стенд **не** ловит: HPA, Redis-лимитер на 3 репликах, rolling drain, CPU TLS, межЦОДовый origin.
- Успешный тест на одном поде **не** является доказательством трёх ЦОДов.

Практическая рекомендация: препрод = **≥2 реплики + PDB + TLS на Ingress + один боевой маршрут Integration API с боевым таймаутом**, даже без боевого RPS.

---

### Прод: 3 ЦОДа, нагрузка

Цифр «реплик под высокую нагрузку» **нет**. Ниже правила, без которых экземпляр не считается готовым.

#### Шаг 0. Макроархитектура (сделать до установки)

**Вариант A — целевой: шлюз внутри Kubernetes, реплики во всех трёх зонах, origin того же ЦОДа/зоны, край (HAProxy/WAF/Ingress) как в документах HAProxy и SafeLine.**

```
                 клиенты
                    │
         DNS / Anycast / городской ADC
            ┌───────┼───────┐
            ▼       ▼       ▼
         край ЦОД1 край ЦОД2 край ЦОД3
            │       │       │
         Ingress     …         …
            │
         SCG (несколько подов, topologySpread / anti-affinity)
            │
         Service origin **этой** площадки
```

- Падение 1 пода: Endpoints исключают его (при живых probes).
- Падение 1 ЦОДа: городской слой перестаёт слать сюда. Шлюз в других залах об этом **не договаривается**.
- Origin — **локальный**. «Шлюз в ЦОД-1, Integration API только в ЦОД-3» превращает RTT и сетевой шторм в деградацию **всего** входа.

**Вариант B — три изолированных кластера K8s.**  
Три независимых выката **одного** Git-артефакта. Без городского failover это три острова. Конфиг маршрутов должен быть идентичен (GitOps), иначе «в одном ЦОДе путь есть, в другом 404».

**Вариант C — шлюз как единственный публичный процесс без HAProxy/Ingress.**  
Возможно на маленьком контуре. На трёх ЦОДах вы тащите на JVM то, что лучше делает L4/L7 край (VIP, PROXY protocol, drain). Для заявленной платформы **не** рекомендуется как единственный слой.

**Вариант D — один под / один ЦОД «пока». **  
Не удовлетворяет отказу площадки.

##### Сильные / слабые стороны A

| Сильное | Слабое |
|---|---|
| Stateless, хорошо ложится на K8s | Нужен грамотный край и probes |
| Отказ зоны переживается репликами в других | Общий Redis-лимитер становится отдельной HA-задачей |
| Совпадает с «у нас Kubernetes» | Ошибка NetworkPolicy = шлюз не видит origin или видит весь кластер |

#### Маршруты и discovery

Прод-правила:

| Правило | Зачем |
|---|---|
| Явный список маршрутов в Git | Аудит, воспроизводимость, меньше сюрпризов |
| `discovery.locator.enabled=false` | Иначе каждый Service может стать публичным путём |
| Предпочесть `http://svc.ns.svc.cluster.local:port` | Балансировка kube-proxy/IPVS, без RBAC на list endpoints |
| `lb://` только с принятым DiscoveryClient | Нужны LoadBalancer, права в API, понимание 503 vs 404 (`loadbalancer.use404`) |
| Не использовать путь в `uri:` | Документация: path на URI маршрута **игнорируется** |
| Отдельный маршрут Integration API | Свои таймауты, без Retry на POST, без кэша |
| Не проксировать Kafka/Camunda gRPC/админки | Чужой протокол и чужая модель доступа |

Canary: предикат `Weight` — штатный. Это не service mesh; веса считаются **на шлюзе**.

#### Таймауты, пул, retry, circuit breaker

| Ручка | Прод-смысл |
|---|---|
| `httpclient.connect-timeout` | Миллисекунды; дефолт appendix **30 с**, если не задан — для внутреннего Service обычно слишком много (дольше держит слот при мёртвом поде) |
| `httpclient.response-timeout` | Duration глобально; на маршруте metadata — **миллисекунды**. Для ведомств — **из замера**, не «как у REST 5s» |
| `httpclient.pool.type=fixed` + `max-connections` | Чтобы elastic не размножил коннекты до OOM/FD при шторме |
| `pool.max-idle-time` / `max-life-time` | Не держать вечно полумёртвые keep-alive за NAT |
| CircuitBreaker + fallback | Обязателен на критичных синхронных маршрутах, когда origin умеет умирать |
| Retry | Только идемпотентные методы/операции. POST в Integration API — **по умолчанию нет** |
| TimeLimiter Resilience4j | В поезде 2025.1.3 отдельно написано, что **дефолт TimeLimiterConfig в factory больше не используется** — не копировать старые гайды 2024 года вслепую |

Spring Cloud CircuitBreaker **5.0.3** едет в том же BOM. Реализацию (Resilience4j) фиксируйте стартером из поезда, не случайным артефактом с Maven Central «как в блоге 2019».

#### Rate limiting

- **Общий лимит на пользователя/IP в проде:** Redis token bucket (`spring-boot-starter-data-redis-reactive`) + осмысленный `KeyResolver` (не query-параметр `user` — в доке прямо: пример **не для прода**).
- Ключ без Principal при JWT: резолвить `sub`/client_id из уже проверенного токена, не из заголовка, которому верит клиент.
- Нет ключа → deny (дефолт). Иначе анонимный трафик обходит лимит.
- Заголовки `X-RateLimit-*` включены по дефолту — не светить лишнюю внутреннюю политику, если не нужно.
- Bucket4j+Caffeine на поде — **не** городской лимит.
- Redis шлюза ≠ Redis сессий приложения «на одной базе SELECT 2», если цель — Cluster: в Redis Cluster живёт только DB 0 (см. документ Redis 7).

Если Redis для лимитера нет — не имитировать «кластерный лимит» тремя подами. Либо WAF/HAProxy, либо честно локально.

#### Kubernetes-специфика

- Deployment, не StatefulSet (нет стабильной личности).
- `topologySpreadConstraints` / anti-affinity по зоне = ЦОД.
- PDB: не допускать `replicas=0` при drain узла.
- Readiness: не готов — нет трафика. Liveness: не убивать под из-за одного медленного origin, если health включает обязательный ping origin — получите каскад рестартов.
- Ресурсы: requests/limits по замеру; без limits JVM+прямой буфер Netty легко съедают ноду.
- Rolling update: `maxUnavailable` с учётом PDB; graceful shutdown Boot (`server.shutdown=graceful`) + `terminationGracePeriodSeconds` **длиннее**, чем самый длинный легитимный запрос интеграций (иначе K8s режет mid-flight).
- NetworkPolicy: Ingress/HAProxy → SCG; SCG → только нужные Service; **запрет** SCG → интернет (southbound — у Integration API).
- RBAC: если **нет** Spring Cloud Kubernetes Discovery — шлюзу **не** нужны list/watch всех Service кластера.
- Секреты: TLS и Redis ACL из Secret/Vault (у вас есть документ Vault), не в Git.
- HPA: включать **после** базового замера; масштаб без потолка пула HttpClient даёт 100 подов, которые вместе сносят origin.

#### Безопасность прода (без этого периметр не считается настроенным)

1. **Сеть**  
   - На мир: только то, что должен видеть клиент (через край).  
   - Actuator, метрики — отдельный порт/NetworkPolicy, scrape из Prometheus.  
   - Origin принимает HTTP **только** из CIDR шлюза (и mesh), не NodePort «на всякий случай».

2. **Поставка**  
   - Pin: Boot + BOM **2025.1.3** + digest образа.  
   - Не тащить Gateway **5.0.2** (CVE-2026-47879).  
   - SBOM/скан зависимостей: шлюз тянет пол-Spring Cloud транзитивно.

3. **Actuator**  
   - `management.endpoint.gateway.access=read-only` (или не экспонировать).  
   - Документация: `unrestricted` / `enabled=true` даёт create/delete/refresh — **только** за сильной аутентификацией, не с мира.  
   - Маршруты из YAML **не удаляются** через DELETE (404) — не считать actuator источником истины.

4. **TLS и заголовки**  
   - `use-insecure-trust-manager: false`.  
   - `trusted-proxies` = regex **IP края** (Ingress/HAProxy/WAF), не `.*`.  
   - Клиентский `X-Forwarded-For` с мира не должен становиться «истиной».  
   - SecureHeaders — точечно: жёсткий CSP ломает портал; HSTS — только если TLS на этом имени стабилен.

5. **Идентификация**  
   - Spring Security resource server / BFF — **до** проксирования внутренних API.  
   - Не полагаться на то, что origin «сам проверит JWT», если шлюз уже в trusted network: origin всё равно должен проверять, иначе обход шлюза = обход auth.

6. **Опасные фильтры**  
   - `RequestHeaderToRequestUri` — не на публичных маршрутах.  
   - `ModifyRequestBody` / чтение тела — память и ПДн.  
   - JSON-to-gRPC фильтр существует; после CVE-2026-47879 не включать «поиграть» без нужды.

7. **SpEL**  
   - `restrictive-property-accessor.enabled=true` (дефолт) не выключать.  
   - `key-resolver: "#{@bean}"` — только свои бины, не пользовательский ввод в SpEL.

8. **Логи**  
   - Access log содержит URL и часто токены в query — вопрос 152-ФЗ.  
   - `httpclient.wiretap` / `httpserver.wiretap` в проде **false**.

9. **CORS**  
   - Явные `allowedOrigins`, не `*` вместе с cookie.  
   - `add-to-simple-url-handler-mapping=true`, если preflight OPTIONS не попадает в предикат маршрута (штатный совет доки).

##### Сильные / слабые стороны выбранной ИБ-схемы

| Сильное | Слабое |
|---|---|
| Маршруты в Git, actuator без записи | Ошибка GitOps = общий 404/открытый лишний путь на всех ЦОДах сразу |
| trusted-proxies по доке 5.x | Неверный regex = слепой IP или поломанный Forwarded |
| Шлюз не ходит в интернет | Integration API всё равно надо защищать отдельно |
| Не ГОСТ | Если СКЗИ обязательны — vanilla не ответ |
| 5.0.3 закрывает известный CVE линии | Следующий CVE потребует того же ритуала патча BOM |

#### Порядок вывода в прод (этапы, не команда за командой)

1. Закрыть таблицу пробелов: место в цепочке края, карта маршрутов, таймауты интеграций, нужен ли Redis-лимит, топология K8s.
2. Зафиксировать 5.0.3 / BOM 2025.1.3 / Boot 4.0.8 (или явно 4.1.x) и JDK.
3. Стенд: два маршрута, проверка, что YAML 5.x биндится, StripPrefix, health.
4. Препрод: ≥2 реплики, Ingress TLS, `trusted-proxies`, actuator закрыт, NetworkPolicy.
5. Маршрут Integration API с боевым timeout; **без** Retry на POST; учение «origin 5xx / timeout».
6. Circuit breaker на 1–2 критичных сервисах; проверить, что fallback не отдаёт чужие данные.
7. Если нужен лимит — Redis HA **в тех же зонах**, KeyResolver, учение 429.
8. Раскатка по зонам/ЦОДам из одного артефакта; учение «ЦОД мёртв» на городском слое.
9. HPA только после замера p99 и FD/пула.
10. Патчи линии 5.0 — по SR поезда Oakwood, не `latest` в пятницу.

Без пунктов 4, 5 и 8 у вас нет отказоустойчивости входа.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | Не требуется | ≥2 пода, зоны/ЦОДы, PDB, probes, локальный origin, городской failover края; Redis HA если общий лимит |
| Производительность / масштаб | Не требуется | Пул HttpClient limited; таймауты по факту интеграций; горизонталь = реплики; HPA после замера; не кэшировать ПДн «на всякий» |
| Безопасность | Actuator не с мира; тестовый HTTP допустим | 5.0.3; YAML 5.x; TLS края; trusted-proxies; actuator read-only; нет SSRF-фильтров; шлюз не в интернет; секреты не в Git |

**Не готов к проду**, если: один под на три ЦОДа; `spring.cloud.gateway.routes` из гайда 4.x (пустой шлюз); `starter-gateway` без `-server-webflux`; discovery locator на всём кластере; `use-insecure-trust-manager`; `/actuator/gateway` unrestricted с мира; `RequestHeaderToRequestUri` на публичном API; Retry на POST в ведомства; шлюз сам ходит в интернет «как интеграция»; Redis-лимитер на 10 подах без Redis; origin в чужом ЦОДе без замера RTT; ждут, что Gateway заменит WAF, Kafka и Integration API; версия 5.0.2.

---

## Источники (чтобы не принимать на веру)

- Релиз поезда **2025.1.3** (20.08.2026), Boot **4.0.8**, Gateway **5.0.3**, CVE-2026-47879, изменение TimeLimiter в CircuitBreaker: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released
- Матрица поездов (Oakwood = Boot 4.0.x и 4.1.x с 2025.1.2): https://spring.io/projects/spring-cloud
- Поддержка поездов следует за поддержкой Boot: https://github.com/spring-cloud/spring-cloud-release/wiki/Supported-Versions
- Обзор продукта (Server vs Proxy Exchange, WebFlux и MVC): https://docs.spring.io/spring-cloud-gateway/reference/
- Как работает (пре/пост фильтры; path в URI маршрута игнорируется): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/how-it-works.html
- Конфигурация 5.x (`server.webflux.routes`): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html
- Appendix свойств **5.0.3** (пул elastic, connect-timeout дефолт 30s, metrics.enabled=false, trusted-proxies, redis rate limiter headers): https://docs.spring.io/spring-cloud-gateway/reference/5.0.3/appendix.html
- Таймауты HTTP (глобальные Duration vs per-route мс; −1 отключает global response-timeout): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/http-timeouts-configuration.html
- TLS/SSL (`useInsecureTrustManager` не для прода): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/tls-and-ssl.html — **в примерах этой страницы ещё встречается старый префикс `spring.cloud.gateway.httpclient`; для 5.0.3 биндинг — `spring.cloud.gateway.server.webflux.httpclient` по appendix**
- CORS: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/cors-configuration.html
- Actuator `/gateway` (access read-only vs unrestricted; YAML-маршруты не DELETE): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html
- Global filters, `lb://`, 503 vs 404, local cache, метрики: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/global-filters.html
- RequestRateLimiter, Redis token bucket, «user query param не для прода»: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/requestratelimiter-factory.html
- RequestHeaderToRequestUri — только trusted environment: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/requestheadertorequesturi-factory.html
- Forwarded/X-Forwarded требуют `trusted-proxies`: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/httpheadersfilters.html
- Java/Boot system requirements (линейка 4.1.1 на дату запроса доки: Java 17–26; для 4.0.8 сверять страницу **той** версии): https://docs.spring.io/spring-boot/system-requirements.html
- Boot 4.0.0: Java 17 + first-class 25: https://spring.io/blog/2025/11/20/spring-boot-4-0-0-available-now
- Тег Gateway 5.0.3: https://github.com/spring-cloud/spring-cloud-gateway/releases/tag/v5.0.3

Утверждения вида «N RPS на нашу JVM», «Gateway переживёт два ЦОДа сам», «discovery locator безопасен в проде», «старый YAML 4.x подхватится» в документации **отсутствуют или прямо опровергаются** — поэтому в этом файле их нет. Порог RTT, при котором шлюз ещё имеет смысл ходить в origin другой площадки, проектом **не задан**; пока замер сети не сделан, межЦОДовый origin в план не кладём.
