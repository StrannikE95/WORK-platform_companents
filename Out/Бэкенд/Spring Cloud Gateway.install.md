# Spring Cloud Gateway 5.0.3 — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `Spring Cloud Gateway.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

Spring Cloud Gateway — HTTP-шлюз. Это **библиотека внутри вашего Spring Boot-приложения**, не готовый продукт с Helm-чартом вендора и не Ingress-контроллер. Официального образа на Docker Hub нет: собираете свой JAR и свой образ.

Версия в этой инструкции: **Gateway 5.0.3**, поезд **Spring Cloud 2025.1.3** (Oakwood), **Spring Boot 4.0.8** (на нём поезд **явно основан**). Стартер: `spring-cloud-starter-gateway-server-webflux`. С **2025.1.2** матрица Oakwood допускает и Boot **4.1.x** — это отдельное решение, не `latest` Boot.

Документация: https://docs.spring.io/spring-cloud-gateway/reference/  
Анонс поезда: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released

В **5.0.3** закрыт **CVE-2026-47879** (SSRF / доступ к локальным файлам через gRPC-путь). **5.0.2 и ниже этой линии без патча в бой не брать.**

«Сессионный кластер шлюзов» на несколько дата-центров здесь **не собираем**: процессам нечего согласовывать через Raft. Между площадками — независимые выкаты **одного** артефакта. Origin — локальный Service этой площадки.

---

## О чём эта инструкция молча договорилась

1. Целевой вкус — **Server WebFlux**. MVC не запрещён, но YAML и фильтры другие; оба стартера в одном процессе не смешивать. `spring-cloud-starter-gateway` без `-server-webflux` — гайды 4.x.
2. Boot **4.0.8** как пин поезда. Java — **21 или 25 LTS**. 17 — легальный минимум Boot 4.0, не лучшая цель для нового контура.
3. Бой — Kubernetes в каждом дата-центре отдельно (см. `Kubernetes.install.md`). Перед шлюзом — край (HAProxy / WAF / Ingress), см. `HAProxy.install.md`, если есть.
4. Роль — **вход HTTP** к микросервисам и Integration API. Исходящие вызовы к госслужбам шлюз **не** делает.
5. Учебный стенд — закрытая сеть. Один процесс и HTTP без TLS допустимы **только там**.
6. Цифр вашей нагрузки нет — нет фразы «хватит N реплик». Сначала два пода на площадку; HPA — после замера.
7. Маршруты в Git (ConfigMap / Helm values), не `POST /actuator/gateway/routes` в бою.
8. Два дата-центра: независимый Deployment в каждом. Три — то же в третьем. Городской failover (DNS / край) — не фича Gateway. Третья площадка **не** склеивает сессии и не делает общий Caffeine-кэш.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** ходить в сервисы **теми же путями**, что в бою; поймать префикс YAML 5.x, StripPrefix, таймауты. **Не зачем:** доказывать отказ дата-центра и запросы в секунду.

### Что должно быть до установки

- JDK 17+ (лучше 21). Maven ≥ 3.6.3 или Gradle из system requirements **вашего** Boot 4.0.8.
- Порт 8080 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (зависимость Maven — основной путь для учёбы)

Родитель и BOM (версии **пинить**, не диапазон):

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>4.0.8</version>
</parent>
```

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2025.1.3</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
</dependency>
```

Версию стартера **не** дублировать: её даёт BOM. После сборки в дереве зависимостей должна быть Gateway **5.0.3**, не 5.0.2.

Минимальные маршруты (`application.yml`). Префикс **5.x WebFlux** — иначе блок молча не биндится:

```yaml
server:
  address: 127.0.0.1
  port: 8080
spring:
  cloud:
    gateway:
      server:
        webflux:
          discovery:
            locator:
              enabled: false
          routes:
            - id: demo
              uri: http://127.0.0.1:8081
              predicates:
                - Path=/api/**
              filters:
                - StripPrefix=1
management:
  endpoint:
    gateway:
      access: read-only
```

`server.address=127.0.0.1` обязателен на стенде: иначе Boot часто слушает все интерфейсы.

Если упаковываете **свой** образ и гоняете Docker:

```bash
docker run -d --name scg-dev \
  -p 127.0.0.1:8080:8080 \
  <ваш-registry>/gateway:<digest-с-5.0.3>
```

`-p 8080:8080` без адреса часто слушает все интерфейсы.

Проверка: `GET http://127.0.0.1:8080/actuator/gateway/routes` (только с localhost) — маршрут `demo` виден. Пустой список при живом YAML — почти всегда префикс 4.x `spring.cloud.gateway.routes`.

На Kubernetes для учёбы: один Deployment с одной репликой, без Redis. Этот YAML в бой не копируйте.

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Одна реплика / один процесс | да | Некому делить трафик |
| TLS на входе | нет | Иначе сертификаты раньше путей |
| Redis rate limit | нет | Нет атаки и нет отказоустойчивого Redis |
| Circuit breaker | можно отложить | Таймауты лучше сразу |
| Discovery locator | **выкл** | Иначе привычка «все Service наружу» |
| `RequestHeaderToRequestUri` | **запрещён** | Готовый SSRF даже на стенде |

Чего **не** упрощаем: BOM 2025.1.3 + Gateway 5.0.3; префикс `server.webflux`; реальные публичные пути и StripPrefix; таймаут маршрута Integration API (иначе край ждёт минуты, шлюз рвёт через заводские **30 с** connect, если не задан); клиенты ходят **на шлюз**, не в обход на под сервиса.

### Как понять, что стенд живой

1. В дереве зависимостей / образе — Gateway 5.0.3, не 5.0.2.
2. Запрос `/api/...` уходит на origin со StripPrefix; прямой обход шлюза в тесте не использовать как «успех».
3. Процесс перезапустили — маршруты из YAML на месте (actuator не источник истины).

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Часы; модель проекта: Boot + YAML | Падение процесса = нет входа |
| Ловит ошибку префикса 5.x до боя | Не ловит HPA, Redis-лимит на N репликах, CPU TLS |
| | Успех на одном процессе ≠ готовность двух–трёх дата-центров |

Перед боем полезен **препрод**: не меньше 2 реплик + PDB + TLS на Ingress + боевой таймаут Integration API — всё **в одном** дата-центре, даже без боевого трафика.

---

## Бой: один живой дата-центр, второй — независимый выкат

**Зачем:** пережить отказ **пода внутри площадки** (два или три пода, PDB, probes). Отказ **всего дата-центра** = входа этой площадки нет, пока городской край (DNS / Anycast / HAProxy) не перестанет слать сюда. Шлюз **без состояния**: упал под — Kubernetes поднимает другой. Цифр запросов в секунду нет.

### Почему «кластер шлюзов» не растягиваем на несколько дата-центров

Несколько JVM шлюза **не** образуют кворум и не реплицируют маршруты. «Сессионный кластер» (липкость, общий кэш в памяти, общий Caffeine-лимит «на город») через площадки — скрытая связность и ложное чувство отказоустойчивости. Согласованность конфига — **GitOps**, не Raft. Origin — **локальный** Service этой площадки: шлюз дата-центра 1 в Integration API дата-центра 2 превращает чужую задержку сети в деградацию **всего** входа.

### Как расставить в активном дата-центре

Свой Deployment:

- **Не меньше 2 реплик** (лучше 3) + anti-affinity / topologySpread **внутри** площадки.
- Service + Ingress (или HAProxy края) **этого** дата-центра.
- Маршруты на `http://svc.ns.svc.cluster.local:port`, не на IP пода и не на origin чужой площадки.
- `discovery.locator.enabled=false`; явный список маршрутов в Git.
- Образ **вашего** приложения с BOM 2025.1.3 / Gateway 5.0.3, digest pinned. Не 5.0.2 и не `:latest`.
- Deployment, не StatefulSet.
- PDB: не допускать `replicas=0` при drain узла.
- Readiness: не готов — нет трафика. Liveness: не убивать под из-за одного медленного origin, если health включает обязательный ping origin — получите каскад рестартов.
- `server.shutdown=graceful` + `terminationGracePeriodSeconds` **длиннее**, чем самый длинный легитимный запрос интеграций (иначе Kubernetes режет запрос на полуслове).
- NetworkPolicy: Ingress/HAProxy → шлюз; шлюз → только нужные Service; **запрет** шлюз → интернет (выход к ведомствам — у Integration API).
- RBAC: если **нет** Spring Cloud Kubernetes Discovery — шлюзу **не** нужны list/watch всех Service кластера.

**Между площадками:**

| Сколько площадок | Что где | Если умер активный дата-центр |
|---|---|---|
| **Две** | Первая и вторая: независимые Deployment **одного** артефакта. Край / DNS режет мёртвую площадку | Вход первой площадки мёртв. Вторая жива, если маршруты совпали и origin локальный |
| **Три** | То же в третьей | То же. Три выката не склеивают сессии и не делают общий кэш ответов |

Local response cache (Caffeine) — память **пода**, не общее на площадки. Общий rate limit — только если есть **Redis в этом дата-центре** (см. документ Redis); иначе лимит на WAF/HAProxy или честно на под (при N репликах квота ×N). Redis шлюза ≠ Redis сессий приложения «на одной базе SELECT 2», если цель — Cluster: в Redis Cluster живёт только DB 0.

Шлюз как единственный публичный процесс без HAProxy/Ingress возможен на маленьком контуре. На нескольких площадках вы тащите на JVM то, что лучше делает край (VIP, PROXY protocol, drain). Для заявленной платформы **не** рекомендуется как единственный слой.

### Что должно быть до боевой установки

- Kubernetes активной площадки. Край входа настроен (иначе шлюз — обычный ClusterIP).
- Карта маршрутов: какие URL публичные, какие только из VPN.
- PKI на крае; к origin — доверенные сертификаты, не `use-insecure-trust-manager: true`.
- Секреты TLS / Redis — Secret / Vault, не Git.
- BOM 2025.1.3 и Gateway 5.0.3 уже прогнаны на стенде: YAML 5.x биндится, StripPrefix работает.

### Порядок установки в активном дата-центре

Это этапы, не замена руководства вашей сборки:

1. Собрать Boot-приложение: parent **4.0.8**, BOM **2025.1.3**, стартер **webflux**, в lockfile Gateway **5.0.3**.
2. Образ в ваш registry, не `:latest`.
3. На кластере площадки: Deployment + Service + Ingress / маршрут края. Values / overlay: URI origin **этой** площадки (имена Service могут совпадать, кластеры — нет).
4. Actuator `/gateway` — `read-only` или не экспонировать; scrape метрик — отдельный порт / NetworkPolicy.

Смысл Deployment (не полный манифест):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-gateway
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: gateway
          image: <registry>/gateway:<digest>   # 5.0.3 внутри, не 5.0.2
          ports:
            - containerPort: 8080
```

Requests — после замера. Не StatefulSet. PDB: не `replicas=0` при drain.

5. Задать `trusted-proxies` = regex **IP края** (Ingress / HAProxy / WAF), не `.*`.
6. Маршрут Integration API с боевым timeout; **без** Retry на POST; учение «origin 5xx / timeout».
7. Circuit breaker на 1–2 критичных сервисах; проверить, что fallback не отдаёт чужие данные.
8. Если нужен общий лимит — Redis в **этой** площадке, осмысленный KeyResolver (не query-параметр `user`), учение 429.
9. Раскатка по площадкам из одного артефакта. HPA только после замера p99 и пула соединений.

### Правила конфигурации боя

| Параметр | В бою | Зачем |
|---|---|---|
| YAML | `spring.cloud.gateway.server.webflux.*` | Префикс 4.x молча пустой |
| Locator | `false` | Не публиковать все Service |
| URI маршрута | DNS Service, без path в `uri:` | Path на URI официально **игнорируется** |
| `httpclient.pool.type` | `fixed` + `max-connections` | Elastic при шторме размножает соединения и память |
| Connect / response timeout | задать явно (завод connect appendix **30 с**, если не задан); Integration API — **из замера** | Внутреннему Service 30 с обычно много; короткий рвёт long-poll |
| Retry / circuit breaker | не Retry на POST в ведомства; автомат на критичных GET / синхронных | Иначе двойная заявка или пул таймаутов |
| Actuator | gateway `read-only`; `/env`, `/heapdump` не публиковать | `unrestricted` = смена маршрутов с мира. YAML-маршруты через DELETE не удаляются (404) — не считать actuator источником истины |
| `trusted-proxies` | regex **IP края**, не `.*` | Иначе подмена IP или слепой Forwarded |
| Метрики | `metrics.enabled=true` явно | Завод выключен; path-tag — взрыв кардинальности |
| `use-insecure-trust-manager` | `false` | Примеры TLS не для боя |
| `lb://` | только с принятым DiscoveryClient | Иначе DNS Service достаточен. Смешивать Eureka + K8s + хардкод — ночные 503 |
| TimeLimiter Resilience4j | не копировать гайды 2024 | В поезде 2025.1.3 дефолт factory **больше не используется** |
| CORS | явные `allowedOrigins`, не `*` вместе с cookie | `add-to-simple-url-handler-mapping=true`, если preflight OPTIONS не попадает в предикат маршрута |
| SpEL | `restrictive-property-accessor.enabled=true` (завод) не выключать | KeyResolver — только свои бины |
| Wiretap | `false` | ПДн и токены в логе |
| `RequestHeaderToRequestUri` / JSON-to-gRPC | не на публичных маршрутах | SSRF; gRPC-путь как раз патчили в 5.0.3 |

Spring Security **до** проксирования; origin всё равно проверяет JWT (обход шлюза иначе = обход входа). Camunda / Grafana — отдельные входы.

### Как расти, когда появятся цифры нагрузки

1. Замерить запросы в секунду, размер тела, p99, долю TLS (если терминация на шлюзе, а не на крае), пул HttpClient, открытые файлы.
2. Горизонталь — реплики **внутри каждого** дата-центра. HPA после замера; без потолка пула 100 подов сносят origin.
3. Вертикаль — CPU / RAM, куча JVM, `max-connections`. Local cache не масштабируется как общая память.
4. «Терабайты озера» на размер шлюза почти не влияют.

### Проверки, без которых это ещё не бой

На **каждой** площадке:

1. В lockfile / образе — Gateway 5.0.3. Маршруты из YAML 5.x видны.
2. Выключили один под: Service исключает его, второй принимает. Drain с PDB.
3. Origin 5xx / timeout: circuit breaker не долбит; POST в Integration API **не** ретраится.
4. Запрос с мира на `/actuator/gateway` с записью — отказ.
5. Шлюз не ходит в интернет (NetworkPolicy). Нет `guest`-привычек вроде insecure trust manager и `RequestHeaderToRequestUri`.

Между площадками: выключить вход первой на препроде — вторая отвечает **теми же** путями (GitOps), origin локальный.

### Что хорошо и что плохо в схеме «остров на дата-центр, без состояния»

| Хорошо | Плохо |
|---|---|
| Нет кворума через город; реплики дешёвые | Ошибка GitOps = 404 / лишний путь на всех площадках сразу |
| Согласовано с запретом «сессионного кластера» | Городской failover — край / DNS, не Gateway |
| 5.0.3 закрывает CVE-2026-47879 линии | Следующий CVE — тот же ритуал BOM |
| | Redis-лимитер «на город» без Redis на площадке — самообман |

**Не готово к бою**, если: один под на площадку; YAML 4.x; стартер без `-server-webflux`; Gateway **5.0.2**; discovery locator на всём кластере; `use-insecure-trust-manager`; actuator unrestricted с мира; `RequestHeaderToRequestUri` на публичном API; Retry на POST в ведомства; шлюз сам ходит в интернет; origin в чужом дата-центре; ждут, что библиотека заменит WAF, Kafka и Integration API; учебный манифест уехал в бой.

---

## Откуда цифры и имена артефактов

- Поезд 2025.1.3, Boot 4.0.8, Gateway 5.0.3, CVE-2026-47879: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released
- Матрица поездов: https://spring.io/projects/spring-cloud
- Конфигурация 5.x и appendix 5.0.3: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html · https://docs.spring.io/spring-cloud-gateway/reference/5.0.3/appendix.html
- Actuator `/gateway`: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html
- Тег v5.0.3: https://github.com/spring-cloud/spring-cloud-gateway/releases/tag/v5.0.3
- Правила и схема компонентов: `Spring Cloud Gateway.md`

Утверждений «N запросов в секунду на нашу JVM» в документации **нет**. Stretch сессий в этой инструкции не предлагается.
