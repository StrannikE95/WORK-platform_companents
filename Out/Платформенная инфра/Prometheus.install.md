# Prometheus 3.13.2 LTS — установка и конфигурирование

Связанный документ (глоссарий, TSDB, federation, почему так): `Prometheus.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Prometheus (один scrape на все ЦОДы) и stretch кластера Alertmanager (gossip 9094) на несколько ЦОДов **не делаем**: порога RTT у проекта нет, `scrape_timeout` дефолт 10s, probe AM — 1s.

Версии: **Prometheus 3.13.2** (LTS), образ `quay.io/prometheus/prometheus:v3.13.2` (в чарте — `v3.13.2-distroless`). На Kubernetes — **kube-prometheus-stack 88.3.0**: Operator **v0.93.0**, Alertmanager **v0.33.1**. Kubernetes чарта ≥ 1.25.  
Документация: https://prometheus.io/docs/  
Не подмешивать latest **3.14.0** и Alertmanager **0.34.0** в чарт 88.3.0 «на глаз».

HA Prometheus — **не** кластерный TSDB: две реплики снимают одни цели независимо (дубли сэмплов) **или** Thanos/Mimir — только если решите отдельным проектом (источник их упоминает, этот файл их не ставит).

---

## Допущения этой инструкции

1. **Stretch запрещён.** Один Prometheus CR и один AM-кластер живут **внутри одного ЦОДа**. Между ЦОДами — **иерархическая federation** (вариант B в `Prometheus.md`): детальный съём локально, глобальный снимает агрегаты через `/federate`.
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`).
3. Dev — изолированная сеть; TLS можно отложить.
4. Нагрузки нет — нет цифры PVC «под терабайты озера». Диск считают от сэмплов/с × retention × 1–2 байта, когда появятся цифры.
5. Thanos/Mimir **не обязательны**, чтобы считать Prometheus поставленным.
6. Для 2 ЦОДов: пара Prometheus + AM-кластер в каждом; глобальный federator — в ЦОД-1. Для 3 ЦОДов: то же в третьем ЦОДе + federator по-прежнему один (его тоже HA внутри его ЦОДа). Третий ЦОД **не** входит в gossip AM ЦОД-1.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** увидеть scrape, PromQL, одно правило. **Не** цель: отказ ЦОДа и кардинальность интеграционного API.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 9090 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

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

### Установка (Kubernetes Dev)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack --version 88.3.0 \
  --set prometheus.prometheusSpec.replicas=1 \
  --set alertmanager.alertmanagerSpec.replicas=1
```

Дефолт чарта и так `replicas: 1`, retention **10d**. emptyDir на тесте допустим. **Не** этот values в прод. 9090 не LoadBalancer в интернет.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Реплики Prometheus / AM | 1 | Нет HA-цели |
| TLS / basic auth | можно без | Иначе PKI раньше PromQL |
| Federation / remote write | выкл | Нечего агрегировать |
| Pushgateway | выкл | Без batch-job — лишняя дыра |
| `honor_labels` | не на всём подряд | Снимает защиту подмены labels |
| Labels | без `user_id` / UUID | Привычка убьёт прод-TSDB |

Чего **не** упрощать: версия **3.13.2**; хотя бы один `up==1` не про сам Prometheus; понимание — дефолт чарта **не HA**.

### Проверка Dev

1. `/-/healthy`, query `up` по demo-job.
2. Рестарт без PVC: история пропала — ожидаемо.
3. Не включать admin API «чтобы curl работал».

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, first steps / values чарта | Нет второго TSDB, нет gossip AM |
| Дешёво показывает kube-ряды | Успех на одном поде ≠ готовность прода |
| | Привычка `replicas: 1` уедет в бой |

Препрод: 2 Prometheus + ≥2 AM, block PVC, TLS — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода/машины внутри ЦОДа** (вторая реплика продолжает снимать); пережить отказ **целого ЦОДа** ценой потери **детальных** рядов этой площадки. Глобальная картина — federator, не «один Prometheus на страну».

### Почему не stretch

Локальный TSDB **не кластеризуется**. Один процесс, который scrape'ит три ЦОДа, при плохом RTT получает `DOWN` живых целей (`scrape_timeout` 10s). Gossip Alertmanager на WAN: fail-open = **дубли** писем; mTLS обязателен по доке для WAN — но мы AM **не** растягиваем. NFS под TSDB официально **не поддерживается**.

### Топология

**Внутри каждого ЦОДа** — свой контур (вариант B):

- Prometheus **`replicas: 2`**, одинаковые scrape/rules, разные `external_labels` (`cluster`, `replica`);
- PVC **по одному на под**, RWO, local/block SSD, **не NFS**;
- Alertmanager **`replicas: 2–3` все в этом ЦОДе**; Prometheus шлёт алерты **на все** AM, **не** через LB;
- gossip 9094 только внутри ЦОДа;
- node_exporter DaemonSet с tolerations на tainted-ноды;
- алерты «горит инстанс» — локально.

**Глобальный слой (в ЦОД-1):**

- отдельный Prometheus (лучше тоже пара) снимает **агрегаты** (`job:`-ряды / recording rules) через `/federate` с `honor_labels: true`;
- широкий `match[]` = второй полный TSDB — не делать;
- алерты «горит платформа» — здесь, иначе дубли с локальными.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | В каждом: пара Prometheus + AM-кластер. В ЦОД-1: ещё federator | Нет детальных рядов ЦОД-1 и глобальной картины, пока federator не в ЦОД-2. Съём и алерты ЦОД-2 живы |
| **3 ЦОДа** | Третий такой же локальный контур; federator по-прежнему в одном ЦОДе (HA внутри него) | То же; третий ЦОД не входит в AM-gossip первых |

Не делать: один Prometheus CR на три Kubernetes; AM-пиры `ЦОД-1:9094` ↔ `ЦОД-2:9094`; LB перед Alertmanager; общий PVC на две реплики.

### Предпосылки прода

- Kubernetes в каждом ЦОДе; сеть **Prometheus → /metrics целей этого ЦОДа**.
- Зеркало образов 3.13.2 / AM 0.33.1 / operator 0.93.0.
- StorageClass block, не NFS.
- On-call и канал уведомлений. Без приёмника HA AM бессмысленна.
- Политика labels: запрет high-cardinality (`request_id`, ИНН) письменно.

### Установка (каждый ЦОД)

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
    retention: 10d          # политика, не «потому что дефолт»; формула диска — отдельно
    walCompression: true
    externalLabels:
      cluster: dc1
    replicaExternalLabelNameClear: false   # чарт запрещает чистить для HA
alertmanager:
  alertmanagerSpec:
    replicas: 3             # все три — ноды ЭТОГО ЦОДа
```

CRD чарт **не обновляет** при обычном helm upgrade — мажор = ручное обновление CRD (README).

### Конфигурирование (смысл)

| Параметр | Прод | Зачем |
|---|---|---|
| Диск TSDB | local/block, свой PVC на реплику | NFS — CAUTION проекта |
| `retention.size` | ≤ 80–85% PVC | Запас на compaction |
| `sample_limit` / `label_limit` | на опасных job | Лучше отвалить цель, чем положить TSDB |
| Admin/lifecycle API | выкл | Нет CSRF; прокси режет `/api/*/admin/` |
| `--web.config.file` | TLS + bcrypt или mTLS | Basic без TLS модель запрещает смыслом |
| Pushgateway | только эфемерные batch | Иначе honor_labels + открытый 9091 = подмена рядов |

Съём: ServiceMonitor, не аннотации `prometheus.io/scrape` как основной способ. Recording rules на тяжёлые дашборды **до** того, как Grafana молотит сырой PromQL.

Federator: отдельный scrape `/federate`, узкий `match[]`.

### Масштабирование (когда появятся цифры)

1. Меньше рядов / разумный interval — главный рычаг.
2. Functional shards (отдельный Prometheus на Kafka-платформу), не автошард «с порога».
3. Federation — официальный путь на много DC, не stretch scrape.
4. Remote write / Thanos — когда нужна **одна** дедуплицированная история; не галочка дня один.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия 3.13.2 на обеих репликах каждого ЦОДа.
2. `up` по kube-компонентам; `alertmanager_cluster_members == N` **внутри ЦОДа**.
3. Убить под Prometheus: алерты идут со второй реплики.
4. Тестовое правило с `for:` дошло до on-call.
5. Federator видит агрегаты, не полный набор instance-рядов.
6. Учение «ЦОД-1 выключен»: локальный съём ЦОД-2 жив; глобальная картина деградировала предсказуемо. Учение «NFS по ошибке» — не делать.

### Сильные / слабые стороны прод-схемы (пара на ЦОД + federation)

| Сильное | Слабое |
|---|---|
| Съём локальный: авария канала ≠ ложный DOWN всего | Два слоя правил; легко забыть, где живёт алерт |
| AM gossip не ходит по WAN | Глобальный Prometheus — SPOF картины, его тоже надо HA |
| Согласовано с «TSDB не кластер» | Без дедупа UI двух реплик «прыгает» — так устроена модель |
| | `/federate` с широким match = второй полный TSDB |

**Не готов к проду**, если: `replicas: 1` как единственный съём на все ЦОДы; NFS-PVC; LB перед Alertmanager; `latest` / 3.14.0 в чарте 88.3.0; 9090 из интернета; `honor_labels` на всём; `trace_id` в labels; stretch AM на 2–3 ЦОДа; чарт «как есть» с одним подом.

---

## Источники

- LTS 3.13.2: https://prometheus.io/download/
- Storage, не кластер, не NFS: https://prometheus.io/docs/prometheus/latest/storage/
- Federation: https://prometheus.io/docs/prometheus/latest/federation/
- Alertmanager HA (не LB, 9094): https://prometheus.io/docs/alerting/latest/high_availability/
- kube-prometheus-stack 88.3.0: https://github.com/prometheus-community/helm-charts/tree/kube-prometheus-stack-88.3.0/charts/kube-prometheus-stack
- Правила: `Prometheus.md`
