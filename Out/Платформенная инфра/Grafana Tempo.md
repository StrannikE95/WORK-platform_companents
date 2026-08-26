# Grafana Tempo 3.0.3 — развёртывание и настройка

Версия ПО: **Grafana Tempo 3.0.3** (патч линии 3.0, опубликован 13 августа 2026; на дату подготовки документа это последний стабильный патч линии 3.0).  
Документация: https://grafana.com/docs/tempo/latest/  
Релиз 3.0: https://grafana.com/docs/tempo/latest/release-notes/v3-0/  
Образ: `grafana/tempo:3.0.3` (Docker Hub / Grafana).  
На Kubernetes официальные пути: Helm-чарт **`tempo-distributed`** (community-репозиторий `grafana-community/helm-charts`) и **Tempo Operator** (`TempoStack`). Чарты Tanka/Jsonnet — эквивалент Helm для тех, кто так привык.

Линия 3.0 — major с ломающей архитектурой: ingester и compactor **удалены**. Писать прод на 2.x «потому что привычнее» — значит сразу закладывать миграцию без in-place downgrade. В прод берём **уже вышедший патч 3.0.3**, не `latest` и не 2.10.x.

Helm-чарт `tempo-distributed` **3.0.6** (17 июля 2026) по умолчанию тянет **Tempo 3.0.2**. Образ нужно **переопределить на 3.0.3**: в 3.0.3 закрыты CVE через обновление Go 1.26.5, gRPC, `x/net`, `x/text` и OpenTelemetry. Чарты Grafana после 30 января 2026 живут в `grafana-community/helm-charts`, не в старом `grafana/helm-charts`.

Этот текст — не мануал «скопируй values.yaml», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

Tempo **не было** в исходном описании архитектуры (Kafka, Camunda, озеро данных, интеграционное API). Ниже — как поставить бэкенд распределённой трассировки на ту же инфраструктуру с Kubernetes и тремя ЦОДами. Это не замена логов (Loki), метрик (Prometheus/Mimir), SIEM (Wazuh) и не «источник истины» клиентских данных.

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **Трейс (trace)** | История одного запроса через систему: «клиент вызвал API → API дернул Kafka → Camunda стартовала процесс → интеграция ждала ведомство». Рисуется как цепочка. |
| **Спан (span)** | Один шаг внутри трейса: вызов сервиса, запрос в БД, consume из Kafka. У спана есть имя, время, атрибуты (теги). |
| **Trace ID** | Уникальный номер трейса. По нему Tempo умеет **достать трейс целиком** очень дёшево. Это главная суперсила Tempo. |
| **Span ID** | Номер конкретного шага. Родительский span ID склеивает дерево. |
| **OTLP** | OpenTelemetry Protocol — современный способ слать трейсы. gRPC обычно порт **4317**, HTTP — **4318**. |
| **OpenTelemetry Collector** | Отдельная программа-посредник: принимает трейсы от приложений, батчит, сэмплирует, режет PII, отдаёт в Tempo. **Не часть Tempo**, но в проде без него почти всегда плохо. |
| **Сэмплирование (sampling)** | «Пишем не каждый трейс, а каждый N-й / только ошибочные / только медленные». Tempo **сам по себе сэмплирование не делает** — это Collector или SDK. |
| **TraceQL** | Язык поиска по трейсам (как LogQL у Loki). «Найди все трейсы сервиса X дольше 2 секунд». Дороже, чем lookup по Trace ID. |
| **TraceQL metrics** | С 3.0 — GA: из трейсов на лету считаются метрики (`rate()`, средняя длительность). Алертинг по ним — **experimental**. |
| **Монолитный режим (monolithic)** | Один процесс (`-target=all`) делает всё. Для теста. Kafka **не нужен**. Прод Grafana так ставить не рекомендует. |
| **Режим микросервисов** | Каждый компонент — отдельный процесс со своим `-target`. Для прода. **Нужен Kafka-совместимый брокер.** |
| **Distributor** | Вход: принимает OTLP/Jaeger/Zipkin, проверяет лимиты, в проде пишет в Kafka. |
| **Kafka (для Tempo)** | Не ваша бизнес-шина, а **буфер записи** (write-ahead log) между входом и остальными компонентами. Пока Kafka не подтвердила запись — клиенту «успех» не возвращают. |
| **Live-store** | Держит **свежие** трейсы в памяти + локальный WAL, чтобы их можно было искать через секунды, пока блок ещё не в object storage. |
| **Block-builder** | Читает Kafka, собирает Apache Parquet-блоки, кладёт их в object storage. Пишет каждый блок **один раз** (RF1). |
| **Object storage** | Долгое хранилище трейсов: S3 / GCS / Azure Blob / S3-совместимое. Без него прод-микросервисы Tempo **не живут**. |
| **Блок (block)** | Файл (набор файлов) с куском трейсов в формате Parquet. Retention удаляет старые блоки. |
| **vParquet4 / vParquet5** | Версии формата блоков. Tempo 3.0 **не читает** vParquet3 и старше. В манифесте дефолт на дату документа — **vParquet4**. |
| **Backend scheduler** | Один процесс, который **раздаёт работы**: компакция, retention, подчистка, redaction. Официально: **одновременно должен работать только один**. |
| **Backend worker** | Исполняет работы scheduler. Их можно несколько. |
| **Компакция (compaction)** | Склеивание мелких блоков в крупные. Без неё поиск и список блоков деградируют. |
| **Retention / `block_retention`** | Сколько хранить трейсы. Дефолт манифеста Tempo: **336 часов = 14 дней**. |
| **Partition ring** | Кольцо «какие партиции Tempo существуют и кто ими владеет». Расходится по кластеру через **memberlist** (gossip). |
| **Memberlist** | Служебный протокол «кто жив в кластере». Порт по умолчанию **7946**. |
| **Zone-aware live-store** | Live-store'ы разложены по зонам (ЦОДам): на каждую партицию — **по одному владельцу в каждой зоне**. Чтение свежих данных: кворум **1** (достаточно одного живого). |
| **RF1 (replication factor 1)** | На write path Tempo **не** копирует трейсы на три инстанса, как в 2.x. Долговечность ingest — у Kafka; долговечность истории — у object storage. |
| **Querier / query-frontend** | Поиск: frontend режет запрос на куски, querier ходит в live-store и в object storage. |
| **Memcached** | Внешний кэш (bloom, parquet-footer, результаты поиска). В примерах Helm/Tanka он есть. Redis для кэша Tempo — **experimental**. |
| **Metrics-generator** | Опционально: из спанов делает Prometheus-метрики (spanmetrics, service graph). В 3.0 тоже читает из Kafka. |
| **Мультитенантность** | Изоляция «организация A не видит трейсы B» по заголовку `X-Scope-OrgID`. Клиентов, которые сами ставят этот заголовок, Tempo **считает доверенными**. |
| **Gateway** | Обратный прокси перед Tempo (в чарте — опция). Сам Tempo **аутентификации не имеет**. |
| **WAL** | Write-ahead log на диске live-store / block-builder: чтобы после рестарта пода не забыть свежие данные, пока Kafka/object storage не подхватили. |
| **Vulture** | Тестовый клиент Grafana: пишет синтетические трейсы и читает их обратно. Учение, не нагрузка прода. |

---

## Основные элементы системы и зависимости

### Что входит в Grafana Tempo 3.0.3 (это одно ПО из нескольких ролей)

Один бинарник, разные `-target`:

| Роль | Зачем | Как масштабируется |
|---|---|---|
| **Distributor** | Приём спанов, лимиты, запись в Kafka (или in-process в монолите) | Горизонтально: больше реплик |
| **Live-store** | Свежие трейсы (минуты–около часа) | По числу партиций × число зон |
| **Block-builder** | Сборка блоков → object storage | Обычно 1 реплика на Kafka-партицию (`partitions_per_instance: 1`) |
| **Query-frontend** | API поиска, нарезка запросов | 2+ реплики для HA |
| **Querier** | Исполнение кусков запроса | Горизонтально под объём поиска |
| **Backend scheduler** | Очередь компакции/retention/redaction | **Ровно 1** |
| **Backend worker** | Исполнение этих работ | Горизонтально |
| **Metrics-generator** | Метрики из трейсов | Опционально, отдельно |

Плюс **не Tempo**, но без них прод-схема не закрывается:

- **Kafka-совместимый брокер** (Apache Kafka / Redpanda / WarpStream) — только микросервисный режим.
- **Object storage** (S3 / GCS / Azure / S3-совместимое).
- **Memcached** (кэш чтения).
- **OpenTelemetry Collector** (сэмплирование, PII, батч, `X-Scope-OrgID`).
- **Grafana** (UI). У Tempo своего «дашборда как у Wazuh» нет.
- **Аутентифицирующий reverse proxy** (HAProxy / nginx / OAuth2-proxy / gateway чарта). Цитата документации: *Grafana Tempo does not come with any included authentication layer.*

### Официальные порты (менять можно, но это контракт сети)

| Компонент | Порт | Назначение |
|---|---|---|
| Distributor | **4317/TCP** | OTLP gRPC |
| Distributor | **4318/TCP** | OTLP HTTP |
| Любой компонент | **3200/TCP** | HTTP API Tempo (`/api/...`, `/metrics`) |
| Любой компонент | **9095/TCP** | Внутренний gRPC |
| Memberlist | **7946/TCP+UDP** | Gossip кольца |
| Memcached | **11211/TCP** | Кэш (отдельное ПО) |

С Tempo **2.7** OTLP-приёмник по умолчанию слушает **localhost**, не `0.0.0.0`. В Docker/Kubernetes endpoint надо явно биндить, иначе «Collector шлёт, Tempo пустой».

Jaeger/Zipkin-приёмники в конфигурации ещё есть. OpenCensus в 3.0 **убран**. Практичный контракт для новой системы: **только OTLP**.

### Чего в Tempo нет (частая путаница)

| Нужно системе | Это не Tempo | Зачем помнить |
|---|---|---|
| Логи приложений | Loki / файлы / SIEM | Трейс — дерево вызовов, не текст лога |
| Метрики инфраструктуры | Prometheus / Mimir | Metrics-generator и TraceQL metrics — про спаны, не про CPU ноды |
| Шина бизнес-событий | Kafka из вашего event-sourcing | Tempo **потребляет** отдельный Kafka-топик как WAL. Это не топик `client.updated` |
| Источник истины по клиентам | Озеро данных | В трейсе может случайно оказаться ФИО/ИНН — это утечка, не SoT |
| UI поиска | Grafana (datasource Tempo) | Без Grafana вы смотрите HTTP API |
| Аутентификация / RBAC на API | Gateway / IdP | Заголовок тенанта без прокси = любой, кто дотянулся, читает все трейсы |
| Сэмплирование на входе | OTel Collector / SDK | 100% трейсов при «терабайтах» съест диск и Kafka раньше, чем вы заметите |
| Гарантия «трейс на каждую транзакцию Camunda навсегда» | Политика retention + сэмплирование | Дефолт хранения **14 дней** |
| Два активных backend-scheduler | Нет. Цитата конфигурации: *Only one scheduler should be running at a time* | Падение этого пода = останавливается компакция/retention, **не** приём трейсов |
| Несколько монолитных инстансов как кластер | В 3.0.3 из документации **убрали** такой совет | Монолит — один процесс. HA = микросервисы |
| ГОСТ TLS / СКЗИ | Не заявлено | Штатный TLS — не криптография по требованиям КИИ, если ИБ их выдвинет |
| Официальный порог RTT между ЦОДами | Нет | Memberlist, Kafka ingest и object storage все чувствительны к сети; миллисекунд в доке Tempo нет |
| Гарантия «терабайты трейсов влезут» | Нет | Считают от **MB/s на пике** × retention × коэффициент сэмплирования. Нагрузки у вас нет |

### Зависимости окружения (обязательны)

- **Kubernetes ≥ 1.25** для чарта `tempo-distributed` 3.0.6 (требование чарта). У вас уже есть отдельный документ по Kubernetes.
- **Helm 3+**, если идёте чартом. Оператор — если хотите CRD `TempoStack`.
- **CSI-тома RWO** под WAL live-store и block-builder. Без PVC рестарт пода = дырка в «свежих» трейсах до реплея из Kafka (а Kafka retention тоже конечен).
- **Object storage**, доступный **из всех ЦОДов**, где крутятся querier/block-builder/backend-worker. Это отдельный кластер хранения, не PVC Tempo.
- **Kafka-совместимый брокер** для микросервисного режима: топик ingest, RF как у боевой Kafka (см. `Apache Kafka.md`), квоты, ACL.
- **Прямая связность** distributor → Kafka, querier → live-store (gRPC), все → object storage, все gossip-члены → 7946.
- **Часы (NTP).** TraceQL и окна компакции завязаны на время спанов.
- **PKI**, если включаете TLS на server/OTLP/gRPC (сертификаты с SAN=DNS).
- **Исходящая сеть приложений** только до Collector (или до gateway), не напрямую в object storage.

### Как Tempo стыкуется с вашей архитектурой

```
Микросервисы / Camunda / интеграционное API / брокеры Kafka
        │  OTLP (SDK или sidecar)
        ▼
   OpenTelemetry Collector   ← сэмплирование, PII, tenant header, TLS
        │  OTLP 4317/4318
        ▼
   Tempo distributor
        │
        ├─ monolithic: in-process → live-store → (потом) object storage
        │
        └─ microservices:
              ▼
         Kafka-топик ingest (WAL Tempo, не бизнес-события)
              │
              ├─ live-store (свежие запросы)
              ├─ block-builder → object storage (история)
              └─ metrics-generator (опционально)
                    ▲
                    │  TraceQL / Trace ID
              query-frontend → querier
                    ▲
                 Grafana
```

Tempo **наблюдает** вызовы. Он не оркестрирует Camunda, не ходит в госслужбы вместо интеграционного API и не хранит эталон клиента. Если в спаны положить тело ответа ведомства или карточку клиента — Tempo станет нелегальным архивом ПДн с дефолтным поиском по атрибутам.

Пересечение с уже описанной Kafka: **два разных использования одного протокола**. Боевой кластер событий и ingest Tempo можно физически разделить (рекомендуемое допущение ниже) или изолировать топиком+квотами+ACL. Смешивать «на один топик, разберёмся» нельзя: Tempo по умолчанию готов **автосоздать топик на 1000 партиций**.

---

## Краткие вводные

### Зачем вам Tempo в этой архитектуре

У вас микросервисы на событиях, процессный движок, 30+ интеграций с **временными лагами**. Когда «заявка зависла», логи одного пода не показывают цепочку. Трейс как раз про это:

1. Связать HTTP интеграционного API → запись в Kafka → воркер Camunda → ожидание ведомства → ответ.
2. Найти **конкретный** запрос по Trace ID из лога/заголовка за секунды (дешёвый путь Tempo).
3. Искать «все вызовы СМЭВ дольше N секунд» через TraceQL (дорогой путь — его надо квотировать).
4. По желанию — RED-метрики сервисов из тех же спанов (metrics-generator или TraceQL metrics), не путая их с алертами инфраструктуры.

Tempo выбран индустрией Grafana как бэкенд «храним в object storage, индекс минимальный». Цена: без Trace ID широкий поиск по месяцам «терабайт» — это не Elasticsearch, и так задумано.

### Как устроена отказоустойчивость (идея, не магия)

Это **три разных контура**. Путать их — главная ошибка.

**1) Приём (distributor + Kafka)**

| Что падает | Что происходит по документации |
|---|---|
| Один distributor | Collector/SDK идут на Service/LB, живые реплики принимают. Непринятое — на стороне клиента/Collector (ретраи). |
| Kafka не подтвердила запись | Distributor **не** считает запись успешной. Клиенту — ошибка. Это правильно: «успех» без WAL = ложь. |
| Весь ЦОД с частью брокеров | Как в документе по Kafka: переживает 1 ЦОД только при RF=3, min.ISR=2, rack=ЦОД. Tempo RF1 **не** спасает, если Kafka потеряла данные. |
| Collector смотрит в один под distributor без Service | Падение этого пода = слепой ingest, даже если реплик пять. |

Следствие: HA **приёма** = несколько distributor + Service/gateway + **живой Kafka-кластер** с нормальной долговечностью. Tempo здесь тонкий.

**2) Свежие данные (live-store)**

| Что падает | Что происходит |
|---|---|
| Live-store одной зоны при zone-aware (две+ зоны) | Другая зона продолжает отдавать ту же партицию. Querier'у достаточно **одного** ответа (read quorum 1). |
| Все live-store партиции | Свежее окно (доки: типично **30 минут–1 час**) недоступно из памяти. История в object storage ещё может искаться. Плюс в 3.0.2+ по умолчанию `live_store.fail_on_high_lag=true`: если лаг Kafka пересекается с окном запроса — **ошибка**, а не «тихо неполный результат». |
| PVC live-store потерян, Kafka retention ещё держит | Реплей с committed offset. Если Kafka уже вычистила — дырка в свежих данных. |

**3) История (object storage + block-builder + scheduler/worker)**

| Что падает | Что происходит |
|---|---|
| Один block-builder | Его партиция перестаёт собираться в блоки, пока под не вернётся (привязка к ordinal/партиции). Данные пока в Kafka. Если Kafka retention истечёт раньше — **потеря истории**. |
| Backend scheduler | Приём идёт. Компакция, retention, redaction **встают**. Диск object storage растёт, поиск деградирует. Автовыбора второго scheduler **нет**. |
| Object storage недоступен | Новые блоки не пишутся; старые не читаются. Это потеря **всей** исторической трассировки, не «одного пода Tempo». |
| Object storage живёт в одном ЦОДе | Падение этого ЦОДа = RPO по всей истории, сколько ни ставьте live-store в трёх залах. |

Официально: в микросервисах после ack Kafka Tempo работает с **RF1** (блок пишется один раз, без 2–2.5× дубля 2.x). Экономия места — ценой того, что **второй копией истории Tempo не занимается**. Копии должны дать Kafka (на коротком окне) и object storage (на длинном).

### Как устроено масштабирование

Единица параллелизма write path — **Kafka-партиция Tempo**, не «ещё один Deployment».

- Distributor шардирует по **hash(Trace ID)** → все спаны одного трейса на одну партицию. Это нужно, чтобы live-store и block-builder собирали трейс целиком.
- Горизонтальный scale live-store / block-builder **упирается** в число партиций. Документация: чтобы добавить инстансы, нужно **сначала** добавить партиции Kafka, потом реплики StatefulSet. С `partitions_per_instance: 1` число block-builder = число партиций.
- Querier масштабируется отдельно под поисковую нагрузку (она может быть огромной при широком TraceQL и нулевой при «только Trace ID»).
- Query-frontend: ориентир документации сайзинга — **2 реплики** (HA), не «по числу ЦОДов».
- Autoscaling distributor относительно безопасен. Autoscaling block-builder «как HTTP Deployment» **ломает** привязку к партициям.

Ориентиры Grafana «с чего начать» (страница Size the cluster; **не ваш замер**, вендор предупреждает, что цифры живут от релиза к релизу):

| Компонент | Старт | CPU / RAM (ориентир вендора) |
|---|---|---|
| Distributor | 1 реплика на **~10 MB/s** входа | 2 ядра / 2 ГБ |
| Live-store | 1 реплика на **~6–10 MB/s**; в Helm — **~10 MB/s** на активную реплику | 1 ядро / **4–20 ГБ** (зависит от состава трейса) |
| Block-builder | 1 реплика на партицию | 0.5 ядра / **5–10 ГБ** |
| Querier | 1 реплика на **~1–2 MB/s** входа (для «типичного» поиска) | RAM 4–20 ГБ |
| Query-frontend | 2 реплики | 4–20 ГБ |
| Backend scheduler | **1** | 0.5 ядра / 1–2 ГБ |
| Backend worker | по нагрузке компакции (у вендора «to be determined») | 0.5 ядра / 1–2 ГБ |

Helm quick-test на одну партицию: нода **от 6 CPU / 16 ГБ RAM** — это лабораторный минимум, не прод.

«Терабайты» в озере **не** равны терабайтам в Tempo. Наполняют: 100% сэмплирование, огромные атрибуты (тела HTTP, XML ведомств), длинный retention, metrics-generator с высокой кардинальностью `span_name`.

Грубая арифметика хранения (порядок величины, не смета):

`объём ≈ пиковый_MB/s × доля_сэмпла × 86400 × дни_retention`

Пик неизвестен — **цифры PVC/бакета нет**. Рычаги, когда цифры появятся: сэмпл в Collector, `max_bytes_per_trace` / `rate_limit_bytes` в overrides, `block_retention`, лимиты metrics-generator (`max_cardinality_per_label`).

Дефолтные лимиты ingest из примеров Helm-доки (это **пример**, не закон): 5 MB/s на тенанта, burst 5 MB/s, 1000 live traces, 10 MB на трейс. На «высокой нагрузке без замеров» эти лимиты либо задушат прод, либо их снимут и получат аварии. Нужен замер.

### Безопасность самого Tempo

Компрометация Tempo / бакета / Kafka-топика ingest = компрометация **карты вызовов** всей системы: кто с кем говорит, идентификаторы заявок, тайминги интеграций с госслужбами, иногда PII в атрибутах. Это не «логи дев-стенда».

Факты из документации, которые нельзя игнорировать:

1. **Встроенной аутентификации нет.** Анонимный доступ к 3200/4317 = читать и писать трейсы.
2. **`X-Scope-OrgID` — не пароль.** Если `multitenancy_enabled: true`, заголовок обязателен, но подставить чужой OrgID может кто угодно, пока прокси не выставляет его **сам** после логина.
3. **`multitenancy_enabled` по умолчанию `false`.** Один общий тенант: любой читатель видит всё.
4. **`log_received_spans`** в доке прямо помечен как не для прода: пишет спаны в лог процесса.
5. Ключи object storage и SASL Kafka в ConfigMap — обычная дыра Helm-установок. Их место — Secret / Vault (у вас уже есть документ по Vault).
6. Трейс с телом SOAP/REST ведомства — это уже контур ПДн/служебных данных, даже если «это же observability».

«Включили `tempo-distributed` и отдали query-frontend в интернет» — это новый внешний архив трассировки, не «просто мониторинг».

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Берём self-hosted open-source Tempo 3.0.3**, не Grafana Cloud Traces и не Grafana Enterprise Traces. Облако вендора в схеме с тремя своими ЦОДами не предполагалось. GET из community-чарта 3.x **убрали**.
2. **Прод — микросервисный режим** (`tempo-distributed` + внешние Kafka и object storage). Тест — **монолит** (`-target=all` / чарт monolithic), без Kafka.
3. **Три ЦОДа = три зоны отказа** Kubernetes (`topology.kubernetes.io/zone`). Tempo не делает «магический stretch». Пока RTT неизвестен, честный прод — один логический Tempo-кластер **только если** Kafka, object storage и memberlist реально живут на этой сети (см. документ по Kafka: порога RTT у вендоров нет).
4. **Цель отказа: пережить 1 ЦОД, не 2 из 3.** Scheduler один. Object storage должен сам переживать 1 ЦОД. Live-store — минимум **две** зоны (официальный пример RF2, quorum 1). Третья зона live-store — это **×3 чтение Kafka** той же партиции: запас по свежим запросам ценой трафика и RAM.
5. **Kafka для ingest Tempo — отдельный кластер (или как минимум отдельный топик с жёсткими quota/ACL), не общий с event-sourcing.** Смешение на одном кластере без изоляции: пик трейсов забивает диск брокеров и латентность `acks=all` у Camunda. Автосоздание топика с **1000 партиций** (`auto_create_topic_default_partitions`) на боевом кластере — отдельная авария. Топик создаём **вручную**.
6. **Object storage — production-grade S3 API с репликацией между ЦОДами** (облачный S3/GCS/Azure **или** свой Ceph RGW / коммерческий MinIO / аналог, который ИБ/эксплуатация готовы сопровождать). Локальный `backend: local` и docker-MinIO из гайда Tempo помечены вендором как **test/eval, not production**. Community MinIO как «бинарник с docker.io» вендор отдельно оговаривает: upstream archived, бинарники CE больше не публикуют.
7. **Нагрузка неизвестна** — поэтому **нет** числа партиций, реплик и терабайт бакета. Есть сигналы (producer errors distributor, consumer lag, `fail_on_high_lag`, heap live-store, длина blocklist, latency TraceQL) и рычаги (партиции, сэмпл, retention, querier).
8. **Формального SLA на «каждый запрос протрассирован» нет.** 24/7 системы ≠ 100% спанов. Сэмплирование и дропы Collector — штатный риск, его надо выбрать сознательно.
9. **Сэмплирование будет** (хотя бы tail-based на ошибки/медленные + head-based на остальное). Иначе «терабайты» наблюдаемости съедят платформу раньше бизнеса.
10. **Инструментация — OpenTelemetry SDK + Collector contrib** (батч, `k8sattributes`, `tail_sampling`). Приложения не пишут в Tempo напрямую из каждого пода в обход Collector: иначе нет единой точки PII/лимитов/TLS.
11. **Grafana как UI уже будет или появится.** Этот документ её не разворачивает, но без datasource Tempo эксплуатация слепая.
12. **ПДн и тела интеграций в спаны не кладём.** Redaction API Tempo 3.0 — аварийный инструмент, не политика «можно писать всё, потом замажем».
13. **TLS на каналах Tempo — стандартный (не ГОСТ).** Если появится требование СКЗИ — vanilla Tempo его не закрывает (критичный пробел).
14. **Один тенант Tempo на контур** (prod / test раздельно), `multitenancy_enabled: true` даже для одного OrgID — чтобы случайный клиент без заголовка не прошёл, а gateway заголовок выставлял сам. Несколько команд на одном Tempo без тенантов = общая куча трейсов.
15. **Тестовый стенд изолирован.** На нём допустимы insecure OTLP и local storage. Прод-секреты и 100% сэмпл на тест не копируем, но **формат экспорта OTLP и запрет PII в атрибутах** лучше сразу как в проде, иначе тест врёт.

---

## Краткое описание ключевых этапов и элементов развертывания

### Для 1 инстанса (тестовый стенд, 1 ЦОД, без нагрузки)

Цель: понять OTLP → поиск в Grafana по Trace ID и простому TraceQL. Не моделировать падение ЦОДа.

**Режим:** monolithic (`-target=all`). Kafka **не ставить** «для правды теста» — в монолите его нет в write path, это будет другой продукт.

**Хранение:** `storage.trace.backend: local` на PVC **или** одноузловой S3-совместимый store из раздела Tempo «local stores for testing». Вендор прямо пишет: эти tools **не для прода**. Для теста — нормально.

**Состав:**

1. Namespace `tempo-test`.
2. Один StatefulSet/Deployment Tempo 3.0.3, PVC на `/var/tempo`.
3. Service: 4317, 4318, 3200.
4. OTLP явно `0.0.0.0:4317` / `4318`.
5. Один Collector contrib в том же namespace, exporter на Tempo, для теста `tls.insecure: true` допустим **только здесь**.
6. Grafana + datasource Tempo на `http://tempo:3200`.
7. Одно-два приложения с OTel SDK (или тестовый `telemetrygen` / Vulture).

**Чего не делать на стенде, если хотите, чтобы он учил прод:**

- не слать тела HTTP/XML в атрибуты;
- не оставлять `log_received_spans`;
- не учить сервисы ходить в Tempo в обход Collector;
- не считать, что «монолит на local выдержал — прод на 3 ЦОДа выдержит».

**Отказоустойчивость:** не требуется. Рестарт пода = реплей с локального WAL; потеря PVC = потеря того, что не успели (в монолите flush делает live-store). Это ок для теста.

**Масштаб:** не требуется. Если упрётесь в CPU — вы уже не на «без нагрузки».

**Безопасность стенда:** сеть изолирована. PLAINTEXT внутри namespace допустим. **Не** публиковать 3200/4317 через Ingress в общую сеть даже «на посмотреть».

**Сильное / слабое этой схемы**

| Сильное | Слабое |
|---|---|
| Совпадает с официальным «getting started» | Ничего не говорит про Kafka-WAL, zone-aware, scheduler=1 |
| Мало движущихся частей | Другой failure mode, чем у прода |
| Дёшево проверить инструментацию | Легко привыкнуть слать 100% спанов и PII |

Минимум проверки стенда: запрос прошёл → в Grafana виден тот же Trace ID, что в логе/заголовке `traceparent`.

---

### Для прода (3 ЦОДа, нагрузка)

Цель: приём трейсов переживает падение **одного** ЦОДа; история живёт в object storage, которое тоже переживает один ЦОД; поиск идёт через аутентифицированный вход; объём контролируется сэмплом и retention.

#### Критичные условия, которых в исходной задаче не было (без них схема — слайд)

| Пробел | Почему критично | Что сделать до «готово» |
|---|---|---|
| RTT/потери между ЦОДами | Kafka ingest, memberlist, S3 API, gRPC querier↔live-store | Замерить. Если сеть «как WAN с джиттером» — не растягивать один Tempo+Kafka, а пересматривать (актив/пассив, отдельный контур) |
| Профиль нагрузки | Сайзинг в **MB/s**, не в «высокая» | Снять пик спанов/с, средний размер спана, долю ошибок; пока нет — не закупать «с запасом на глаз» как финальную ёмкость |
| Сэмплирование | Иначе object storage и Kafka Tempo станут вторым озером | Политика: ошибки и медленные — почти 100%; успешные быстрые — N% |
| Object storage на 3 ЦОДа | Tempo историю **не** реплицирует | Выбрать и **прогнать** отказ ЦОДа на бакете, не на подах Tempo |
| Kafka ingest vs бизнес-Kafka | Конкуренция за диск/CPU брокеров | Отдельный кластер **или** жёсткие quota+ACL+свой топик; топик **руками**, RF=3, min.ISR=2 |
| Политика ПДн в спанах | Интеграции с госслужбами | Allowlist атрибутов; запрет raw payload; процесс redaction на инцидент |
| SLA поиска | TraceQL по широкому окну бьёт querier | Квоты overrides, таймауты query-frontend, кэш Memcached |
| Кто ходит в Tempo | Нет auth | Gateway + IdP; Grafana только через него |

#### Элементы прода (микросервисы)

Официальный инсталлятор: **Helm `tempo-distributed`** (values) **или** Tempo Operator. Tanka — тот же манифест, другой синтаксис. Не собирать 15 Deployment'ов руками «по блогу», если нет команды, которая это сопровождает.

Обязательный внешний контур:

1. **Kafka ingest** (правила из `Apache Kafka.md`): 3 или 5 контроллеров, брокеры пачками по ЦОДам, TLS+SASL+ACL, топик например `tempo-traces`. Партиции = план параллелизма (старт часто 3–12, не 1000). Retention топика **длиннее**, чем цикл block-builder + запас на рестарт/лаг (иначе live-store/builder не реплейнут). Мониторинг **consumer lag**.
2. **Object storage** с шифрованием at-rest (у S3 в Tempo есть SSE-KMS / SSE-S3 / SSE-C — это настройки бэкенда, не «Tempo сам шифрует файлы»). Доступ по IAM/IRSA или ключам из Vault. Бакет **только** для Tempo. Версионирование/immutable — решение ИБ, Tempo его не заменяет.
3. **Memcached** 3 реплики (как в jsonnet-примерах Grafana: 3), размазанные по зонам. Без кэша Trace ID lookup по длинному retention дорожает.
4. **OTel Collector** как DaemonSet и/или шлюз: единственная точка, куда приложения шлют OTLP. От Collector до distributor — **TLS** (в доке Collector для прода `tls.insecure: true` запрещён смыслом гайда).
5. **Gateway / HAProxy** перед query-frontend **и** перед OTLP, если Collector не единственный клиент. Basic/OAuth/mTLS — на прокси. После успеха прокси ставит `X-Scope-OrgID`.
6. **Grafana** datasource Tempo на URL gateway, тот же заголовок тенанта.

Компоненты Tempo (раскладка по зонам):

| Компонент | Реплики на старт (пока нет MB/s) | Зоны | Зачем так |
|---|---|---|---|
| Distributor | ≥ 3 (по одной в ЦОД) + HPA потом | 3 | Stateless вход |
| Live-store | = партиции × **2** (две зоны) или × **3** (все ЦОДы) | 2 или 3 | Zone-aware; quorum чтения 1 |
| Block-builder | = число партиций (при 1 партиции на инстанс) | размазать, но это не RF: каждый ординал уникален | Потеря пода = отставание **его** партиций |
| Query-frontend | 2–3 | разные ЦОДы | HA API |
| Querier | ≥ 3, дальше по latency поиска | 3 | Чтение истории |
| Backend scheduler | **1** + чёткий Runbook «поднять заново» | любой, но не «по одному в ЦОД» | Два scheduler — конфликт работ |
| Backend worker | ≥ 2 | разные ЦОДы | Компакция переживает 1 ЦОД, scheduler — нет |
| Metrics-generator | 0 пока нет квоты кардинальности; иначе отдельно | — | Легко взорвать Prometheus |

Live-store: PVC, anti-affinity, `topologySpreadConstraints` по зоне. PDB, чтобы drain не снял все владельцы одной партиции сразу.

`fail_on_high_lag: true` (дефолт 3.0) и `query_frontend.query_end_cutoff: 30s` оставляем: лучше ошибка «свежие данные отстают», чем тихий неполный TraceQL. Срезать cutoff «чтобы графики красивые» — значит врать себе.

Overrides (лимиты) **включить до** открытия контура:

- `ingestion.rate_limit_bytes` / `burst_size_bytes` / `max_traces_per_user`
- `global.max_bytes_per_trace`
- для metrics-generator — `max_cardinality_per_label`, фильтры service graph; санитайзер `span_name` — experimental, не единственная защита

Retention: **не оставлять 14 дней по привычке и не ставить «7 лет как в озере»**. Трейсы — операционный контур. Срок согласует ИБ; чем длиннее, тем больнее blocklist и lookup без окна времени.

#### Безопасность прода (без этого кластер не считается настроенным)

Три слоя, все три:

1. **Сеть.** NetworkPolicy: приложения → только Collector; Collector → только distributor:4317/4318; Grafana/люди → только gateway; querier → live-store 9095 и object storage; memberlist 7946 только внутри Tempo. Object storage и Kafka не торчат в подсеть разработки.
2. **Шифрование канала.** TLS на OTLP (Collector↔Tempo), на server HTTP/gRPC внутри по возможности mTLS, SASL_SSL до Kafka ingest (как в документе Kafka: SCRAM без TLS недопустим), HTTPS/TLS до S3.
3. **Кто ты и что можно.** Gateway аутентифицирует. Tempo видит только выставленный OrgID. ACL на Kafka-топик: писать — distributor; читать — live-store, block-builder, metrics-generator. Бакет: писать — block-builder/worker; читать — querier/worker. Ключи в Vault, не в Git.

Дополнительно:

- `multitenancy_enabled: true`;
- отдельные инсталляции test/prod (общий бакет «для экономии» смешивает retention и права);
- не включать приёмники, которыми не пользуетесь (Zipkin «на всякий случай» — лишняя поверхность);
- аудит доступа к gateway и к бакету;
- шифрование томов WAL;
- запрет `allow` на TraceQL без лимитов с рабочих мест разработчиков напрямую в query-frontend.

##### Сильные / слабые стороны выбранной ИБ-схемы (gateway + TLS + тенант + Vault)

| Сильное | Слабое |
|---|---|
| Совпадает с официальной моделью «у Tempo нет auth» | Ошибка прокси (не тот OrgID) = чужие трейсы или отказ |
| ACL Kafka и IAM бакета закрывают обход API | Спаны уже уехали в бакет — revoke токена Grafana их не вычищает |
| Collector — единая точка вырезания PII | Если SDK шлёт мимо Collector, политика исчезает |
| SSE на S3 закрывает диск стораджа | SSE-C ключ у Tempo = ещё один секрет, который нельзя терять |
| Не зависит от GET | Нет встроенного RBAC «эта роль видит только сервис X» — это Grafana/прокси |

#### Kubernetes-специфика прода

- Live-store и block-builder — **StatefulSet**, стабильные ordinal'ы (привязка к партициям).
- `WaitForFirstConsumer` + StorageClass зоны, чтобы PVC не оказался в ЦОДе A, а под — в B.
- Ресурсы live-store: лимит памяти **с запасом** под состав трейса (таблица вендора 4–20 ГБ — это диапазон, не «поставим 4 и забудем»). OOM live-store = дырка в свежем окне и реплей из Kafka.
- PDB на distributor/querier/frontend. На block-builder Grafana в jsonnet допускает высокий `maxUnavailable`, потому что инстансы независимы по партициям — но **не** снимайте все сразу при маленьком числе партиций.
- Rollout: для block-builder в jsonnet есть опция concurrent rollout через rollout-operator. Без понимания ordinal'ов «kubectl rollout» в пятницу — способ накопить лаг.
- Не класть WAL на NFS.
- Один scheduler: PDB `minAvailable: 1` его не спасёт от мёртвого ЦОДа — нужен мониторинг и приоритет восстановления.

#### Порядок вывода в прод (этапы, не команда за командой)

1. Замерить сеть между ЦОДами. Решение: один Tempo-кластер vs нельзя.
2. Поднять **HA object storage** и проверить отказ одного ЦОДа **чтением/записью бакета**, без Tempo.
3. Поднять **Kafka ingest** (отдельно от бизнес-шины). Создать топик вручную: RF=3, min.ISR=2, разумное число партиций, retention с запасом. ACL+TLS. **Выключить** надежду на `auto_create_topic` в проде.
4. Выбрать число зон live-store (2 vs 3) сознательно: стоимость ×N чтения Kafka.
5. Поставить `tempo-distributed`, образ **3.0.3**, все компоненты из таблицы, Memcached, overrides-лимиты, retention.
6. Включить TLS, gateway, `multitenancy_enabled`, секреты из Vault **до** боевого трафика.
7. Поставить Collector: tail sampling, allowlist атрибутов, k8s labels, TLS в Tempo.
8. Подключить Grafana. Прогнать Vulture или синтетику: write → read Trace ID → TraceQL. Убить под distributor, под live-store одной зоны, под block-builder. Смотреть лаг Kafka и `fail_on_high_lag`.
9. Только потом — 1–2 реальных сервиса (интеграционный API, один воркер Camunda), не «все 30 ведомств сразу».
10. Снять фактический MB/s и размер спана. Пересчитать партиции, live-store RAM, querier, бакет. Добавлять партиции **пакетом** с репликами builder/live-store.
11. Прогон «ЦОД недоступен» на препроде: выключить зону. Проверить **приём**, поиск свежего, поиск истории, компакцию (scheduler жив?). Без этого учения отказоустойчивости нет.

Без пунктов 2, 3 и 11 у вас есть красивые поды Tempo и надежда.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | Не требуется | Микросервисы; Kafka ingest RF=3/min.ISR=2; object storage переживает 1 ЦОД; live-store ≥2 зоны; scheduler=1 с runbook; PVC на WAL |
| Производительность / масштаб | Не требуется | Партиции как единица масштаба; сайзинг от MB/s; сэмпл; retention не «как озеро»; кэш Memcached; querier отдельно от ingest |
| Безопасность | PLAINTEXT только в изоляции | Нет публичного 3200/OTLP; gateway+TLS; тенант; ACL Kafka; IAM/ключи бакета в Vault; PII не в спанах |

**Не готов к проду**, если: monolithic на нагрузке, `backend: local`, docker-MinIO как «хранилище на годы», ingest на боевом Kafka-топике с auto-create 1000 партиций, RF1 Kafka, один live-store, scheduler «потом», query-frontend в интернете без auth, 100% спанов с телами ответов ведомств, RTT между ЦОДами неизвестен, а схема уже stretch, образ чарта 3.0.2 без патча 3.0.3.

---

## Источники (чтобы не принимать на веру)

- Релиз v3.0.3 (13 Aug 2026), security-патчи: https://github.com/grafana/tempo/releases
- Release notes 3.0 (архитектура, RF1, TraceQL metrics GA, scheduler/worker, redaction): https://grafana.com/docs/tempo/latest/release-notes/v3-0/
- Планирование, Kafka обязателен в микросервисах, object storage: https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/plan/
- Режимы deployment: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/deployment-modes/
- Компоненты / Kafka как WAL / RF1: https://grafana.com/docs/tempo/latest/introduction/architecture/
- Live-store, zone-aware, quorum 1: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/components/live-store/ и https://grafana.com/docs/tempo/latest/operations/manage-advanced-systems/zone-aware-live-stores/
- Kafka-партиции, replay, retention топика: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/components/kafka/
- Distributor пишет в Kafka только после ack: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/components/distributor/
- «Only one scheduler»: https://grafana.com/docs/tempo/latest/configuration/ (блок backend scheduler)
- Сайзинг: https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/plan/size/
- Auth отсутствует, нужен reverse proxy: https://grafana.com/docs/tempo/latest/operations/authentication/
- Мультитенантность, `X-Scope-OrgID`: https://grafana.com/docs/tempo/latest/configuration/multitenancy/
- Манифест дефолтов (порты 3200/9095, memberlist 7946, `block_retention: 336h`, ingest kafka auto-create 1000 партиций): https://grafana.com/docs/tempo/latest/configuration/manifest/
- OTLP 4317/4318, localhost с 2.7, TLS Collector: https://grafana.com/docs/tempo/latest/set-up-for-tracing/instrument-send/set-up-collector/otel-collector/
- Object storage S3/GCS/Azure, local для dev: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/object-storage/
- MinIO/SeaweedFS/rclone — test, not production: https://grafana.com/docs/tempo/latest/configuration/hosted-storage/s3/
- Миграция 2.x→3.0, нет downgrade, vParquet3 не поддерживается, SSB удалён: https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/migrate-to-3/
- Helm get-started (внешние Kafka и S3, ~10 MB/s на live-store, community charts): https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/ (и репозиторий grafana-community/helm-charts)
- Кэш Memcached/Redis experimental: https://grafana.com/docs/tempo/latest/operations/caching/
- Artifact Hub `tempo-distributed` 3.0.6 → appVersion **3.0.2**, K8s **^1.25**: https://artifacthub.io/packages/helm/grafana-community/tempo-distributed

Утверждения про «типичный порог RTT для трёх ЦОДов» и про «хватит N реплик под ваши терабайты» в документации Grafana Tempo **отсутствуют** — поэтому в этом файле они не выдаются за стандарт, а вынесены в допущения и в таблицу пробелов.
