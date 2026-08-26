# Apache Flink 2.2.1 — термины и сокращения

Словарь к файлу `Apache Flink.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная JVM. JobManager и TaskManager — разные процессы (поды). Клиент сдаёт граф и обычно не остаётся «как сервис».

**Файл / каталог на диске** — рабочее состояние TM (RocksDB) на **локальном** диске; копия при чекпоинте — во внешнем checkpoint storage (S3/HDFS). `emptyDir` на TM переживёт рестарт пода только пока жив узел. `file:///flink-data/ha` из примеров оператора — том пода, **не** HA на 3 ЦОДа.

**TCP-порт** — **8081** REST API и Web UI JobManager (`rest.port` / `rest.bind-port`). **6123** RPC JobManager (`jobmanager.rpc.port`). blob / TM RPC / TM data часто **0** = ОС выдаёт свободный порт: в NetworkPolicy нельзя «разрешить только 6123». UDP нет. Клиенты и люди ходят на **8081**, не на RPC TM. На HA-сетапах TM **находят** лидера через HA-сервис, статическое «все знают 6123» — модель standalone без HA.

**ЦОД** — отдельная площадка. У Flink нет официального stretch-кластера с репликацией state по стойкам и нет rack-awareness как у Kafka `broker.rack`. Shuffle TM по ЦОДам = сеть между ЦОДами.

**RTT** — время туда-обратно. Shuffle и выгрузка чекпоинта едут по этой сети. Порога у проекта Flink нет. Мерить TCP между будущими TM и до object storage, не ICMP ping.

**TLS / SSL** — внутренний трафик (RPC Pekko, blob, TM data) и внешний REST — **разные** флаги: `security.ssl.internal.enabled` / `security.ssl.rest.enabled` и уточняющие `pekko.ssl.enabled`, `blob.service.ssl.enabled`, `taskmanager.data.ssl.enabled`. Штатный TLS JDK, не ГОСТ/СКЗИ. На старых линейках (1.18) оператор фиксировал конфликт mTLS REST + HA; на 2.2.1 проверять handshake оператора на стенде.

**Java** — док 2.2: **Java 17 рекомендуется** (дефолт Docker-образов с Flink 2.0). Java 11 поддерживается. Java 21 в 2.2 — **experimental**. Образ: `flink:2.2.1-java17`. Пин 17 на JM, TM и в job-jar.

**Job (задание)** — одна запущенная программа Flink: читает потоки, считает, пишет результат. Живёт, пока её не остановили (streaming) или пока не дочитала вход (batch).

**JobGraph** — внутренний граф операторов, который клиент/JobManager собирает из вашего `main()`.

**JobManager (JM)** — планирует задачи, следит за чекпоинтами, держит REST/UI. В один момент лидер **один**. Без HA это единая точка отказа. JM без HA: новые submit нельзя; running job **падают** (формулировка HA Overview). JM почти не масштабируют «от MB/s»; RAM под метаданные job.

**TaskManager (TM)** — на них крутятся операторы (source / map / window / sink). Падение TM — не гибель кластера, если есть чекпоинт; задание **перезапускается** с последнего снимка.

**Slot (слот)** — единица вместимости TM. `taskmanager.numberOfTaskSlots` — сколько параллельных кусков задания TM готов принять. Пустые TM без слотов job не ускоряют.

**Parallelism (параллелизм)** — на сколько параллельных задач режется оператор. Parallelism source **не выше** числа партиций топика Kafka (лишние субаски простаивают). Сначала партиции Kafka, потом Flink.

**Application mode** — на каждое приложение — свой мини-кластер. `main()` выполняется на JobManager. Для прода — основной режим: изоляция ресурсов. HA в application mode **не поддерживается** для multi-`execute()` приложений. Один `execute()` на приложение.

**Session mode** — один кластер, много заданий. Дешевле на тесте. В проде: шумный сосед, одна упавшая TM бьёт несколько job.

**SessionJob** — задание, которое оператор сажает в уже существующий session-кластер. CR `FlinkSessionJob`.

**Checkpoint (чекпоинт)** — автоснимок состояния задания + позиций во входных потоках. После сбоя Flink откатывает *managed state* и offsets на этот снимок и перечитывает вход. Lifecycle ведёт Flink, снимки частые. Если чекпоинтов нет, дефолтная restart strategy — **не рестартовать**. При включённом checkpointing дефолт — `exponential-delay`. Автоматическое HA-восстановление оператора идёт с **последнего чекпоинта**, не с периодических savepoint. Интервал «как в туториале 10 секунд» на большом RocksDB-state может не успевать; цифры интервала под вашу нагрузку нет.

**Savepoint** — снимок «для людей»: смена версии Flink, смена графа, переезд. Вы создаёте и удаляете сами. Дороже чекпоинта, зато переносимее. Canonical, если меняете backend. Путать с чекпоинтом — типичная причина «после релиза state не встал».

**State backend** — где **рабочее** состояние живёт *между* чекпоинтами: куча JVM (`HashMapStateBackend`) или диск RocksDB (`EmbeddedRocksDBStateBackend`). HashMap — дефолт, стенд с крошечным state. Прод с неизвестным/большим state — RocksDB + инкрементальные чекпоинты.

**Checkpoint storage** — куда **копия** состояния уезжает при чекпоинте: куча JobManager (`JobManagerCheckpointStorage`, только стенд / крошечное state) или файловая/объектная система (`FileSystemCheckpointStorage`). Для всех HA-сетапов — filesystem (S3, HDFS, GCS, смонтированный том). Падение JM при storage в куче = нет снимка.

**HA (JobManager)** — несколько JM, из них один лидер. Адрес лидера и указатели на чекпоинты лежат во внешнем HA-сервисе (Kubernetes или ZooKeeper) плюс каталог `high-availability.storageDir`. На K8s типичен Kubernetes HA (без ZooKeeper). Kubernetes HA хранит lease/ConfigMap **в etcd этого кластера**. Три изолированных K8s ≠ один HA-домен Flink.

**Standby JobManager** — запасной JM уже запущен. Ускоряет смену лидера. **Не** делает падение лидера прозрачным: задание всё равно перезапускается. Это формулировка оператора. `spec.jobManager.replicas` ≥ 2 сокращает failover JM, job всё равно restart.

**Exactly-once (у Flink)** — каждый входной факт **один раз повлияет на managed state**. Это не «каждый байт ровно один раз прошёл через сеть» и не «sink снаружи тоже магически идемпотентен». Для Kafka-sink нужен отдельный `DeliveryGuarantee`. JDBC «просто INSERT» при рестарте даст дубли.

**At-least-once** — после рестарта часть записей может обработаться повторно. Проще и быстрее, дубли возможны.

**Barrier / alignment** — служебные метки в потоке, которые «выравнивают» чекпоинт по всем входам. Для exactly-once alignment нужен.

**Unaligned checkpoint** — чекпоинт без полного выравнивания: быстрее при backpressure, но **нельзя** использовать для произвольного апгрейда графа / minor-апгрейда Flink (таблица checkpoints vs savepoints). По умолчанию не включать.

**RocksDB** — встроенная LSM-база на локальном диске TM. Состояние может быть больше кучи. Доступ медленнее heap. Единственный из стабильных backend'ов с **инкрементальными** чекпоинтами. Restore медленнее heap (док прямо). NFS как рабочий каталог TM не брать; штатный путь чекпоинтов — S3-compatible. Шифрования state на диске TM из коробки нет: LUKS / CSI encryption.

**LSM** — структура хранения RocksDB (логарифмически сливаемые SST-файлы).

**ForStStateBackend** — disaggregated state (SST на S3/HDFS). В доке 2.2 прямо: **experimental, not fully available for production**. Native S3 filesystem из 2.3 в прод 2.2.1 **не** берём.

**KafkaSource / KafkaSink** — актуальные API коннектора. Старые `FlinkKafkaConsumer` / `FlinkKafkaProducer` — наследие, в 1.15 producer уже deprecated.

**Apache Flink Kafka Connector 5.0.0** — совместим с Flink 2.1.x и **2.2.x**. Коннекторы выпускаются **отдельно** от ядра. Не апгрейдить одновременно Flink и версию коннектора. Прогон на вашем брокере **4.3.1** обязателен.

**DeliveryGuarantee** — гарантия sink в Kafka: `NONE` (дефолт sink!), `AT_LEAST_ONCE`, `EXACTLY_ONCE` (транзакции Kafka, коммит на чекпоинте). «Поставили Flink, значит exactly-once» — **неверно**. Видимость записей в топике при EXACTLY_ONCE **задерживается до чекпоинта**.

**transactionalIdPrefix** — префикс `transactional.id` для exactly-once sink. Должен быть **уникален** среди job на одном Kafka-кластере. Конфликт, если старый job ещё жив при restore на другой площадке.

**transaction.timeout.ms** — на брокере **больше**, чем max(длительность чекпоинта) + max(время рестарта). Иначе брокер абортит транзакцию → риск потери (док коннектора, `ProducerFencedException`). Потребители проекции: `isolation.level=read_committed`.

**uid оператора** — стабильное имя оператора в графе. Без него смена кода ломает восстановление state из savepoint. Проставлять **до** первого боевого savepoint.

**Watermark** — метка «события старше этого времени больше не жду» для окон по event-time. Не про HA. Если ведомство отвечает часами, «watermark 10 секунд» порежет опоздавшие факты.

**Backpressure** — оператор не успевает — тормозит тех, кто пишет в него. Это защита, не «Flink медленный».

**REST / Web UI** — управление и просмотр job. Порт **8081**. REST по умолчанию **не аутентифицирует** клиента. Док SSL: mutual TLS можно включить, но рекомендуемый паттерн — REST только на loopback/pod-local + sidecar-прокси (Envoy/NGINX) с нормальной аутентификацией. Компрометация REST = остановить job, снять savepoint, иногда влиять на submit.

**Operator (Kubernetes)** — программа: смотрит CR `FlinkDeployment` / `FlinkSessionJob` / `FlinkStateSnapshot` и делает выкат, savepoint, HA, откат. **Apache Flink Kubernetes Operator 1.15.0** (26 мая 2026), валидирован против Flink 2.2; матрица: `2.2.x, 2.1.x, 2.0.x, 1.20.x, 1.19.x`. Линейку **2.3.0** (25 июня 2026) анонс 1.15.0 **не перечисляет**. Nightlies `operator-docs-main` — незарелизенная ветка. Helm тег **1.15.0**, не `latest`. Падение оператора **не** роняет уже бегущие job. `replicas: 2` **без** leader election = два активных контроллера. Миграция с 1 реплики — через scale-to-zero. Оператор логирует diff CR (FLINK-30306): ключи MinIO/Kafka в YAML = ключи в логах.

**Native Kubernetes** — Flink сам создаёт поды JM/TM без оператора, либо Native mode оператора. Cluster lifecycle без оператора — на вас. Standalone mode оператора: док HA — `--host` с IP пода ломает leader election. Целевой прод файла — Native.

**upgradeMode** — `savepoint` / `last-state` (нужны живые HA metadata) для stateful прода; `stateless` только для job без state (на тесте, пока граф прыгает).

**Blue/green** — у оператора с 1.14: два кластера и удвоенные слоты на время, не бесплатно.

**RPO / RTO** — сколько данных готовы потерять / как быстро подняться. У Flink RPO ≈ «не дальше последнего **успешного** чекпоинта», не «ноль, как у Kafka `acks=all`». RPO ≠ 0. RTO restore с RocksDB — замер; документация не даёт формулы минут. Kafka retention короче дыры без чекпоинта → offset указывает в вычищенный лог.

**Shuffle** — пересылка записей между TM (network buffers). Ходит по внутренней сети (Pekko RPC / TM data).

**Pekko** — RPC-стек JM↔TM (наследник Akka в проекте).

**Плагин FS** — JAR из `opt/` → `plugins/` **в образе** до старта. **flink-s3-fs-presto** в доке S3 назван рекомендуемым *для чекпоинтов в S3*. MinIO: `s3.endpoint` + часто `s3.path.style.access: true`. Схема `s3p://` — Presto; `s3a://` — Hadoop FS для FileSink, если оба нужны сразу. Access key в `flink-conf` док считает discouraged относительно IAM.

**Externalized checkpoints RETAIN** — не удалять чекпоинт при cancel, пока нет отдельной политики savepoint.

**job-result-store** — каталоги оператор **не** чистит (workaround FLINK-27569): GC, **последний** каталог не трогать.

**Restart exponential-delay** — редкий сбой рестартуется быстро, лавина не долбит Kafka.

**`taskmanager.memory.process.size` / `taskmanager.memory.flink.size`** — задавать **явно**, не оба сразу (конфликт → возможен отказ старта). Для контейнера process.size = размер пода. Для RocksDB managed memory не обнуляют (в отличие от рекомендации zero managed memory для HashMap). Лимит = только `-Xmx` без модели Flink → OOM или отказ старта.

**Autoscaler оператора** — смотрит lag/utilization и меняет parallelism вершин. Включать **после** стабильного чекпоинта и метрик.

**`commit.offsets.on.checkpoint`** — commit оффсетов в Kafka, чтобы людям/лагам было видно progress. Оффсеты живут в **state Flink**. Падение commit не ломает целостность checkpoint (метрика `commitsFailed`).

**acks** — продюсер Flink подчиняется настройке брокера. Док коннектора: даже после ack возможна потеря, если брокер настроен слабо. Прод Kafka — `acks=all` / min.ISR=2.

**Kerberos / GSSAPI / SCRAM / OAUTH** — аутентификация к Kafka; иначе механизм из документа Kafka поверх TLS.

**CSI / IOPS** — диск TM для RocksDB с предсказуемым IOPS. Чекпоинты **не** кладут на RWO-том одной TM как единственную копию.

**PDB / anti-affinity / topology spread** — не снять все TM сразу; один узел не унёс половину параллелизма. Для варианта A — **не** размазывать TM по ЦОДам «для галочки HA».

**Webhook оператора** — Helm `--set webhook.create=false` только отладка.

**Flink CDC** — отдельный Apache-проект. Их HA проектируется отдельно.

**Schema Registry** — схемы Avro/JSON/Protobuf. Без схем стримы из 30+ интеграций разъедутся. Не часть Flink.

**SoT / озеро** — эталон клиентских данных. Flink держит *рабочее* состояние job. Чекпоинт — не «копия озера». Нечестная роль: «положили клиентов только в keyed state».

**Camunda / интеграционное API** — не исполняются Flink. Ходить в ведомства «вместо API» — сломать контур ожидания лагов.

**Лицензия Apache 2.0** — на ядро. License-server нет.

**Вариант A** — активный Flink в одном ЦОДе (или одном K8s), чекпоинты на хранилище, которое переживает этот ЦОД. Падение ЦОДа Flink = остановка обработки до restore в другом ЦОДе. Целевой «честный» для неизвестной сети.

**Вариант B** — TM и JM размазаны по 3 ЦОДам одного Kubernetes. Только после замера RTT shuffle. Зависит от stretch etcd.

**Вариант C** — три независимых Flink + разделение входных топиков. Ровно-один раз обработать общекластерный факт само не получится.

**Ververica / Amazon Managed Flink / Cloudera** — другие манифесты. Файл — Apache Flink 2.2.1.

Источники формулировок: глоссарий и тело `Apache Flink.md`. Новых порогов RTT и размеров диска здесь нет.
