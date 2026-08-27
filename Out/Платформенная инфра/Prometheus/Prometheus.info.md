# Prometheus 3.13.2 LTS — термины и сокращения

Словарь к файлу `Prometheus.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа: память и открытые файлы. Prometheus server — один процесс с одним каталогом TSDB на диске. Два процесса = две разные базы, не одна копия.

**Файл / каталог на диске** — то, что остаётся после рестарта. Каталог TSDB: WAL и блоки. Официально: локальная POSIX-ФС. **NFS (включая AWS EFS) не поддерживается**.

**TCP-порт** — номер в сети. Prometheus **9090**, Pushgateway **9091**, Alertmanager **9093**, gossip Alertmanager **9094** TCP+UDP, node exporter **9100**. 9092 проект специально не занимает (коллизия с Kafka).

**ЦОД** — отдельная площадка с серверами. «Три ЦОДа» — три зоны отказа.

**RTT** — время туда-обратно по сети. Порога в мс у проекта нет. Съём цели ограничен `scrape_timeout` (дефолт **10s**). Gossip Alertmanager пробивает пиров раз в **1s**.

**TLS** — шифрование байтов в сети. На 9090 — `--web.config.file`. На 9094 между ЦОДами документация требует **mTLS**. Basic Auth без TLS = пароль открытым текстом. Минимум TLS 1.2, цель оценки Qualys **A**. ГОСТ/СКЗИ не заявлено.

**CVE** — в файле отдельным номером не фигурирует. Модель безопасности: кто открыл HTTP Prometheus, видит все ряды. Эндпоинты не торчат в интернет.

**LTS** — линия с длинной поддержкой. В файле прод — **3.13.2** (релиз 29 июля 2026). **3.14.0** — latest, не LTS; в чарт 88.3.0 не подмешивать.

**Под (Pod)** — контейнер(ы) Kubernetes. Prometheus и Alertmanager — StatefulSet + PVC. Два реплики = два тома.

**PVC / RWO** — том одному поду. У каждой реплики Prometheus **свой** диск. Общий PVC на два реплики некорректен.

**CSI** — плагин Kubernetes, который создаёт диск для пода. Block / local SSD, не NFS StorageClass.

**Helm / чарт** — шаблон манифестов. В файле: **kube-prometheus-stack 88.3.0** (11 августа 2026): Operator **v0.93.0**, Prometheus **v3.13.2-distroless**, Alertmanager **v0.33.1**, node-exporter **v1.12.1-distroless**. Kubernetes **≥ 1.25**. Дефолт чарта: `replicas: 1`, retention **10d**.

**Метрика** — число, которое меняется во времени: сколько запросов, сколько секунд, сколько байт. Не лог и не событие Kafka.

**Time series (ряд)** — одна метрика + набор меток (labels) + значения с временем. `http_requests_total{job="api",code="500"}` — это **один** ряд.

**Label** — ключ-значение на ряде (`job`, `instance`, `code`). Каждый **уникальный набор** labels = новый ряд в RAM и на диске.

**Cardinality (кардинальность)** — сколько разных рядов получилось. `user_id` / UUID / номер заявки в label раздувает RAM head и диск.

**Sample (сэмпл)** — одно значение ряда в конкретный момент. Официально на диске в среднем **1–2 байта** на сэмпл.

**Scrape (съём)** — Prometheus сам ходит по HTTP на `/metrics` цели и **забирает** цифры (pull). Цель не пушит в него постоянно.

**scrape_interval** — как часто снимать. В спецификации конфига дефолт **1m**. В first steps — 15s.

**scrape_timeout** — сколько ждать ответа цели. Дефолт **10s**, не может быть больше interval. Если RTT съедает окно — цель будет `DOWN`.

**Target** — конкретный адрес съёма (`kafka-0:9308`, под, node-exporter).

**Job** — группа однотипных целей в `scrape_configs` (`job_name: kafka`).

**Service discovery (SD)** — Prometheus сам находит цели (Kubernetes, DNS, файлы), а не только список IP руками.

**kubernetes_sd_configs** — SD из API Kubernetes: поды, сервисы, ноды, endpoints.

**Relabeling** — переписать/выкинуть labels **до** записи в базу. Фильтр «не снимать мусор».

**honor_labels** — поверить labels, которые прислала цель. Нужно для federation и Pushgateway. Цель может притвориться чужой.

**PromQL** — язык запросов к рядам. Grafana и алерты говорят на нём. Считается **внутри** этого процесса Prometheus.

**TSDB** — локальная база Prometheus на диске. **Не кластер**: один процесс — один каталог. Цитата storage: local storage is not clustered or replicated.

**WAL** — журнал на диске «ещё не уехало в блок». После краша процесс его проигрывает. Сегменты по **128 МБ**; минимум три файла. Без снимка риск потерять WAL (последние ~2–3 часа).

**Block** — кусок TSDB обычно на **2 часа**. Потом компактируется в более длинные.

**Compaction** — слияние блоков на диске. Вендор: не занимать больше **80–85%** диска — остальное на двойное место во время compaction.

**Retention** — сколько хранить. Если не задать ни время, ни размер — **15d**. В kube-prometheus-stack 88.3.0 дефолт чарта — **10d**. `retention.size` ≤ 80–85% PVC.

**Head** — текущий кусок в памяти (ещё не закрытый 2-часовой блок).

**Active series** — ряды, которые сейчас живые в head. FAQ: один инстанс умеет **десятки миллионов** active series. Это потолок порядка, не замер контура.

**Remote write** — отдать снятые сэмплы ещё куда-то (долгое хранилище). Протокол 1.0 стабилен; 2.0 в доке — experimental.

**Remote read** — читать старые сэмплы извне. Официально **не** стабильный API. Чужая база не станет распределённым PromQL.

**Federation** — один Prometheus снимает **выбранные** ряды с другого через HTTP `/federate`. Официальный способ вырасти на много ЦОДов. Широкий `match[]` превращается во второй полный TSDB.

**Recording rule** — заранее посчитать тяжёлый PromQL и писать результат как новый ряд.

**Alerting rule** — условие на PromQL: если столько-то времени правда — отправить алерт в Alertmanager.

**`for:`** — сколько условие должно держаться, прежде чем алерт станет firing. Без `for` один неудачный scrape может дёрнуть тревогу. Практики проекта: ориентир **не меньше 5 минут**, если нет особой причины.

**Alertmanager** — другой бинарь той же экосистемы: группирует, молчит (silence), дедуплицирует, шлёт в почту/мессенджер/webhook. Алерты сами **не** пишутся на диск Alertmanager: их регулярно пересылает Prometheus.

**Silence** — «не орём по этому алерту до времени T». В HA-кластере Alertmanager силенсы **реплицируются** gossip'ом. В чарте retention силенсов по умолчанию **120h**.

**Gossip / Memberlist (Alertmanager)** — обмен между процессами Alertmanager на **9094**. Кворума/голосования **нет**: жив один — уведомления идут. Дизайн **fail-open**: при разрыве сети лучше **два** письма, чем ноль. Advertise-address — **IP:port**, не hostname. По умолчанию gossip **без шифрования**. `--cluster.peer-timeout` дефолт **15s**.

**Fail-open** — при split (сеть между ЦОДами) Alertmanager шлёт дубли. Это цель дизайна.

**Pushgateway** — промежуточный процесс, куда **короткие** batch-джобы пушат метрики, а Prometheus их снимает. Порт **9091**. Не замена pull для обычных сервисов. Открытый Pushgateway + `honor_labels` = можно нарисовать любой ряд.

**Exporter** — процесс рядом с ПО, которое само не умеет `/metrics`: переводит JMX/SNMP/систему в формат Prometheus.

**Node exporter** — метрики машины: CPU, диск, FS, сеть. Порт **9100**. На Linux/Unix. В Kubernetes — DaemonSet с **tolerations** на tainted-ноды.

**kube-state-metrics** — метрики **объектов** Kubernetes (ReplicaSet, статус PVC), не нагрузка ноды.

**cAdvisor / kubelet** — метрики контейнеров/ноды. Часто отдельный Service в `kube-system`.

**Prometheus Operator** — контроллер Kubernetes: вы описываете CR (`Prometheus`, `ServiceMonitor`, `PrometheusRule`), он собирает конфиг и поды. Падение Operator не останавливает уже запущенный Prometheus, но останавливает применение новых ServiceMonitor/Rule. В чарте — **один** под.

**CR / CRD** — объект своего типа в API Kubernetes. Не файл на диске Prometheus.

**ServiceMonitor / PodMonitor** — CR «сними вот этот Service/Pod». Оператор **не** использует аннотацию `prometheus.io/scrape` как основной способ.

**PrometheusRule** — CR с alerting/recording rules. Admission webhook правил в чарте включён: выкат ломается при мёртвом Operator.

**Shard** — разрезать цели между несколькими Prometheus: каждый снимает **свою** часть. Это масштаб съёма, не «один TSDB на троих». Operator предпочитает функциональные шарды (сервисы A/B vs D/E), не автошард по адресу.

**Replica (HA Prometheus)** — два (и больше) **одинаковых** сервера снимают **одни и те же** цели. Это отказоустойчивость, не шардирование. Базы у них **разные**. FAQ: *run identical Prometheus servers on two or more separate machines*.

**external_labels** — метки, которые экземпляр добавляет, когда говорит с внешним миром (Alertmanager, federation, remote write). Ими отличают реплики (`replica: A` / `replica: B`). Чарта запрещает чистить replica external labels (`replicaExternalLabelNameClear`) для HA.

**Grafana** — рекомендованный проектом UI для графиков. **Другое ПО**. Этот документ его не настраивает.

**Thanos / Mimir** — внешнее хранилище и слой дедупликации запросов. **Не** часть бинаря Prometheus. Без него два HA-реплики дают два слегка разных TSDB; графики могут «прыгать».

**Metrics Server / Prometheus Adapter** — чтобы HPA в Kubernetes ел кастомные метрики. Чарта kube-prometheus-stack **явно не ставит** Adapter.

**`up`** — ряд «цель ответила на scrape». `up=0` = цель недоступна (если Prometheus жив).

**`prometheus_tsdb_head_series`** — сколько живых рядов в head. Сигнал кардинальности и RAM.

**`--web.enable-admin-api`** — удаление рядов. По умолчанию **выкл**. CSRF на admin-ручке нет специально (чтобы curl работал).

**`--web.enable-lifecycle`** — `/-/reload`, `/-/quit`. По умолчанию **выкл**.

**`--web.listen-address`** — дефолт `0.0.0.0:9090`.

**`sample_limit` / `label_limit` / `target_limit`** — потолки на job: лучше отвалить цель, чем положить весь TSDB.

**WAL compression** — сжатие WAL. В 3.x норма; чарт 88.3.0: `walCompression: true`.

**bcrypt** — способ хранить пароль HTTP basic auth (хеш, не открытый текст).

**Blackbox / SNMP exporter** — процесс, который по указанному URL/адресу сам ходит «проверить цель». Кто дошёл до exporter, может заставить его сходить куда угодно.

**Sticky session** — на Service перед двумя Prometheus: графики не прыгают между двумя TSDB. Дедупа рядов это не даёт.

**Anti-affinity / topology spread** — не класть обе реплики Prometheus в один ЦОД.

**PDB** — не убивать сразу обе реплики при drain.

**ServiceMonitorSelector** — какие ServiceMonitor видит Prometheus. Дефолт чарта завязан на label релиза.

**Image tag `latest`** — не брать. Прод: pin `v3.13.2-distroless` и digest, свой registry.

**Apache 2.0** — лицензия. License-server нет. Сборка бинарей у проекта на чужой CI.

**evaluation_interval** — как часто считать rules. Дефолт конфига **1m**.

**nflog** — журнал уведомлений Alertmanager на диске PVC (вместе с силенсами).

**Split-brain (Alertmanager)** — сеть между членами разорвана: каждый шлёт. Дубли ожидаемы. Смотреть `alertmanager_cluster_members`, `alertmanager_cluster_failed_peers`.

**OOM** — процесс убит из-за памяти. Часто тяжёлый PromQL / compaction, не `up==1`.

**cgroup** — лимит памяти/CPU, который Kubernetes ставит на контейнер. Throttle CPU → отставание scrape и правил.

**`insecure_skip_verify`** — не проверять сертификат цели. Только стенд.

**SIEM / логи / трейсы** — FAQ: логи в Prometheus не класть. Рядом Loki/OpenSearch/Tempo.

**SoT** — Prometheus не источник истины клиентов. Наблюдает числа, не заменяет Kafka/озеро.

Источники формулировок: глоссарий и тело `Prometheus.md`. Новых порогов RTT и размеров диска здесь нет.
