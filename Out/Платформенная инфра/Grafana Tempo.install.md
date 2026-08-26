# Grafana Tempo 3.0.3 — установка и конфигурирование

Связанный документ (глоссарий, RF1, Kafka-WAL, почему так): `Grafana Tempo.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Tempo-кластера (memberlist, live-store, Kafka ingest) на несколько ЦОДов **не делаем**: микросервисы **одного** кластера живут в одном ЦОДе.

Версии: **Grafana Tempo 3.0.3**, образ `grafana/tempo:3.0.3`. На Kubernetes — Helm **`tempo-distributed`** (репозиторий `grafana-community/helm-charts`). Чарт **3.0.6** по умолчанию тянет Tempo **3.0.2** — образ **переопределить на 3.0.3**.  
Документация: https://grafana.com/docs/tempo/latest/  
Релиз 3.0: https://grafana.com/docs/tempo/latest/release-notes/v3-0/

Линия 3.0 — major: ingester и compactor **удалены**. Прод на 2.x не ставим: in-place downgrade нет, сразу закладываете миграцию.

---

## Допущения этой инструкции

1. **Stretch запрещён.** Distributor, live-store, block-builder, querier, scheduler **одного** кластера — **внутри одного ЦОДа**. Другие ЦОДы: отдельный Tempo **или** приложения шлют трейсы в Collector ЦОД-1.
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`). Прод-режим — **микросервисы** + внешние Kafka ingest и object storage. Тест — **монолит** (`-target=all`), без Kafka.
3. Dev — изолированная сеть; insecure OTLP допустим только там.
4. Нагрузки нет — нет числа партиций «хватит для прода». Есть минимум ролей и рычаги (партиции, сэмпл, retention).
5. Object storage для прода **есть или будет**. `backend: local` и docker-MinIO вендор помечает как test/eval, not production.
6. Kafka ingest Tempo — **отдельный** кластер или жёстко изолированный топик (см. `Apache Kafka.install.md`), не бизнес-шина. Топик создаём **вручную**, не auto-create на 1000 партиций.
7. Для 2 ЦОДов: микросервисы Tempo в ЦОД-1; ЦОД-2 шлёт OTLP в Collector ЦОД-1 **или** свой Tempo. Для 3 ЦОДов: то же. Третий ЦОД **не** добавляет live-store «для кворума» в чужой кластер.
8. Не смешивать 2.x и 3.0. Backend scheduler — **ровно один** процесс на кластер.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** OTLP → поиск по Trace ID в Grafana. **Не** цель: отказ ЦОДа и Kafka-WAL.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порты 3200, 4317, 4318 свободны на localhost.
- Grafana на стенде уже есть или появится (UI Tempo нет).

### Установка (Docker — основной путь Dev)

Монолит: один процесс `-target=all`. Kafka **не** ставить «для правды теста» — в монолите его нет в write path.

С Tempo 2.7 OTLP по умолчанию слушает **localhost** внутри процесса: без явного `0.0.0.0` Collector с соседнего контейнера получит пустой Tempo. На хосте порты всё равно публикуем только на `127.0.0.1`.

Минимальный `tempo.yaml` (local backend — вендор: не для прода):

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

Файл `tempo.yaml` — на стенде рядом с командой, не в репозиторий документации. Привязка к `127.0.0.1` обязательна. Образ **3.0.3**, не тег чарта 3.0.2 и не `latest`.

Проверка: HTTP API на `http://127.0.0.1:3200/ready` (путь сверять с манифестом 3.0.3). Затем Collector/SDK → OTLP → Grafana datasource `http://127.0.0.1:3200`.

На Kubernetes Dev: один StatefulSet/Deployment Tempo 3.0.3, PVC на `/var/tempo`, Service 4317/4318/3200. **Не** этот манифест в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Режим | monolithic | Иначе команда утонет в Kafka раньше OTLP |
| Хранение | `local` на volume | Вендор: не для прода |
| Kafka | нет | В монолите не в write path |
| TLS OTLP | insecure допустим | Изолированный стенд |
| Сэмпл | можно 100% | Нет нагрузки; в прод не копировать |
| `log_received_spans` | выкл | Не для прода и не полезная привычка |
| 3200/4317 с мира | запрещён | Даже «на посмотреть» |

Чего **не** упрощать: тег **3.0.3**; OTLP через Collector, не из каждого пода в обход; тела HTTP/XML ведомств в атрибуты не класть; формат экспорта как в проде.

### Проверка Dev

1. Образ/процесс **3.0.3**, не 2.10 и не 3.0.2 «потому что чарт».
2. Запрос прошёл → в Grafana тот же Trace ID, что в `traceparent`.
3. Рестарт без PVC: свежие трейсы пропали — ожидаемо для local/монолита.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный getting started | Ничего не говорит про Kafka-WAL, scheduler=1, object storage |
| Мало движущихся частей | Другой failure mode, чем у прода |
| Дешёво проверить SDK | Легко привыкнуть слать 100% спанов и PII |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** приём трейсов переживает отказ **машины/пода внутри ЦОДа**; отказ **целого ЦОДа** с Tempo = нет этого бэкенда, пока не шлёте в другой Tempo / не поднимете DR. История — в object storage, которое само должно переживать 1 ЦОД (это не Tempo).

### Почему не stretch

Memberlist **7946**, gRPC querier↔live-store, ingest в Kafka и S3 API чувствительны к RTT. Порога в доке Tempo **нет**. Zone-aware live-store в документации — про зоны **одного** кластера; разносить микросервисы по ЦОДам без замера = ложные выборы и дырки в свежем окне. Один backend-scheduler на кластер: второй в чужом ЦОДе — конфликт работ, не HA.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один микросервисный Tempo:

- Helm `tempo-distributed`, образ **явно** `grafana/tempo:3.0.3`;
- внешний **Kafka ingest** в **этом** ЦОДе (RF=3, min.ISR=2, топик руками);
- **object storage** (S3 API); бакет переживает отказ ЦОДа своими средствами, не репликацией Tempo (RF1);
- live-store и block-builder — StatefulSet, PVC, не NFS;
- query-frontend **2** реплики; backend scheduler **1**;
- Memcached в том же ЦОДе;
- OTel Collector — единственная точка OTLP от приложений; TLS до distributor;
- gateway/HAProxy перед 3200 и при необходимости перед OTLP: у Tempo **нет** встроенной auth.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: весь микросервисный Tempo + Kafka ingest. ЦОД-2: Collector шлёт в ЦОД-1 **или** отдельный Tempo (свой Kafka/S3) | Нет приёма/поиска **этого** кластера. Второй Tempo не подхватит те же блоки сам |
| **3 ЦОДа** | То же; ЦОД-3 аналогично ЦОД-2 | То же; третий ЦОД не делает «RF live-store между городами» |

Не открывать 7946/9095/4317 между ЦОДами «чтобы один кластер». Не ставить второй scheduler «на всякий случай в ЦОД-2».

### Предпосылки прода

- Kubernetes ЦОД-1; CSI RWO под WAL.
- Kafka ingest и object storage спроектированы **до** Helm.
- NetworkPolicy: приложения → Collector; Collector → distributor 4317/4318; Grafana → gateway; memberlist только внутри Tempo ЦОД-1.
- Секреты S3 и SASL Kafka — Vault, не ConfigMap.

### Установка (Kubernetes, ЦОД-1)

1. Kafka-топик ingest вручную (партиции = план параллелизма, старт часто 3–12, не 1000). ACL: писать distributor, читать live-store/block-builder.
2. Бакет S3 только для Tempo; IAM/ключи из Vault.
3. Helm `tempo-distributed` из grafana-community; в values:

```yaml
image:
  repository: grafana/tempo
  tag: "3.0.3"
# не оставлять appVersion чарта 3.0.2
```

4. Включить overrides-лимиты (`rate_limit_bytes`, `max_bytes_per_trace`) **до** боевого трафика.
5. `multitenancy_enabled: true`; gateway сам ставит `X-Scope-OrgID`. `X-Scope-OrgID` — не пароль.
6. Retention не оставлять «14 дней по привычке» и не ставить «как озеро». Срок — ИБ.
7. Grafana datasource на URL **gateway**, не сырой query-frontend в интернет.

Tempo Operator (`TempoStack`) — допустимый путь с теми же правилами: все компоненты **в одном** ЦОДе, образ 3.0.3.

### Конфигурирование (смысл, не полный values)

| Параметр | Прод | Зачем |
|---|---|---|
| Режим | microservices | Монолит Grafana для прода не рекомендует |
| `backend` | s3 (или совместимый прод-store) | local — не прод |
| Kafka auto-create | выкл | 1000 партиций на боевом кластере — авария |
| Scheduler | 1 | Цитата доки: only one at a time |
| `fail_on_high_lag` | true (дефолт 3.0) | Лучше ошибка, чем тихий неполный TraceQL |
| OTLP приёмники | только нужные; Zipkin «на всякий случай» не включать | Лишняя поверхность |
| `log_received_spans` | выкл | Пишет спаны в лог процесса |

Collector: tail sampling (ошибки/медленные ≈ 100%, остальное N%), allowlist атрибутов, `tls.insecure: true` в проде запрещён смыслом гайда.

### Масштабирование (когда появятся цифры)

1. Единица write path — **Kafka-партиция**, не «ещё один Deployment». Сначала партиции, потом реплики live-store/block-builder.
2. Querier — отдельно под TraceQL. Autoscaling block-builder «как HTTP» ломает привязку к партициям.
3. Формула объёма порядка: пиковый MB/s × доля сэмпла × 86400 × дни retention. Пика нет — **цифры бакета нет**.

### Проверка прода (пока это не пройдено — это не прод)

1. Образ **3.0.3** на всех компонентах ЦОД-1 (не 3.0.2 из чарта).
2. Write → read Trace ID → простой TraceQL. Vulture/синтетика.
3. Убить distributor: Service жив. Убить live-store: свежее окно/лаг Kafka под контролем. Убить scheduler: приём идёт, компакция встаёт — runbook «поднять scheduler».
4. Gateway без токена — отказ. Прямой 3200 с мира — закрыт.
5. Учение «ЦОД-1 выключен»: Collector ЦОД-2 либо буферизует/шлёт в DR Tempo, либо честная потеря приёма. Object storage читается с другой площадки — отдельная проверка бакета, не подов Tempo.

### Сильные / слабые стороны прод-схемы (кластер в одном ЦОДе)

| Сильное | Слабое |
|---|---|
| Memberlist и Kafka ingest локальные | Падение ЦОД-1 = нет этого Tempo |
| Согласовано с запретом stretch и с RF1 | История живёт в object storage: его HA — ваш, не Tempo |
| Один scheduler, понятный runbook | Второй ЦОД не «догоняет» чужие блоки сам |
| | OTLP через город, если приложения в ЦОД-2 шлют в ЦОД-1 — сеть ваша |

**Не готов к проду**, если: monolithic на нагрузке; `backend: local`; образ чарта 3.0.2 без override 3.0.3; линия 2.x; ingest на боевом Kafka с auto-create 1000 партиций; query-frontend в интернет без auth; 100% спанов с телами ведомств; микросервисы размазаны по ЦОДам; два scheduler; Dev-манифест скопирован в бой.

---

## Источники

- Релиз 3.0.3: https://github.com/grafana/tempo/releases
- Архитектура 3.0, RF1, scheduler: https://grafana.com/docs/tempo/latest/release-notes/v3-0/
- Режимы deployment: https://grafana.com/docs/tempo/latest/reference-tempo-architecture/deployment-modes/
- Auth отсутствует: https://grafana.com/docs/tempo/latest/operations/authentication/
- Helm `tempo-distributed` 3.0.6 → appVersion 3.0.2: https://artifacthub.io/packages/helm/grafana-community/tempo-distributed
- Правила: `Grafana Tempo.md`
