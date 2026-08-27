# SonarQube Server 2026.1.5 LTA — термины и сокращения

Словарь к файлу `SonarQube.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа: память и открытые файлы. В Community/Developer/Enterprise Web + Compute Engine + вшитый Elasticsearch — **один** процесс. В DCE — отдельные процессы application node и search node.

**Файл / каталог на диске** — то, что остаётся после рестарта. У search-ноды — индекс Elasticsearch на PVC. Источник истины настроек и истории анализов — PostgreSQL, не этот индекс.

**TCP-порт** — номер в сети. UI/API **9000**, app→search **9001**, search↔search **9002**, app↔app (Hazelcast) **9003**. PostgreSQL — **5432** (не часть образа).

**ЦОД** — отдельная площадка с серверами. Search nodes вендор разрешает разносить по AZ **одной region**. Порога RTT в миллисекундах нет.

**RTT** — время туда-обратно. Критичны пути app↔search :9001/:9002, search↔search :9002, JDBC :5432.

**TLS** — шифрование байтов в сети. HTTPS на reverse proxy перед 9000. Между app и search в DCE — `nodeEncryption` / PKCS#12. JDBC к Postgres — `ssl=true` / `verify-full` платформы. ГОСТ/СКЗИ не заявлено.

**CVE** — в файле отдельным номером не фигурирует. Компрометация Admin = карта кода, токены CI, секреты в findings.

**EOL / окно поддержки** — LTA 2026.1: активная поддержка до **30 января 2027**, security-фиксы до **2 августа 2027** (endoflife.date, сверка 7 августа 2026). Latest после следующего релиза стандартную поддержку предыдущего теряет.

**LTA (Long-Term Active)** — «долгая» линейка Server: первый релиз года. Для прода в файле — **2026.1.5**, не Latest **2026.4.1**.

**Latest** — промежуточный релиз каждые ~2 месяца. Helm-чарт `sonarsource/sonarqube` / `sonarqube-dce` **2026.4.1** ставит Latest, не LTA. В 2026.4 — переезд на ES 9 с пересборкой индекса.

**Под (Pod)** — контейнер(ы) Kubernetes. Search — StatefulSet + PVC. Application — Deployment (можно HPA).

**PVC / RWO** — том только одному поду. Search-ноды без постоянного диска после рестарта **теряют индекс**. Access mode в чарте — `ReadWriteOnce`. PVC search часто зональный: под после бинда не переедет в другой ЦОД.

**NFS / SMB / NAS** — вендор: не использовать как диск search. Нужен локальный SSD.

**Helm / чарт** — `sonarqube` (Community/Developer/Enterprise) и `sonarqube-dce` (только DCE). С 2026.1 встроенный PostgreSQL-subchart **удалён**. Нет `jdbcOverwrite` → H2 (тест) или инстанс не о том.

**SAST** — статический анализ: смотрит **исходный код** в момент сборки CI, не то, что процесс делает в рантайме.

**SonarQube Server** — коммерческий продукт (Developer / Enterprise / Data Center). Лицензия **на инстанс в год** от объёма **LOC**.

**SonarQube Community Build** — бесплатная отдельная линейка. Не путать с Server. У Community **нет** LTA: поддерживается только свежий релиз. На загрузках: **26.8.0.126808**; чарт 2026.4.1 по умолчанию тянет Community **26.7.0.124771**.

**LOC (Lines of Code)** — лицензионная единица Server: сколько строк кода инстанс имеет право анализировать. Не терабайты озера.

**Scanner (сканер)** — программа **в CI** (Maven/Gradle/CLI/.NET/npm). Сканирует репозиторий и **отправляет отчёт** на сервер HTTP 9000. Сервер сам репозиторий с диска разработчика не читает.

**Quality Gate** — порог «можно сливать / нельзя»: покрытие, баги, уязвимости. В коммерческих редакциях умеет **блокировать merge** PR.

**Quality profile** — набор правил языка (Java, Python, …).

**Web process** — процесс UI и REST API. Принимает людей и отчёты сканеров. Порт **9000** (`sonar.web.port`).

**Compute Engine (CE)** — фоновый движок внутри application: берёт отчёт сканера из очереди и превращает его в issues/метрики. Узкое место «сканы зависли в очереди».

**Background task** — задача в очереди CE. Смотреть: Administration → Projects → Background Tasks (`pending_count` / `pending_time`).

**CE worker** — сколько отчётов CE обрабатывает параллельно. Менять число — **Enterprise и выше**, в UI. На DCE число **глобальное**, на каждом application node оно **повторяется** (4 workers × 2 node = 8 после рестарта всех app). Память CE (`sonar.ce.javaOpts`) делится между workers.

**Elasticsearch / Search node** — поисковый индекс issues и кода на диске. В Developer/Enterprise **вшит в тот же процесс**. В DCE — **отдельные** search-ноды, это отдельный ES-кластер. Порты 9001 (app→search) и 9002 (search↔search).

**Application node** — нода DCE с Web + CE. Масштаб «больше одновременных сканов и UI» = больше этих нод.

**DCE (Data Center Edition)** — единственная редакция Server, в которой вендор даёт **кластер и HA** самого SonarQube. Дефолтная топология: **2 application + 3 search**. Цитата: один app и один search можно потерять без влияния на пользователей. HA в Community/DE/EE **нет**.

**Hazelcast** — встроенная шина между application nodes (порт **9003**). Ставить Hazelcast отдельно **не нужно**.

**JWT** — токен сессии в cookie. Поэтому балансировщику **не** нужны sticky sessions — так в требованиях DCE. `applicationNodes.jwtSecret` — свой, одинаковый на всех app nodes.

**H2** — встроенная файловая БД. С 2026.1 Helm больше не кладёт PostgreSQL в чарт. H2 — только тест. В проде **запрещена** вендором (нет репликации, HA, нормальных бэкапов).

**JDBC** — строка подключения к внешней БД (`jdbc:postgresql://…`). В Helm 2026.1+ обязательный `jdbcOverwrite.*`. Пароль — из Secret, не plaintext (поле пароля в values deprecated).

**Force user authentication** — «без логина нельзя». По умолчанию **включено**. Выключение открывает кусок API анонимно (`api/users/search`, `api/system/status`, …).

**Token (`sonar.token` / `SONAR_TOKEN`)** — ключ для сканера и API вместо пароля. `sonar.login`/`sonar.password` — deprecated.

**JIT / SCIM** — JIT — пользователь создаётся при первом входе через IdP. SCIM — автосинхронизация пользователей/групп (Entra/Okta: **Enterprise+**).

**sonar-administrators** — встроенная группа админов. Вендор советует переименовать: при синхронизации групп с IdP это типичный путь захвата.

**Branch / PR analysis** — анализ веток и pull request, не только `main`. Это **Developer+**, в Community Build этого фокуса нет.

**vm.max_map_count** — лимит ядра Linux на mmap. Для ES в доке 2026.1: **≥ 524288** (не старый 262144 из части гайдов Docker). Ещё: `fs.file-max ≥ 131072`, `nofile ≥ 131072`, `nproc ≥ 8192`, seccomp, `/tmp` writable.

**High disk watermark** — ES при ~**90%** диска начинает экономить, при **95%** может закрыть индексы на запись.

**Active-Cold Standby** — официальный пример DR: второй кластер **холодный**, БД реплицируется, трафик переключается. **Не** Active/Active. Read-only реплика БД для живого SonarQube **не поддерживается**.

**Forced Elasticsearch reindex** — после failover Kubernetes-кластера вендор требует **принудительную переиндексацию ES**. UI может встать, данные в PostgreSQL живы — индекс надо отстроить заново.

**Кворум search (три ноды)** — большинство из трёх процессов Elasticsearch. Потеря двух ЦОДов, где жили search, снимает этот кворум. Вендор **не** обещает «два ЦОДа из трёх».

**`admin` / `admin`** — первый вход продукта, сразу просит смену.

**Helm `setAdminPassword.newPassword`** — дефолт **`AdminAdmin_12$`** (публичный values). `currentPassword` дефолт **`admin`**.

**Edition Developer / Enterprise / Data Center** — разные машины отказа, не «тариф UI». Несколько CE workers — EE+. Кластер app+search — только DCE. SAML SSO — коммерческие. SCIM — Enterprise+.

**Reverse proxy / LB** — перед 9000 в DCE обязательно. HTTP, **без** sticky, **с** health check. Встроенный ingress-nginx чарта — testing only, зависимость deprecated.

**IdP** — Keycloak / Entra / GitLab / LDAP. Без IdP в проде останетесь с локальным `admin`.

**Java** — для ZIP: JDK **21 или 25**. Образы Docker/Helm несут свою JVM. PKCS#12 для ES в доке 2026.1 с пометкой *readable by Java 17* — проверять на JVM образа.

**PostgreSQL** — прод: **14–18**, charset **UTF-8**, отдельная машина/кластер, низкая задержка до нод SQ. HA БД — Patroni / ваша схема, не фича SonarQube. Oracle/SQL Server — реже.

**initSysctl** — privileged init-контейнер чарта ставит `vm.max_map_count`. На тесте часто `true`. В проде выключают, sysctl делает администратор нод. `initFs` тоже выкл в restricted namespace.

**HPA** — автомасштаб application pods в DCE (чарт `sonarqube-dce`). Search nodes HPA **не** масштабирует. `minReplicas: 2`, `maxReplicas: 10`, по умолчанию выключен. На длинный апгрейд HPA выключить.

**Rolling upgrade кластера DCE** — *Cluster downtime is required for upgrades or plugin installations*. Это не rolling Kafka. Плагины **не шарятся**: ставить на **все** application nodes, при установке **все** app nodes стопают.

**ES authentication** — `sonar.cluster.search.password` / Helm `searchAuthentication`. Иначе 9001/9002 — открытый поисковый кластер во внутренней сети.

**FIPS** — SQ на RHEL с ограничениями. ES authentication в DCE на FIPS **не работает** (PEM пока нет). SAML с подписью/шифрованием assertion — ещё нет. Webhook secret в FIPS ≥ 16 символов.

**`sonar.secretKeyPath`** — файл ключа для шифрования паролей в конфиге.

**`monitoringPasscode`** — пароль monitoring в Helm. Не из примера.

**ArgoCD** — *Not currently fully supported or validated*. GitOps — риск, не гайд вендора.

**Azure Fileshare PVC / Fargate / App Runner / Azure App Service** — не поддерживаются (ES prerequisites / известный слом NTFS).

**topologySpreadConstraints** — разнести search по `topology.kubernetes.io/zone`, `whenUnsatisfiable: DoNotSchedule`, `maxSkew: 1` (пример README чарта). Иначе три PVC search в одном ЦОДе.

**PDB** — выкат не должен снять сразу 2 из 3 search (кворум).

**Image tag `latest`** — не брать. Прод: pin **2026.1.5** (digest).

**Falco / Wazuh / WAF** — рантайм и хост. SonarQube смотрит дерево исходников в пайплайне, не syslog Kafka.

**SoT** — SonarQube не озеро клиентов. Проекты = репозитории микросервисов/Helm/SQL.

Источники формулировок: глоссарий и тело `SonarQube.md`. Новых порогов RTT и размеров диска здесь нет.
