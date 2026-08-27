# Falco 0.44.1 — установка (учебный контур)

**Допущение:** один учебный Kubernetes ≥ 1.29, Linux-ноды (x86_64 или ARM64), закрытая сеть. Не бой и не stretch на несколько дата-центров.

Falco — агент на ноде: читает системные вызовы (вход процесса в ядро: открыть файл, запустить программу, открыть сокет) и пишет тревогу. Сам атаку не останавливает.

| Что пиним | Значение |
|---|---|
| Falco | **0.44.1**, образ `falcosecurity/falco:0.44.1` |
| Operator | **v0.4.1**, чарт `falcosecurity/falco-operator` **0.3.0** |
| Драйвер | **`modern_ebpf`** (программа в ядре, встроена в бинарник; без сборки `.ko`) |
| Режим | **DaemonSet** — по одному поду на каждую ноду |
| Правила | OCI `falco-rules` **5.1.0** (в составе 0.44.0/0.44.1) |
| Плагин контейнера | **0.7.1** (анонс 0.44.0; патч 0.44.1 плагин не перечисляет) |

Если Kubernetes **старше 1.29** — не Operator (нужны native sidecars), а Helm `falcosecurity/falco` **9.1.0** (тот же 0.44.1). Windows-ноды Falco не покрывает.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** В **этот** учебный Kubernetes, не на отдельную Docker-VM и не «один Falco на три зала». Съём идёт с ядра **этой** машины: чужую ноду агент не видит. Deployment с `replicas: 2` для ядра не использовать — ложное HA и слепые ноды.

**Helm** — программа, которая ставит набор манифестов Kubernetes из «чарта» (пакет с шаблонами). **kubectl** — программа, которая говорит API-серверу Kubernetes: создать/показать/удалить объект.

**Сколько железа.** Таблицы вендора «минимум N ядер / M ГиБ, чтобы процесс поднялся» **нет**. Не выдумывать смету под Kafka/Camunda.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум «процесс поднялся» | **в доке нет** | **в доке нет** | не склад данных | — |
| Дефолт Operator, DaemonSet (запрос / лимит на контейнер `falco`) | **100m / 1000m** | **512Mi / 1024Mi** | локальный диск ноды только если включён файловый выход | [configuration.md](https://github.com/falcosecurity/falco-operator/blob/v0.4.1/docs/configuration.md) |
| Дефолт Helm `falco` 9.1.0 | те же 100m / 1000m | те же 512Mi / 1024Mi | то же | ArtifactHub: «subjective on the actual workload» |
| Operator Deployment (сам контроллер) | запрос **10m**, лимит **500m** | **64Mi / 128Mi** | нет PVC | ArtifactHub `falco-operator` 0.3.1 (values; `replicaCount` по умолчанию **1**) |
| Учебный ориентир по шуму syscall | не ядра VM | не ГиБ | — | troubleshooting: **< ~1–1.5 тыс. событий/с на одно CPU** обычно терпимо, **> ~3 тыс./с** часто тяжело — «grain of salt», не смета стенда |

Жёсткий CPU limit «чтобы не мешал» → ядро не успевает записать вызов в буфер (**дроп**) → Falco этого вызова **не видел**.

**Сильная сторона:** официальный путь Operator, драйвер тот же, что планируете в бой (`modern_ebpf`), стенд за часы.  
**Слабая сторона:** детект `exec` на простой ноде не доказывает детект на нагруженном брокере; HTTP до Sidekick и UI — учебный каркас.

**Критично:**

- Порты **8765** (healthz/метрики агента), **2801** (приём JSON у Falcosidekick), **2802** (UI Sidekick) — не LoadBalancer в интернет.
- NFS как диск Falco вендор **не описывает**; агенту общий том не нужен.
- Не ставить теги **`latest`** (quickstart оператора так делает — не копировать).
- «Кластер Falco» между дата-центрами **нечего растягивать**: нет кворума, порога RTT в доке нет. Два Kubernetes = две установки.

---

## Установка для новичка

Официальная страница: https://falco.org/docs/setup/operator/

### Что должно быть до установки

Есть:

- Учебный Kubernetes **≥ 1.29**, Linux-ноды.
- `kubectl` с правами ставить CRD и ClusterRole.
- Helm 3.
- Сеть до образов (`docker.io/falcosecurity/…`, `ghcr.io` для OCI-правил/плагинов) или зеркало.
- На ядре: **BTF** (таблица типов ядра) и **BPF ring buffer** — иначе `modern_ebpf` не встанет, и вы отладите не тот драйвер.

Нет (и не требуется на этом стенде): веб-консоль у самого Falco, учётка агента, PVC, TLS до Sidekick, Talon (убить под).

Проверка ядра (если есть `bpftool`):

```bash
sudo bpftool feature probe kernel | grep -q "map_type ringbuf is available" && echo "ringbuf: true" || echo "ringbuf: false"
```

Страница драйверов: https://falco.org/docs/concepts/event-sources/kernel/

### Этапы (Operator — основной путь)

**1. Поставить Operator**

**Что делаем:** контроллер в namespace `falco-operator` начинает следить за объектами Falco/правил/плагинов.

Чарт **0.3.0** в репозитории оператора v0.4.1 имеет `appVersion: 0.4.0`. Образ оператора пиним явно **0.4.1**.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco-operator falcosecurity/falco-operator \
  --version 0.3.0 \
  --namespace falco-operator --create-namespace \
  --set image.tag=0.4.1
```

Успех: `kubectl wait pods --for=condition=Ready --all -n falco-operator`.

**2. Экземпляр Falco — DaemonSet, версия 0.44.1**

**Что делаем:** на каждой Linux-ноде появляется под `falco`. Пока нет правил — агент в idle (драйвер есть, детекта нет).

```bash
kubectl create namespace falco
kubectl apply -f - <<'EOF'
apiVersion: instance.falcosecurity.dev/v1alpha1
kind: Falco
metadata:
  name: falco
  namespace: falco
spec:
  type: DaemonSet
  version: "0.44.1"
EOF
```

Успех: `kubectl get falco -n falco` и поды `Running`; в статусе версия **0.44.1**. В логах не должно быть отката на kmod.

**3. Плагин контейнера, затем правила** (порядок обязателен: иначе поля `container.*` в штатных правилах мертвы)

**Что делаем:** агент начинает понимать, из какого контейнера вызов; затем включается набор правил 5.1.0.

```bash
kubectl apply -f - <<'EOF'
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Plugin
metadata:
  name: container
  namespace: falco
spec:
  ociArtifact:
    image:
      repository: falcosecurity/plugins/plugin/container
      tag: "0.7.1"
    registry:
      name: ghcr.io
---
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Rulesfile
metadata:
  name: falco-rules
  namespace: falco
spec:
  priority: 50
  ociArtifact:
    image:
      repository: falcosecurity/rules/falco-rules
      tag: "5.1.0"
    registry:
      name: ghcr.io
EOF
```

Успех: `kubectl get plugins,rulesfiles -n falco` — Reconciled/Available; в `kubectl logs -n falco -l app.kubernetes.io/name=falco -c falco --tail=30` правила загружены.

**4. Приёмник на стенде (stdout уже включён дефолтом оператора)**

**Что делаем:** опционально Falcosidekick — отдельный процесс: принимает JSON по HTTP и раздаёт дальше. На учёбе 1 реплика, HTTP только внутри кластера. Пример оператора: версия Sidekick **2.32.0**, порт **2801**.

```bash
kubectl apply -f - <<'EOF'
apiVersion: instance.falcosecurity.dev/v1alpha1
kind: Component
metadata:
  name: sidekick
  namespace: falco
spec:
  replicas: 1
  component:
    type: falcosidekick
    version: "2.32.0"
---
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Config
metadata:
  name: sidekick-output
  namespace: falco
spec:
  priority: 60
  config:
    json_output: true
    http_output:
      enabled: true
      url: "http://sidekick:2801"
EOF
```

Успех: под Sidekick Running; `http_output` с **самоподписанным** сертификатом Falco **не** примет (на этом стенде HTTP, не TLS).

**5. Контролируемое событие**

**Что делаем:** в тестовом поде читаем чувствительный файл — штатное правило должно сработать. **kubectl exec** — API Kubernetes запускает процесс внутри контейнера.

```bash
kubectl create deployment nginx --image=nginx
kubectl exec -it "$(kubectl get pods --selector=app=nginx -o name | head -n 1)" -- cat /etc/shadow
kubectl logs -n falco -l app.kubernetes.io/name=falco -c falco | grep -E "Warning|Critical"
```

Официальный сценарий: https://falco.org/docs/getting-started/falco-kubernetes-quickstart/  
Генератор событий проекта (шумнее, часть действий трогает `/bin` `/etc`): https://falco.org/docs/concepts/event-sources/kernel/sample-events/

Успех: строка алерта в логах **того** пода Falco, который на ноде с nginx. Если поднимали Sidekick — тот же JSON доходит на `:2801`.

### Если Kubernetes < 1.29

```bash
helm install falco falcosecurity/falco --version 9.1.0 \
  --namespace falco --create-namespace \
  --set image.tag=0.44.1 \
  --set driver.kind=modern_ebpf
```

Чарт по умолчанию уже DaemonSet. `driver.kind` в 9.1.0 по умолчанию `auto` — для учёбы фиксируем `modern_ebpf`.

### Чего этот стенд не доказывает

Отказ зала, выборы лидера (их нет), ёмкость буферов на ноде Kafka, TLS и PKI, покрытие tainted-нод, что Falco *остановит* атаку. Quickstart оператора с `latest` + UI + Redis — не этот файл.

Не удалять объект `Falco` раньше Plugin/Rulesfile/Config: sidecar не снимет finalizer.

---

## Первый запуск — URL, порт, учётка, смена пароля

У **самого Falco веб-консоли нет.** Живость:

| Что | Как | Учётка |
|---|---|---|
| Агент | логи контейнера `falco`; HTTP **8765** `/healthz` и Prometheus-метрики (дефолт Operator, `webserver`) | нет |
| gRPC-выход | **удалён в 0.44** | — |
| Falcosidekick | ClusterIP **2801**; `/healthz` → 200 | нет |
| Falcosidekick UI (если включили) | ClusterIP **2802**; port-forward → http://127.0.0.1:2802 | **`admin` / `admin`** |

`admin`/`admin` — **только закрытый стенд**. В бою UI не публиковать; если нужен — свой секрет `FALCOSIDEKICK_UI_USER` (формат `login:password`), не git. У агента Falco пароля нет — менять нечего.

Проверка Sidekick (из пода в том же namespace):

```bash
kubectl -n falco exec deploy/nginx -- true 2>/dev/null || true
kubectl -n falco port-forward svc/sidekick 2801:2801
# в другом терминале:
curl -sS http://127.0.0.1:2801/healthz
```

Имя Service смотрите своим `kubectl -n falco get svc`.

---

## Подключение к своей системе

Falco не «подключают JDBC». Клиент — **не** ваше приложение. Клиент канала алертов — **Falcosidekick** (или сборщик логов ноды). Дальше в этой платформе — **Wazuh (SIEM)** и/или NATS; UI Sidekick SIEM не заменяет.

| Канал | Протокол / порт | Кто клиент | Куда секрет |
|---|---|---|---|
| Запасной | stdout / syslog (дефолт Operator: оба включены) | агент сбора логов ноды | не нужен |
| Основной на стенде | HTTP JSON POST → **2801** | поды Falco → Sidekick | URL/токен приёмника SIEM — Secret, не git |
| Бой (не этот файл) | HTTPS, **валидный** сертификат; invalid/self-signed Falco отвергает | то же | сертификаты и webhook в Secret / Vault |

В git: манифесты CR, pin версий, overlay правил без токенов.  
Не в git: URL с токеном, basic UI, клиентские сертификаты.

Falco **не** WAF (это SafeLine, HTTP на периметре), **не** замена SIEM (это Wazuh), **не** NetworkPolicy, **не** сканер образов до запуска, **не** продюсер Kafka/NATS из коробки (0.44). Два/три Kubernetes = две/три установки; склеивают алерты, не процессы на нодах.

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз Falco 0.44.1 (11 июня 2026), образы | https://github.com/falcosecurity/falco/releases/tag/0.44.1 |
| Установка Operator, DaemonSet vs Deployment, K8s 1.29+, quickstart | https://falco.org/docs/setup/operator/ |
| Operator v0.4.1 | https://github.com/falcosecurity/falco-operator/releases/tag/v0.4.1 |
| Матрица: с v0.4.0 default Falco = 0.44.1 | https://github.com/falcosecurity/falco-operator/blob/main/docs/version-matrix.md |
| Дефолты DaemonSet: `modern_ebpf`, ресурсы 100m/512Mi, webserver **8765** | https://github.com/falcosecurity/falco-operator/blob/v0.4.1/docs/configuration.md |
| `spec.version`, пример образа 0.44.1 | https://github.com/falcosecurity/falco-operator/blob/main/docs/crds/falco.md |
| Getting started: плагин → правила; Sidekick 2.32.0, URL `:2801` | https://github.com/falcosecurity/falco-operator/blob/v0.4.1/docs/getting-started.md |
| Helm Operator (values, `replicaCount: 1`); на ArtifactHub линия 0.3.x / appVersion 0.4.1 | https://artifacthub.io/packages/helm/falcosecurity/falco-operator |
| Helm `falco` **9.1.0** → appVersion **0.44.1**, ресурсы, `driver.kind` | https://artifacthub.io/packages/helm/falcosecurity/falco |
| modern eBPF, BTF/ringbuf, capabilities | https://falco.org/docs/concepts/event-sources/kernel/ |
| Дропы (`ignore` / `log` / `alert` / `exit`) | https://falco.org/docs/concepts/event-sources/kernel/dropped-events/ |
| Буферы; ориентир событий/с на CPU | https://falco.org/docs/troubleshooting/dropping/ |
| HTTP/HTTPS, запрет invalid/self-signed; gRPC в 0.44 нет | https://falco.org/docs/concepts/outputs/channels/ |
| Триггер `cat /etc/shadow`; UI **2802**, default `admin`/`admin` | https://falco.org/docs/getting-started/falco-kubernetes-quickstart/ |
| event-generator | https://falco.org/docs/concepts/event-sources/kernel/sample-events/ |
| Sidekick listen **2801** | https://github.com/falcosecurity/falcosidekick |
| UI default `admin:admin` | https://github.com/falcosecurity/falcosidekick-ui |
| Анонс 0.44.0: rules **5.1.0**, container plugin **0.7.1**, удаление legacy eBPF и gRPC | https://falco.org/blog/falco-0-44-0/ |
| Зачем продукт, порты, роль | `Falco.md` |
| Словарь | `Falco.info.md` |
| Стыковка с платформой | `Falco.shema.md` |
| Карточка консультанта | `Falco.consultant.md` |

**В доке вендора нет (не угадано):** минимум CPU/RAM/диск «чтобы процесс поднялся»; millicores «хватит агенту на брокер»; порог RTT между ЦОД; NFS как том Falco; пароль/учётка у агента Falco; встроенный продюсер Kafka/NATS у 0.44; парный тег `artifact-operator` к v0.4.1 (в configuration.md дефолт образа sidecar — **`latest`**).
