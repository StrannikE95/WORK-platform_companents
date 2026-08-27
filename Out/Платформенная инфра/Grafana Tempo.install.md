# Grafana Tempo 3.0.3 — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `Grafana Tempo.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

Grafana Tempo — хранилище трейсов (цепочек вызовов). Ставите **свою** копию, не облако вендора.

Версия в этой инструкции: **Grafana Tempo 3.0.3**, образ `grafana/tempo:3.0.3`. На Kubernetes — Helm **`tempo-distributed`** (репозиторий `grafana-community/helm-charts`). Чарт **3.0.6** по умолчанию тянет Tempo **3.0.2** — образ **переопределить на 3.0.3**.  
Документация: https://grafana.com/docs/tempo/latest/

Линия 3.0 — major: ingester и compactor **удалены**. Бой на 2.x не ставим: in-place downgrade нет. GET из community-чарта 3.x тоже **убрали**.

Один кластер микросервисов Tempo (memberlist, live-store, Kafka ingest), размазанный на несколько дата-центров, здесь **не собираем**. Порога задержки в документации Tempo нет. Живой кластер — внутри одной площадки. Вторая площадка — свой Tempo **или** приложения шлют OTLP в Collector первой.

---

## О чём эта инструкция молча договорилась

1. Distributor, live-store, block-builder, querier, scheduler **одного** кластера живут **в одном** дата-центре. Между площадками не открываем 7946/9095/4317 «чтобы один кластер».
2. Бой — Kubernetes в каждом дата-центре отдельно (см. `Kubernetes.install.md`). Режим боя — **микросервисы** + внешние Kafka ingest и object storage. Тест — **монолит** (`-target=all`), без Kafka.
3. Учебный стенд — закрытая сеть. Insecure OTLP допустим **только там**.
4. Цифр вашей нагрузки нет — нет фразы «хватит N партиций». Сначала считают MB/s и сэмпл; партиции — когда одна машина мала.
5. Object storage для боя **есть или будет**. `backend: local` и docker-MinIO вендор помечает как test/eval, not production.
6. Kafka ingest Tempo — **отдельный** кластер или жёстко изолированный топик (см. `Apache Kafka.install.md`), не бизнес-шина. Топик создаём **вручную**, не auto-create на 1000 партиций.
7. Два дата-центра: микросервисы Tempo в первом; второй шлёт OTLP в Collector первого **или** свой Tempo. Три — то же. Третья площадка **не** добавляет live-store «для кворума» в чужой кластер.
8. Backend scheduler — **ровно один** процесс на кластер. Второй «на всякий случай» в чужом дата-центре — конфликт работ, не отказоустойчивость.
9. Не смешивать 2.x и 3.0.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** OTLP → поиск по Trace ID в Grafana. **Не зачем:** отказ дата-центра и Kafka-WAL.

### Что должно быть до установки

- Docker Engine **или** однонодовый Kubernetes.
- На localhost свободны порты 3200, 4317, 4318.
- Grafana на стенде уже есть или появится (своего UI у Tempo нет).
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь для учёбы)

Монолит: один процесс `-target=all`. Kafka **не** ставить «для правды теста» — в монолите его нет в write path.

С Tempo 2.7 OTLP по умолчанию слушает **localhost** внутри процесса: без явного `0.0.0.0` Collector с соседнего контейнера получит пустой Tempo. На хосте порты всё равно публикуем только на `127.0.0.1`.

Минимальный `tempo.yaml` (local backend — вендор: не для боя):

```yaml
server:
  http_listen_port: 3200
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
```

```bash
docker run -d --name tempo-dev \
  -p 127.0.0.1:3200:3200 \
  -p 127.0.0.1:4317:4317 \
  -p 127.0.0.1:4318:4318 \
  -v "$PWD/tempo.yaml:/etc/tempo.yaml:ro" \
  grafana/tempo:3.0.3 -config.file=/etc/tempo.yaml
```

Файл `tempo.yaml` — на стенде рядом с командой. Образ **3.0.3**, не тег чарта 3.0.2 и не `latest`.

Проверка: HTTP API на `http://127.0.0.1:3200/ready` (путь сверять с манифестом 3.0.3). Затем Collector/SDK → OTLP → Grafana datasource `http://127.0.0.1:3200`.

На Kubernetes для учёбы: один StatefulSet/Deployment Tempo 3.0.3, PVC на `/var/tempo`, Service 4317/4318/3200. **Не** этот манифест в бой.

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Режим | monolithic | Иначе Kafka раньше, чем OTLP |
| Хранение | `local` на volume | Вендор: не для боя |
| Kafka | нет | В монолите не в write path |
| TLS OTLP | insecure допустим | Изолированный стенд |
| Сэмпл | можно 100% | Нет нагрузки; в бой не копировать |
| `log_received_spans` | выкл | Пишет спаны в лог процесса |
| 3200/4317 с мира | запрещён | Даже «на посмотреть» |

Чего **не** упрощаем: тег **3.0.3**; OTLP через Collector, не из каждого пода в обход; тела HTTP/XML ведомств в атрибуты не класть; формат экспорта как в бою.

### Как понять, что стенд живой

1. Образ/процесс **3.0.3**, не 2.10 и не 3.0.2 «потому что чарт».
2. Запрос прошёл → в Grafana тот же Trace ID, что в `traceparent`.
3. Рестарт без PVC: свежие трейсы пропали — ожидаемо для local/монолита.

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Минуты, официальный getting started | Ничего не говорит про Kafka-WAL, scheduler=1, object storage |
| Мало движущихся частей | Другой failure mode, чем у боя |
| Дешево проверить SDK | Легко привыкнуть слать 100% спанов и персональные данные |

Перед боем полезен **препрод**: микросервисы + Kafka ingest + бакет + gateway — всё **в одном** дата-центре.

---

## Бой: один живой дата-центр, второй — свой Tempo или путь в первый

**Зачем:** приём трейсов переживает отказ **машины или пода внутри площадки** (несколько distributor, Kafka с RF=3, live-store, PVC на WAL). Отказ **всего дата-центра с Tempo** = нет этого бэкенда, пока не шлёте в другой Tempo / не поднимете DR. История живёт в object storage: его отказоустойчивость — отдельный проект, не Tempo (RF1: блок пишется один раз).

### Почему кластер не растягиваем на несколько дата-центров

Memberlist **7946**, gRPC querier↔live-store, ingest в Kafka и S3 API чувствительны к задержке. Порога в доке Tempo **нет**. Zone-aware live-store в документации — про зоны **одного** кластера. Один backend-scheduler на кластер: второй в чужом дата-центре — конфликт работ.

### Как расставить в активном дата-центре

Один микросервисный Tempo:

- Helm `tempo-distributed`, образ **явно** `grafana/tempo:3.0.3`.
- Внешний **Kafka ingest** в **этом** дата-центре (RF=3, min.ISR=2, топик руками; партиции = план параллелизма, старт часто 3–12, не 1000). Retention топика **длиннее**, чем цикл block-builder + запас на рестарт.
- **Object storage** (S3 API); бакет переживает отказ площадки своими средствами.
- Live-store и block-builder — StatefulSet, PVC, не NFS.
- Query-frontend **2** реплики; backend scheduler **1**; backend worker ≥ 2.
- Memcached в том же дата-центре.
- OTel Collector — единственная точка OTLP от приложений; TLS до distributor (`tls.insecure: true` в бою запрещён смыслом гайда).
- Gateway/HAProxy перед 3200: у Tempo **нет** встроенной auth. После успеха прокси ставит `X-Scope-OrgID`.

**Между площадками:**

| Сколько площадок | Что где | Если умер активный дата-центр |
|---|---|---|
| **Две** | Первая: весь микросервисный Tempo + Kafka ingest. Вторая: Collector шлёт в первую **или** отдельный Tempo (свой Kafka/S3) | Нет приёма/поиска **этого** кластера. Второй Tempo не подхватит те же блоки сам |
| **Три** | Третья как вторая | То же. Третий дата-центр не делает «RF live-store между городами» |

Не открывать 7946/9095/4317 между площадками «чтобы один кластер». Не ставить второй scheduler «на всякий случай».

### Что должно быть до боевой установки

- Kubernetes активной площадки; CSI RWO под WAL.
- Kafka ingest и object storage спроектированы **до** Helm. Бакет **только** для Tempo; ключи из Vault.
- NetworkPolicy: приложения → Collector; Collector → distributor 4317/4318; Grafana → gateway; memberlist только внутри Tempo этой площадки.
- Политика сэмплирования (ошибки/медленные ≈ 100%, остальное N%) и allowlist атрибутов.
- NTP на всех машинах.

### Порядок установки в активном дата-центре

1. Kafka-топик ingest вручную. ACL: писать distributor, читать live-store/block-builder. **Выключить** надежду на `auto_create_topic`.
2. Бакет S3; IAM/ключи из Vault. Прогнать отказ площадки **чтением/записью бакета**, без Tempo.
3. Helm `tempo-distributed` из grafana-community; в values:

```yaml
image:
  repository: grafana/tempo
  tag: "3.0.3"
# не оставлять appVersion чарта 3.0.2
```

4. Включить overrides-лимиты (`rate_limit_bytes`, `max_bytes_per_trace`) **до** боевого трафика.
5. `multitenancy_enabled: true`; gateway сам ставит `X-Scope-OrgID`.
6. Retention не оставлять «14 дней по привычке» и не ставить «как озеро». Срок — решение ИБ.
7. Grafana datasource на URL **gateway**, не сырой query-frontend в интернет.

Tempo Operator (`TempoStack`) — допустимый путь с теми же правилами: все компоненты **в одном** дата-центре, образ 3.0.3.

### Правила конфигурации боя

| Параметр | В бою | Зачем |
|---|---|---|
| Режим | microservices | Монолит Grafana для боя не рекомендует |
| `backend` | s3 (или совместимый прод-store) | local — не бой |
| Kafka auto-create | выкл | 1000 партиций на боевом кластере — авария |
| Scheduler | 1 | Цитата доки: only one at a time |
| `fail_on_high_lag` | true (дефолт 3.0) | Лучше ошибка, чем тихий неполный TraceQL |
| OTLP приёмники | только нужные; Zipkin «на всякий случай» не включать | Лишняя поверхность |
| `log_received_spans` | выкл | Пишет спаны в лог процесса |
| Metrics-generator | 0, пока нет квоты кардинальности | Легко взорвать Prometheus |

Live-store: PVC, anti-affinity, `topologySpreadConstraints`. PDB, чтобы drain не снял всех владельцев одной партиции сразу. WAL не на NFS.

### Как расти, когда появятся цифры нагрузки

1. Единица write path — **Kafka-партиция**, не «ещё один Deployment». Сначала партиции, потом реплики live-store/block-builder.
2. Querier — отдельно под TraceQL. Autoscaling block-builder «как HTTP» ломает привязку к партициям.
3. Формула объёма порядка: пиковый MB/s × доля сэмпла × 86400 × дни retention. Пика нет — **цифры бакета нет**.
4. Ориентиры вендора (distributor 2 ядра / 2 ГБ на ~10 MB/s и т.д.) — стартовые точки, не смета.

### Проверки, без которых это ещё не бой

1. Образ **3.0.3** на всех компонентах активной площадки (не 3.0.2 из чарта).
2. Write → read Trace ID → простой TraceQL. Vulture/синтетика.
3. Убить distributor: Service жив. Убить live-store: свежее окно/лаг Kafka под контролем. Убить scheduler: приём идёт, компакция встаёт — runbook «поднять scheduler».
4. Gateway без токена — отказ. Прямой 3200 с мира — закрыт.
5. Учение «активный дата-центр выключен»: Collector второй площадки либо буферизует/шлёт в DR Tempo, либо честная потеря приёма. Object storage читается с другой площадки — отдельная проверка бакета, не подов Tempo.

### Что хорошо и что плохо в схеме «кластер в одном дата-центре»

| Хорошо | Плохо |
|---|---|
| Memberlist и Kafka ingest локальные | Падение этой площадки = нет этого Tempo |
| Один scheduler, понятный runbook | История живёт в object storage: его HA — ваш, не Tempo |
| Согласовано с RF1 | Второй дата-центр не «догоняет» чужие блоки сам |
| | OTLP через город, если приложения шлют в первую площадку — сеть ваша |

**Не готово к бою**, если: monolithic на нагрузке; `backend: local`; образ чарта 3.0.2 без override 3.0.3; линия 2.x; ingest на боевом Kafka с auto-create 1000 партиций; query-frontend в интернет без auth; 100% спанов с телами ведомств; микросервисы размазаны по дата-центрам; два scheduler; учебный манифест скопирован в бой.

---

## Откуда цифры и имена образов

- Релиз 3.0.3: https://github.com/grafana/tempo/releases
- Архитектура 3.0, RF1, scheduler: https://grafana.com/docs/tempo/latest/release-notes/v3-0/
- Режимы deployment: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/deployment-modes/
- Auth отсутствует: https://grafana.com/docs/tempo/latest/operations/authentication/
- Сайзинг: https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/plan/size/
- Helm `tempo-distributed` 3.0.6 → appVersion 3.0.2: https://artifacthub.io/packages/helm/grafana-community/tempo-distributed
- Правила и схема компонентов: `Grafana Tempo.md`
