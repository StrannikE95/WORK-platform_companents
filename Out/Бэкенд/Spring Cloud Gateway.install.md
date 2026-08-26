# Spring Cloud Gateway 5.0.3 — установка и конфигурирование

Связанный документ (глоссарий, маршруты, SSRF, почему так): `Spring Cloud Gateway.md`.

Этот файл — **как поставить и настроить**. Stretch «сессионного кластера шлюзов» на несколько ЦОДов **не делаем**: процессам нечего согласовывать через Raft; между ЦОДами — независимые выкаты одного артефакта.

Версии: **Spring Cloud Gateway 5.0.3**, поезд **Spring Cloud 2025.1.3** (Oakwood), **Spring Boot 4.0.8** (на нём поезд **явно основан**). Стартер: `spring-cloud-starter-gateway-server-webflux`.  
Документация: https://docs.spring.io/spring-cloud-gateway/reference/ · анонс поезда: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released

Это **библиотека внутри вашего Spring Boot-приложения**, не Helm-«продукт gateway» и не Ingress-контроллер. В **5.0.3** закрыт **CVE-2026-47879** (SSRF / доступ к локальным файлам через gRPC-путь). **5.0.2 и ниже этой линии без патча в прод не брать.**

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Целевой вкус — Server WebFlux.** MVC не запрещён, но YAML/фильтры другие; оба стартера в одном процессе не смешивать. `spring-cloud-starter-gateway` без `-server-webflux` — гайды 4.x.
2. **Boot 4.0.8** как пин поезда. 4.1.x допустим матрицей Oakwood с 2025.1.2 — отдельное решение, не `latest` Boot.
3. Java — **21 или 25 LTS**. 17 — легальный минимум Boot 4.0, не лучшая цель greenfield.
4. Прод — **vanilla Kubernetes** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`). Перед шлюзом — край (HAProxy / WAF / Ingress), см. `HAProxy.install.md`.
5. Роль — **northbound HTTP** к микросервисам и Integration API. Исходящие вызовы к госслужбам шлюз **не** делает.
6. Dev — изолированная сеть. Нагрузки HTTP нет — **нет** числа реплик и ядер «хватит для прода».
7. Маршруты в Git (ConfigMap/Helm values), не `POST /actuator/gateway/routes` в бою.
8. Для 2 ЦОДов: независимый Deployment шлюза в каждом. Для 3 ЦОДов — то же в третьем. Городской failover (DNS/край) — не фича Gateway.

Критические пробелы: карта «HAProxy → WAF → Ingress → SCG»; шлюз ≠ Integration API; нужен ли общий Redis-лимит; таймауты ведомств; какие URL публичные. Пока это не закрыто, YAML ниже — каркас, не периметр.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** ходить в сервисы **теми же путями**, что в проде; поймать префикс 5.x, StripPrefix, таймауты. **Не** цель: отказ ЦОДа и RPS.

### Предпосылки

- JDK 17+ (лучше 21). Maven ≥ 3.6.3 или Gradle из system requirements **вашего** Boot 4.0.8.
- Порт 8080 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (зависимость — основной путь Dev)

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

`server.address=127.0.0.1` обязателен на Dev: иначе Boot часто слушает все интерфейсы.

Если упаковываете **свой** образ и гоняете Docker:

```bash
docker run -d --name scg-dev \
  -p 127.0.0.1:8080:8080 \
  <ваш-registry>/gateway:<digest-с-5.0.3>
```

Официального образа «Spring Cloud Gateway как продукт» нет — это ваш JAR. `-p 8080:8080` без адреса часто слушает все интерфейсы.

Проверка: `GET http://127.0.0.1:8080/actuator/gateway/routes` (только с localhost) — маршрут `demo` виден. Пустой список при живом YAML — почти всегда префикс 4.x `spring.cloud.gateway.routes`.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Одна реплика / один процесс | да | Некому делить трафик |
| TLS на входе | нет | Иначе PKI раньше путей |
| Redis rate limit | нет | Нет атаки и нет HA Redis |
| Circuit breaker | можно отложить | Таймауты лучше сразу |
| Discovery locator | **выкл** | Иначе привычка «все Service наружу» |
| `RequestHeaderToRequestUri` | **запрещён** | Готовый SSRF даже на стенде |

Чего **не** упрощать: BOM 2025.1.3 + Gateway 5.0.3; префикс `server.webflux`; реальные публичные пути и StripPrefix; таймаут маршрута Integration API (иначе край ждёт минуты, шлюз рвёт через дефолтные **30 с** connect, если не задан); клиенты ходят **на шлюз**, не в обход на под сервиса.

### Проверка Dev

1. Зависимость 5.0.3, не 5.0.2.
2. Запрос `/api/...` уходит на origin со StripPrefix; прямой обход шлюза в тесте не использовать как «успех».
3. Рестарт процесса: маршруты из YAML на месте (actuator не источник истины).

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы; модель проекта: Boot + YAML | Падение процесса = нет входа |
| Ловит ошибку префикса 5.x до прода | Не ловит HPA, Redis-лимит на N репликах, CPU TLS |
| | Успех на одном процессе ≠ готовность двух–трёх ЦОДов |

Препрод: **≥2 реплики + PDB + TLS на Ingress + боевой таймаут Integration API**, даже без боевого RPS.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой того, что городской край перестанет слать на мёртвую площадку. Шлюз **stateless**: упал под — Kubernetes поднимает другой. Цифр RPS нет.

### Почему не stretch

Несколько JVM шлюза **не** образуют кворум и не реплицируют маршруты. «Сессионный кластер» (липкость, общий in-memory кэш, общий Caffeine-лимит «на город») через ЦОДы — скрытая связность и ложное чувство HA. Согласованность конфига — **GitOps**, не Raft. Origin — **локальный** Service этой площадки: шлюз ЦОД-1 в Integration API ЦОД-2 превращает чужой RTT в деградацию **всего** входа.

### Топология

**Внутри каждого ЦОДа** — свой Deployment:

- **≥2 реплики** (лучше 3) + anti-affinity / topologySpread **внутри** площадки;
- Service + Ingress (или HAProxy края) **этого** ЦОДа;
- маршруты на `http://svc.ns.svc.cluster.local:port`, не на IP пода и не на origin чужого ЦОДа;
- `discovery.locator.enabled=false`; явный список маршрутов в Git;
- образ **вашего** приложения с BOM 2025.1.3 / Gateway 5.0.3, digest pinned.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1 и ЦОД-2: независимые Deployment **одного** артефакта. Край/DNS режет мёртвую площадку | Вход ЦОД-1 мёртв. ЦОД-2 жив, если маршруты совпали и origin локальный |
| **3 ЦОДа** | То же в ЦОД-3 | То же; третий выкат не склеивает сессии |

Local response cache (Caffeine) — RAM **пода**, не общее на площадки. Общий rate limit — только если есть **Redis HA в этом ЦОДе** (см. документ Redis); иначе лимит на WAF/HAProxy или честно per-pod (при N репликах квота ×N).

### Предпосылки прода

- Kubernetes в каждом ЦОДе. Край входа настроен (иначе шлюз — обычный ClusterIP).
- NetworkPolicy: Ingress/HAProxy → SCG; SCG → только нужные Service; **запрет** SCG → интернет (southbound — у Integration API).
- PKI на крае; к origin — доверенные сертификаты, не `use-insecure-trust-manager: true`.
- Секреты TLS/Redis — Secret/Vault, не Git.

### Установка (это не Helm-чарт вендора)

1. Собрать Boot-приложение: parent **4.0.8**, BOM **2025.1.3**, стартер **webflux**, в lockfile Gateway **5.0.3**.
2. Образ в ваш registry, не `:latest`.
3. На **каждом** кластере ЦОДа: Deployment + Service + Ingress/маршрут края. Values/overlay: URI origin **этой** площадки (имена Service могут совпадать, кластеры — нет).
4. Actuator `/gateway` — `read-only` или не экспонировать; scrape метрик — отдельный порт/NetworkPolicy.

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

Requests — после замера. Readiness без обязательного ping origin. `terminationGracePeriodSeconds` длиннее легитимного запроса интеграций. Не StatefulSet. PDB: не `replicas=0` при drain.

### Конфигурирование (смысл, не полный appendix)

| Параметр | Прод | Зачем |
|---|---|---|
| YAML | `spring.cloud.gateway.server.webflux.*` | Префикс 4.x молча пустой |
| Locator | `false` | Не публиковать все Service |
| URI маршрута | DNS Service, без path в `uri:` | Path на URI официально **игнорируется** |
| `httpclient.pool.type` | `fixed` + `max-connections` | Elastic при шторме размножает FD/OOM |
| Connect / response timeout | задать явно (дефолт connect appendix **30 с**, если не задан); Integration API — **из замера** | Внутреннему Service 30 с обычно много; короткий рвёт long-poll |
| Retry / circuit breaker | не Retry на POST в ведомства; автомат на критичных GET/синхронных | Иначе двойная заявка или пул таймаутов |
| Actuator / `trusted-proxies` / metrics | gateway `read-only`; regex **IP края**, не `.*`; `metrics.enabled=true` явно | `unrestricted` = смена маршрутов; `.*` = подмена IP; path-tag — кардинальность |

TimeLimiter Resilience4j: в поезде 2025.1.3 дефолт factory **больше не используется** — не копировать гайды 2024.

`lb://` — только с принятым DiscoveryClient (Spring Cloud Kubernetes + RBAC) **или** не использовать: DNS Service достаточен. Смешивать Eureka + K8s + хардкод — ночные 503.

### Масштабирование (когда появятся цифры)

1. Замерить RPS, размер тела, p99, долю TLS (если терминация на шлюзе, а не на крае), FD, пул HttpClient.
2. Горизонталь — реплики **внутри каждого** ЦОДа. HPA после замера; без потолка пула 100 подов сносят origin.
3. Вертикаль — CPU/RAM, куча JVM, `max-connections`. Local cache не масштабируется как общая память.
4. «Терабайты озера» на размер шлюза почти не влияют.

### Безопасность (без этого не прод)

1. Pin **5.0.3**, не **5.0.2** (CVE-2026-47879). SBOM: шлюз тянет пол-Spring Cloud.
2. На мир — только клиентский API. Actuator `/env`, `/heapdump`, запись `/gateway` — не публиковать.
3. Spring Security **до** проксирования; origin всё равно проверяет JWT (обход шлюза иначе = обход auth).
4. `RequestHeaderToRequestUri` и JSON-to-gRPC не на публичных маршрутах. SpEL/KeyResolver — свои бины. Wiretap в проде **false**. CORS без `*` с cookie. Camunda/Grafana — отдельные входы.

### Проверка прода (пока это не пройдено — это не прод)

На **каждом** кластере:

1. В lockfile / образе — Gateway 5.0.3. Маршруты из YAML 5.x видны.
2. Убить под: Service исключает его, второй принимает. Drain с PDB.
3. Origin 5xx/timeout: circuit breaker не долбит; POST в Integration API **не** ретраится.
4. Запрос с мира на `/actuator/gateway` с записью — отказ.
5. Шлюз не ходит в интернет (NetworkPolicy).

Между ЦОДами: выключить вход ЦОД-1 на препроде — ЦОД-2 отвечает **теми же** путями (GitOps), origin локальный.

### Сильные / слабые стороны (остров на ЦОД, stateless)

| Сильное | Слабое |
|---|---|
| Нет кворума через город; реплики дешёвые | Ошибка GitOps = 404/лишний путь на всех площадках сразу |
| Согласовано с запретом stretch | Городской failover — край/DNS, не Gateway |
| 5.0.3 закрывает CVE-2026-47879 линии | Следующий CVE — тот же ритуал BOM |
| | Redis-лимитер «на город» без Redis HA — самообман |

**Не готов к проду**, если: один под на площадку; YAML 4.x; стартер без `-server-webflux`; Gateway **5.0.2**; discovery locator на всём кластере; `use-insecure-trust-manager`; actuator unrestricted с мира; `RequestHeaderToRequestUri` на публичном API; Retry на POST в ведомства; шлюз сам ходит в интернет; origin в чужом ЦОДе; ждут, что библиотека заменит WAF, Kafka и Integration API.

---

## Источники

- Поезд 2025.1.3, Boot 4.0.8, Gateway 5.0.3, CVE-2026-47879: https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released
- Матрица поездов: https://spring.io/projects/spring-cloud
- Конфигурация 5.x и appendix 5.0.3: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html · https://docs.spring.io/spring-cloud-gateway/reference/5.0.3/appendix.html
- Actuator `/gateway`: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html
- Тег v5.0.3: https://github.com/spring-cloud/spring-cloud-gateway/releases/tag/v5.0.3
- Правила и пробелы: `Spring Cloud Gateway.md`

Утверждений «N RPS на нашу JVM» в документации **нет**. Stretch сессий в этой инструкции не предлагается.
