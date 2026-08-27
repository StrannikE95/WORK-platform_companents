# Spring Cloud Gateway 5.0.3 — схемы устройства

Связанные документы: правила — `Spring Cloud Gateway.md`; установка — `Spring Cloud Gateway.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Это **библиотека** в **вашем** Spring Boot **4.0.8** (поезд **2025.1.3** / Oakwood), не Helm-кластер «как Kafka».

Допущения: stretch сессии шлюза на 2–3 ЦОДа **нет** — реплики **stateless**, по Deployment **в каждом** ЦОДе. Не Ingress-контроллер и не Kafka. В **5.0.3** закрыт **CVE-2026-47879**. Нагрузки HTTP нет.

---

## 1. Контекст

Шлюз — **northbound HTTP**: маршруты к вашим сервисам. Исходящие вызовы к ведомствам делает интеграционное API, не Gateway.

```mermaid
flowchart TB
  CLI["Клиенты портала / партнёры"]
  EDGE["HAProxy / WAF / Ingress\nкрай ЦОДа"]
  SCG["Gateway 5.0.3\nвнутри Boot 4.0.8"]
  MS["Микросервисы HTTP"]
  INT["Integration API"]
  GOV["30+ госслужб"]
  KF["Kafka / Zeebe gRPC"]

  CLI --> EDGE
  EDGE --> SCG
  SCG --> MS
  SCG --> INT
  INT --> GOV
  KF -.->|"мимо шлюза"| SCG
```

Он **не** оркестрирует «какие ведомства дергать», **не** хранит досье и **не** читает Ingress-объекты Kubernetes. SecureHeaders и rate limit — не WAF.

Несколько независимых JVM с одним Git-конфигом. Согласованность маршрутов — Git/ConfigMap, не Raft.

---

## 2. Контейнеры (ваше приложение, не «кластер Gateway»)

```mermaid
flowchart TB
  subgraph dc["Один ЦОД"]
    ING["Ingress / Service"]
    subgraph dep["Deployment вашего Boot-приложения"]
      P1["под Gateway"]
      P2["под Gateway"]
    end
    ORG["Service origin этой площадки"]
  end

  RD["Redis\nтолько если общий rate limit"]
  GIT["Маршруты в Git"]

  ING --> P1
  ING --> P2
  P1 --> ORG
  P2 --> ORG
  P1 -.-> RD
  GIT --> dep
```

Deployment, не StatefulSet. `topologySpread` / anti-affinity по зоне. Origin — **локальный**: «шлюз в ЦОД-1, API только в ЦОД-3» превращает чужой RTT в деградацию всего входа.

Городской VIP / Anycast — документ HAProxy, не фича библиотеки. Реплики сами IP между площадками **не** таскают.

Стартер прода: `spring-cloud-starter-gateway-server-webflux`. MVC — другой YAML и фильтры; оба стартера в одном процессе **нельзя**. Старый префикс `spring.cloud.gateway.routes` в 5.x **молча не биндится**.

---

## 3. Компоненты внутри процесса

```mermaid
flowchart LR
  REQ["HTTP вход"] --> PR["Предикаты\npath / host / AND"]
  PR --> RT["Маршрут id + URI"]
  RT --> FL["Фильтры\nStripPrefix, retry, CB"]
  FL --> HC["Netty HttpClient\nк origin"]
```

| Компонент | Для чего настраивать |
|---|---|
| Route + predicates | Контракт входа. Path в `uri:` маршрута официально **игнорируется** |
| GatewayFilter | StripPrefix, заголовки, rate limit, circuit breaker, retry |
| GlobalFilter | На все маршруты: `lb://`, запись ответа |
| HttpClient pool | Дефолт `elastic`; в проде `fixed` + `max-connections` |
| Circuit breaker | Фильтр; реализация — Spring Cloud CircuitBreaker (Resilience4j из того же BOM) |
| Actuator `/actuator/gateway` | Дефолт запись **выключена**. `unrestricted` = менять маршруты на живую |

`discovery.locator` дефолт **выключен**. Включить «чтобы не писать YAML» = выставить в интернет все найденные Service. `lb://` — не магия Kubernetes: нужен LoadBalancer/DiscoveryClient **или** DNS Service `http://name.ns.svc`.

---

## 4. Поток запроса

```mermaid
sequenceDiagram
  participant C as Клиент
  participant E as Край Ingress
  participant G as Под Gateway
  participant O as Origin
  participant CB as Circuit breaker

  C->>E: HTTPS
  E->>G: HTTP + X-Forwarded
  G->>G: предикат, фильтры pre
  G->>O: HttpClient
  alt origin сыплет ошибками
    O-->>G: 5xx / timeout
    G->>CB: открыть автомат
    CB-->>C: fallback или ошибка шлюза
  else ок
    O-->>G: ответ
    G-->>C: фильтры post
  end
```

Упал под — клиент на этом TCP получает обрыв; это норма HTTP-прокси. Retry на **GET** иногда уместен. Retry на **POST в ведомство** без идемпотентности = двойная заявка. Не копировать старые гайды TimeLimiter: в поезде 2025.1.3 дефолт factory **больше не используется**.

---

## 5. Отказоустойчивость — что настраивать

```mermaid
flowchart TB
  subgraph must["Внутри ЦОДа"]
    R2[">=2 пода + PDB"]
    PRB["probes без ping origin"]
    CB["CB + timeout на маршруте"]
    LOC["origin этой площадки"]
  end

  subgraph city["Между ЦОДами"]
    DNS["DNS / Anycast / край"]
  end

  must -->|"падение пода"| OK["Service исключает"]
  must -->|"падение ЦОДа"| city
```

| Ручка | Если забыть |
|---|---|
| ≥2 реплики + PDB | Drain узла = нет входа |
| Readiness ≠ обязательный ping origin | Каскад рестартов, когда бэкенд тупит |
| `trusted-proxies` = IP края, не `.*` | Либо слепой IP, либо клиент подделывает XFF |
| Circuit breaker на критичном маршруте | Шлюз копит таймауты и жжёт пул |
| Redis HA, если общий лимит | 10 реплик Caffeine = квота ×10; падение Redis = дыра или 429 |
| Pin **5.0.3** | 5.0.2 и ниже линии — CVE-2026-47879 (SSRF / файлы на gRPC-пути) |

Один под на три ЦОДа отказоустойчивости **не даёт**. Шлюз в других залах о мёртвой площадке **не договаривается**.

---

## 6. Масштабируемость — что настраивать

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> CPU["CPU TLS / куча JVM"]
  Q --> RPS["RPS / долгие запросы"]
  Q --> POOL["Пул HttpClient / FD"]

  CPU --> V["Вертикаль пода; TLS лучше на крае"]
  RPS --> H["Реплики + HPA после замера"]
  POOL --> F["pool type fixed"]
```

«Терабайты озера» на размер шлюза почти не влияют. Влияют RPS, размер тела, число висящих запросов, retry (умножает нагрузку на origin). LocalResponseCache — RAM **пода**, не общее на три ЦОДа.

HPA без потолка пула даёт сто подов, которые вместе сносят origin. Цифр RPS у проекта Gateway **нет**.

---

## 7. 2–3 ЦОДа без stretch

```mermaid
flowchart LR
  subgraph a["ЦОД-1"]
    G1["Реплики SCG\norigin локальный"]
  end
  subgraph b["ЦОД-2"]
    G2["Тот же Git-артефакт"]
  end
  subgraph c["ЦОД-3"]
    G3["Тот же артефакт"]
  end
  EDGE["Городской слой"] --> G1
  EDGE --> G2
  EDGE --> G3
```

**Сильное:** процесс почти stateless; отказ зоны переживается репликами в других ЦОДах, если край перестаёт слать на мёртвую.  
**Слабое:** ошибка GitOps = 404 или лишний путь **на всех** площадках сразу; общий Redis-лимитер — отдельная HA-задача; межЦОДовый origin проектом **не нормирован**.

---

## 8. Безопасность на той же картине

1. На мир — только публичные пути через край. Actuator, `/env`, heapdump — не с интернета. `/gateway` = `read-only` или выкл.
2. `RequestHeaderToRequestUri` — только доверенная среда, не публичный маршрут.
3. NetworkPolicy: шлюз → нужные Service; **запрет** шлюз → интернет (southbound у Integration API).
4. `use-insecure-trust-manager: false`. Wiretap Netty в проде **false**.
5. Не тащить 5.0.2.

Источники: `Spring Cloud Gateway.md`. Порога RTT «шлюз в чужой ЦОД» у Spring **нет** — на схемах межЦОДовый origin не целевой.
