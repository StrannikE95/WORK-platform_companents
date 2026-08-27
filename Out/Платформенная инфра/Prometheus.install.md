# Prometheus 3.13.2 LTS — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `Prometheus.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

Prometheus — система метрик: сам забирает числа с `/metrics`, хранит ряды, считает алерты. Ставите **свою** копию, не облако вендора.

Версия в этой инструкции: **Prometheus 3.13.2** (LTS), образ `quay.io/prometheus/prometheus:v3.13.2` (в чарте — `v3.13.2-distroless`). На Kubernetes — **kube-prometheus-stack 88.3.0**: Operator **v0.93.0**, Alertmanager **v0.33.1**. Kubernetes чарта ≥ 1.25.  
Документация: https://prometheus.io/docs/

Не подмешивать latest **3.14.0** и Alertmanager **0.34.0** в чарт 88.3.0 «на глаз».

HA Prometheus — **не** кластерный TSDB: две реплики снимают одни цели независимо (дубли сэмплов). Thanos/Mimir — только если решите отдельным проектом; этот файл их не ставит.

Один Prometheus, который scrape'ит все дата-центры, и один Alertmanager-кластер на несколько городов здесь **не собираем**. Порога RTT у проекта нет; `scrape_timeout` дефолт 10s; probe AM — 1s. Живой съём — внутри одной площадки. Между площадками — federation агрегатов.

---

## О чём эта инструкция молча договорилась

1. Один Prometheus CR и один AM-кластер живут **внутри одного дата-центра**. Между площадками — **иерархическая federation**: детальный съём локально, глобальный снимает агрегаты через `/federate`.
2. Бой — Kubernetes в каждом дата-центре отдельно (`Kubernetes.install.md`).
3. Учебный стенд — закрытая сеть. TLS можно отложить **только там**.
4. Цифр нагрузки нет — нет PVC «под терабайты озера». Диск считают от сэмплов/с × retention × 1–2 байта, когда появятся цифры. Дефолт retention: 15 дней у проекта, **10d** у чарта 88.3.0.
5. Thanos/Mimir **не обязательны**, чтобы считать Prometheus поставленным.
6. Две площадки: пара Prometheus + AM-кластер в каждой; глобальный federator — в первой. Три — то же в третьей + federator по-прежнему один (его тоже HA внутри его площадки). Третья площадка **не** входит в gossip AM первой.
7. NFS под TSDB официально **не поддерживается**. У каждой реплики свой диск.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** увидеть scrape, PromQL, одно правило. **Не зачем:** отказ дата-центра и кардинальность интеграционного API.

### Что должно быть до установки

- Docker Engine **или** однонодовый Kubernetes.
- На localhost свободен порт 9090.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь для учёбы)

```bash
docker run -d --name prometheus-dev \
  -p 127.0.0.1:9090:9090 \
  -v "$PWD/prometheus-dev.yml:/etc/prometheus/prometheus.yml:ro" \
  quay.io/prometheus/prometheus:v3.13.2
```

Привязка к `127.0.0.1` обязательна. Конфиг — first steps: снять сам себя с `/metrics`.

Проверка:

```bash
curl -s http://127.0.0.1:9090/-/healthy
curl -s 'http://127.0.0.1:9090/api/v1/query?query=prometheus_build_info'
```

В `prometheus_build_info` должна быть **3.13.2**. Рядом при желании Alertmanager 0.33.1 на `127.0.0.1:9093` — чтобы увидеть цепочку алерта, не только график.

### Установка (Kubernetes, если стенд уже в кластере)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack --version 88.3.0 \
  --set prometheus.prometheusSpec.replicas=1 \
  --set alertmanager.alertmanagerSpec.replicas=1
```

Дефолт чарта и так `replicas: 1`, retention **10d**. emptyDir на тесте допустим. **Не** этот values в бой. 9090 не LoadBalancer в интернет.

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Реплики Prometheus / AM | 1 | Не учимся переживать выкат |
| TLS / basic auth | можно без | Иначе PKI раньше PromQL |
| Federation / remote write | выкл | Нечего агрегировать |
| Pushgateway | выкл | Без batch-job — лишняя дыра |
| `honor_labels` | не на всём подряд | Снимает защиту подмены labels |
| Labels | без `user_id` / UUID | Привычка убьёт бой-TSDB |

Чего **не** упрощаем: версия **3.13.2**; хотя бы один `up==1` не про сам Prometheus; понимание — дефолт чарта **не HA**; admin API не включать «чтобы curl работал».

### Как понять, что стенд живой

1. `/-/healthy`, query `up` по demo-job.
2. Рестарт без PVC: история пропала — ожидаемо.
3. 9090 слушает только localhost.

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Часы, first steps / values чарта | Нет второго TSDB, нет gossip AM |
| Дешево показывает kube-ряды | Успех на одном поде ≠ готовность боя |
| | Привычка `replicas: 1` уедет в бой |

Перед боем полезен **препрод**: 2 Prometheus + ≥2 AM, block PVC, TLS — в **одном** дата-центре.

---

## Бой: свой съём в каждом дата-центре, глобальная картина — federation

**Зачем:** пережить отказ **пода или машины внутри площадки** (вторая реплика продолжает снимать); пережить отказ **целого дата-центра** ценой потери **детальных** рядов этой площадки. Глобальная картина — federator, не «один Prometheus на страну».

### Почему кластер не растягиваем на несколько дата-центров

Локальный TSDB **не кластеризуется**. Один процесс, который scrape'ит три площадки, при плохом RTT получает `DOWN` живых целей (`scrape_timeout` 10s). Gossip Alertmanager на WAN: fail-open = **дубли** писем. NFS под TSDB официально не поддерживается.

### Как расставить машины

**Внутри каждого дата-центра** — свой контур:

- Prometheus **`replicas: 2`**, одинаковые scrape/rules, разные `external_labels` (`cluster`, `replica`). `replicaExternalLabelNameClear: false` — чарт это прямо запрещает чистить для HA.
- PVC **по одному на под**, RWO, local/block SSD, **не NFS**. `retention.size` ≤ 80–85% тома. WAL compression вкл (чарт 88.3.0: `walCompression: true`).
- Alertmanager **`replicas: 2–3` все в этом дата-центре**; Prometheus шлёт алерты **на все** AM, **не** через LB.
- Gossip 9094 только внутри площадки.
- node_exporter DaemonSet с tolerations на tainted-ноды (иначе слепые самые важные ноды).
- Алерты «горит инстанс» — локально.

**Глобальный слой (в первой площадке):**

- Отдельный Prometheus (лучше тоже пара) снимает **агрегаты** (`job:`-ряды / recording rules) через `/federate` с `honor_labels: true`.
- Широкий `match[]` = второй полный TSDB — не делать.
- Алерты «горит платформа» — здесь, иначе дубли с локальными.

**Между площадками:**

| Сколько площадок | Что где | Если умер первый дата-центр |
|---|---|---|
| **Две** | В каждой: пара Prometheus + AM-кластер. В первой: ещё federator | Нет детальных рядов первой и глобальной картины, пока federator не во второй. Съём и алерты второй живы |
| **Три** | Третий такой же локальный контур; federator по-прежнему в одной площадке (HA внутри неё) | То же. Третья площадка не входит в AM-gossip первых |

Не делать: один Prometheus CR на три Kubernetes; AM-пиры между городами; LB перед Alertmanager; общий PVC на две реплики.

### Что должно быть до боевой установки

- Kubernetes в каждом дата-центре; сеть **Prometheus → /metrics целей этой площадки**.
- Зеркало образов 3.13.2 / AM 0.33.1 / operator 0.93.0.
- StorageClass block, не NFS.
- On-call и канал уведомлений. Без приёмника HA AM бессмысленна.
- Политика labels: запрет high-cardinality (`request_id`, ИНН) письменно.
- NTP на всех машинах.

### Порядок установки (каждый дата-центр)

```bash
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  --version 88.3.0 \
  --namespace monitoring --create-namespace \
  -f kps-prod-values.yaml
```

Смысл values (сверять с values 88.3.0):

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    retention: 10d
    walCompression: true
    externalLabels:
      cluster: dc1
    replicaExternalLabelNameClear: false
alertmanager:
  alertmanagerSpec:
    replicas: 3
```

Все три Alertmanager — ноды **этого** дата-центра. Retention — политика, не «потому что дефолт»; формула диска — отдельно.

CRD чарт **не обновляет** при обычном helm upgrade — мажор = ручное обновление CRD (README).

Дальше:

1. TLS + bcrypt (или mTLS) на 9090. Admin/lifecycle API выкл. Прокси режет `/api/*/admin/` и `/-/quit`.
2. ServiceMonitor на одно реальное приложение с `sample_limit`. Съём — ServiceMonitor, не аннотации `prometheus.io/scrape` как основной способ.
3. Recording + alerting rules с `for:` (ориентир практик проекта — не меньше 5 минут, если нет особой причины). Тестовый page в on-call.
4. Federator: отдельный scrape `/federate`, узкий `match[]`.
5. Бэкап: **снимки** TSDB, не `cp` каталога на живую. Учитывать дырку WAL (~часы).

### Правила конфигурации боя

| Параметр | В бою | Зачем |
|---|---|---|
| Диск TSDB | local/block, свой PVC на реплику | NFS — CAUTION проекта |
| `retention.size` | ≤ 80–85% PVC | Запас на compaction |
| `sample_limit` / `label_limit` | на опасных job | Лучше отвалить цель, чем положить TSDB |
| Admin/lifecycle API | выкл | Нет CSRF |
| `--web.config.file` | TLS + bcrypt или mTLS | Basic без TLS модель запрещает смыслом |
| Pushgateway | только эфемерные batch | Иначе honor_labels + открытый 9091 = подмена рядов |
| PDB | не убивать сразу обе реплики при drain | Иначе дырка съёма |
| ServiceMonitorSelector | понять, видит ли Prometheus чужие namespace | Дефолт чарта завязан на label релиза — «поставили exporter в другой namespace и тишина» |

Съём ваших систем (иначе платформа слепая): Kubernetes — kubelet, cAdvisor, kube-state-metrics, node_exporter; микросервисы — RED без `user_id`/UUID в labels; Kafka — JMX с лимитами, не «каждый топик×partition×consumer» без потолка; интеграционное API — число вызовов и ошибки, **без** тела ответа ведомства.

### Как расти, когда появятся цифры нагрузки

1. Меньше рядов / разумный interval — главный рычаг.
2. Functional shards (отдельный Prometheus на Kafka-платформу), не автошард «с порога».
3. Federation — официальный путь на много площадок, не stretch scrape.
4. Remote write / Thanos — когда нужна **одна** дедуплицированная история; не галочка дня один.
5. FAQ «десятки миллионов active series» — потолок порядка, не смета. Смотреть `prometheus_tsdb_head_series`, RAM, scrape duration vs timeout.

### Проверки, без которых это ещё не бой

1. Версия **3.13.2** на обеих репликах каждого дата-центра. Нет смеси с 3.14.0.
2. `up` по kube-компонентам; `alertmanager_cluster_members == N` **внутри** площадки.
3. Убить под Prometheus: алерты идут со второй реплики.
4. Тестовое правило с `for:` дошло до on-call.
5. Federator видит агрегаты, не полный набор instance-рядов.
6. Учение «первый дата-центр выключен»: локальный съём второй площадки жив; глобальная картина деградировала предсказуемо. Учение «NFS по ошибке» — не делать.
7. 9090 не из интернета. Labels без ИНН/`trace_id`.

### Что хорошо и что плохо в схеме «пара на площадку + federation»

| Хорошо | Плохо |
|---|---|
| Съём локальный: авария канала ≠ ложный DOWN всего | Два слоя правил; легко забыть, где живёт алерт |
| AM gossip не ходит по WAN | Глобальный Prometheus — SPOF картины, его тоже надо HA |
| Согласовано с «TSDB не кластер» | Без дедупа UI двух реплик «прыгает» — так устроена модель |
| | `/federate` с широким match = второй полный TSDB |

**Не готово к бою**, если: `replicas: 1` как единственный съём на все площадки; NFS-PVC; LB перед Alertmanager; `latest` / 3.14.0 в чарте 88.3.0; 9090 из интернета; `honor_labels` на всём; `trace_id` в labels; stretch AM на 2–3 ЦОДа; чарт «как есть» с одним подом; учебный манифест скопирован в бой.

---

## Откуда цифры и имена образов

- LTS 3.13.2: https://prometheus.io/download/
- Storage, не кластер, не NFS, 1–2 байта/сэмпл, 15d, compaction 80–85%: https://prometheus.io/docs/prometheus/latest/storage/
- Federation: https://prometheus.io/docs/prometheus/latest/federation/
- Alertmanager HA (не LB, 9094): https://prometheus.io/docs/alerting/latest/high_availability/
- kube-prometheus-stack 88.3.0: https://github.com/prometheus-community/helm-charts/tree/kube-prometheus-stack-88.3.0/charts/kube-prometheus-stack
- Правила и схема компонентов: `Prometheus.md`
