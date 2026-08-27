# GitLab CI/CD 19.3.0 — термины и сокращения

Словарь к файлу `GitLab CI.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа. GitLab CI — не один бинарь: координатор (Rails/Puma, Sidekiq, Workhorse, Gitaly) и отдельно GitLab Runner.

**Файл / каталог на диске** — репозитории живут на диске **Gitaly**, не в поде Webservice. Артефакты и логи в проде — в object storage, не на диске пода GitLab.

**TCP-порт** — номер в сети. Дефолты Linux package: **443/80** (HTTPS/HTTP UI и API, сюда же Runner); **22** (Git SSH); **8075** (Gitaly, TLS **9999**); **2305** (Praefect, TLS **3305**); **5432** (PostgreSQL); **6379** (Redis), Sentinel **26379**; **5050/5000** (Container Registry); **8150** (KAS); **8080/8181** (внутренние Puma/Workhorse, снаружи обычно не торчат). Runner с kubernetes-executor ходит к API GitLab (**443**) и к API Kubernetes (обычно **6443**).

**ЦОД** — отдельная площадка. Один GitLab environment **между регионами** вендор не поддерживает. Несколько своих ЦОДов: *synchronous capable latency*, избыточные линки, **один geographic region**, **нечётное** число площадок под кворум. Support multi-DC: *generally at your own risk*.

**RTT** — время туда-обратно. Для HA вендор: сеть **< 5 ms**. Praefect: *< 1 с, желательно единицы миллисекунд*, официально **одна площадка**. Пока RTT не измерен, stretch Praefect/Patroni/Redis на 3 ЦОДа — гипотеза.

**TLS** — шифрование канала. Штатный TLS — не ГОСТ/СКЗИ.

**CVE / security-патч** — критический патч GraphQL 17 августа 2026 закрыт в **19.2.4 / 19.1.6 / 19.0.8 / 18.11.11**. 19.3.0 вышел 20 августа. Патчи 19.3.x ставить по мере выхода, не замирать на `.0`.

**Self-managed** — GitLab у вас, не GitLab.com и не Dedicated.

**GitLab CI/CD** — встроенный конвейер: файл `.gitlab-ci.yml` описывает, *что* собрать/протестировать/выкатить. Отдельного продукта «поставить только CI» нет. Без инстанса GitLab Runner не к чему регистрироваться.

**Координатор** — сам GitLab: принимает push/MR, создаёт pipeline, отдаёт джобы Runner'ам, принимает логи и артефакты.

**GitLab Runner** — программа, которая **спрашивает** у GitLab «есть работа?» и **исполняет** джоб. Без Runner пайплайны висят в `pending`. Версия агента в файле: **19.3.0** (тот же день, что GitLab 19.3). Runner не старше GitLab более чем на одну major; целевой пин — та же minor 19.3.

**Runner manager** — процесс Runner, который ходит в API GitLab и создаёт поды джобов. Сам код приложения обычно **не** собирает — это делают эфемерные поды.

**Executor** — как Runner запускает джоб: `kubernetes` (под на джоб), `docker`, `shell`, … Для вашего Kubernetes целевой — **kubernetes**.

**Shell executor** — джоб на общей машине как скрипт ОС. Джобы видят друг друга. Для прода микросервисов не ваш путь.

**Pipeline** — один прогон CI по событию (push, MR, tag, schedule): набор джобов, связанных стадиями/правилами.

**Job (джоб)** — одна единица работы (`build`, `test`, `deploy`). Один джоб = как правило один под Kubernetes (при kubernetes-executor).

**Stage** — группа джобов, которые логически идут пачкой (`build` → `test` → `deploy`).

**`.gitlab-ci.yml`** — YAML в корне (или include) репозитория. Это **код**, его ревьюят как код.

**MR (merge request)** — запрос на слияние ветки. Пайплайн MR — отдельный прогон.

**Shared / group / project runner** — общий на инстанс / на группу / только на проект. Shared на всём инстансе удобен и опасен: любой проект может поставить джоб на ваши ноды.

**Tag (тег Runner)** — ярлык (`k8s`, `docker-build`, `prod-deploy`). Джоб с `tags:` попадёт только на Runner с этими тегами.

**Protected runner / protected-ветки** — Runner берёт джобы **только** с protected-веток/тегов. Нужен, чтобы прод-секреты не утекли из MR с форка.

**Authentication token (`glrt-`)** — современный токен регистрации Runner. Хранить в Secret.

**Registration token (legacy)** — старый токен: кто украл, регистрирует **любой** Runner. Deprecated, снятие запланировано на **GitLab 20.0**.

**CI/CD variable** — переменная для джобов. **Masked** — не светить в логе; **Protected** — только protected-ветки. Не путать с секретом в Vault. Masked ≠ недоступен Developer'у в UI.

**`CI_JOB_TOKEN`** — короткоживущий токен джоба: клонировать репо, пушить в Registry, дергать API в рамках прав. Утечка = доступ от имени того, кто запустил пайплайн, пока джоб жив. Включать **ограничение scope** (allowlist проектов).

**Artifact** — файлы джоба, которые GitLab сохраняет (отчёт, бинарь, покрытие). На проде это **object storage**. Retention: `expire_in` в YAML + админские TTL. Worker `Ci::ScheduleBulkDeleteJobArtifactCronWorker` раз в **30 мин** чистит expired; если TTL не задали, бакет растёт.

**Cache** — зависимости между джобами (npm/maven). На нескольких Runner без **distributed cache** (S3) кэш «прилипает» к одному менеджеру.

**Job log / incremental logging** — лог сборки. В мультинодовом GitLab без NFS логи надо вести через incremental logging (куски в Redis → object storage). Иначе лог может потеряться, если его принял один Rails, а Sidekiq живёт на другом.

**Sidekiq** — фоновые задачи GitLab: архивация логов, пайплайны, почта, Geo. Очередь CI без живого Sidekiq деградирует. При нагрузке — отдельные queues под CI.

**Webservice (Puma/Rails)** — HTTP API и UI. Runner ходит сюда за джобами и шлёт трейс лога. Минимум **2** пода в разных зонах. Алгоритм LB: **least connections**, не round-robin.

**Workhorse** — прокси перед Rails: git HTTP, загрузка артефактов, крупные тела. Клиенты снаружи обычно видят его, не «голый» Puma.

**Gitaly** — хранилище Git-репозиториев. Джоб **клонирует** код отсюда (через GitLab). Упал Gitaly — CI не из чего собирать. Диск не burstable «по кредитам»; NFS как диск репозиториев в проде — путь страдания.

**Gitaly Cluster (Praefect)** — несколько Gitaly + маршрутизатор Praefect: копии репозитория, автоfailover. Фича есть в Free; техподдержка Praefect у GitLab Support — Premium+. Latency: **< 1 с, желательно единицы миллисекунд**, официально **одна площадка**. Для HA git в проде вендор указывает Cloud Native Hybrid (Praefect на VM). Praefect в Kubernetes — **beta / not GA**.

**Sharded Gitaly** — не кластер: каждый репозиторий живёт на **одном** Gitaly. Потеря этой ноды/пода = эти репо недоступны. В Cloud Native на Kubernetes вендор кладёт именно шарды. GA с 18.11; рестарт StatefulSet = простой **этих** репо.

**replication factor 3 (Praefect)** — три копии репозитория. Падение 1 Gitaly из 3: автоfailover. RPO вендора для одного узла: *less than 1 minute* (асинхронная реплика записи; strong consistency уменьшает потери в части сценариев); RTO *less than 10 seconds* при 10 неудачных health check. Snapshot одной ноды Praefect — нельзя. Бэкап — официальные Rake.

**Гонка `PostReceiveHook` vs реплика** — известный баг: CI не находит `refs/merge-requests/...`. Лечение вендора: retry джоба / fetch stage (`job stages attempts`).

**Geo** — вторая (и далее) площадка GitLab: реплика «на чтение» + ручной failover. **Premium/Ultimate**, не Free. Для нескольких **регионов**, не замена Gitaly Cluster. Задержка *up to one minute*, консистентность eventual, покрывает **весь** инстанс. Active/Active на 3 ЦОДа нет.

**Object storage** — S3-совместимое хранилище артефактов, LFS, uploads, packages, часто Registry. Для распределённого GitLab **обязателен**. Consolidated: один connection, разные bucket'ы. Класс отказа бакета ≥ класс отказа GitLab.

**LFS (Git Large File Storage)** — большие файлы рядом с git, обычно в object storage.

**Cloud Native** — GitLab-компоненты в Kubernetes; PostgreSQL, Redis, object storage — **снаружи**. Gitaly в K8s — шарды, Praefect в K8s **beta**. Цитата чарта: дефолтный Helm **не для production**.

**Cloud Native Hybrid** — stateless (Webservice, Sidekiq) в Kubernetes; Gitaly Cluster / тяжёлое состояние — на VM или внешней службе. Целевой прод координатора в файле.

**Omnibus (Linux package)** — пакет `gitlab-ee`/`gitlab-ce` на VM. Удобен для стенда и для «железа» Gitaly/Praefect.

**Helm `gitlab/gitlab` 10.3.0** — чарт инстанса → GitLab **19.3.0**. Чарт Runner **`gitlab/gitlab-runner`**: версия чарта **не равна** версии Runner. На дату документа тег `v0.91.0` (16 июля 2026, линия 19.2). Пинить по колонке **APP VERSION = 19.3.0**.

**Встроенный `gitlab-runner` чарта** — `gitlab-runner.install` default **true**, `concurrent` дефолт **20**. На проде либо жёстко конфигурируют, либо выключают и ставят отдельный чарт.

**`concurrent`** — сколько джобов один Runner manager готов вести сразу. Дефолт 20 — не ёмкость кластера. Считать от реальных CPU job-подов.

**`clusterWideAccess`** — широкий RBAC Runner к кластеру. Без нужды не включать.

**RPS** — запросов в секунду к GitLab. Вендор **сайзит координатор по RPS**, не по «терабайтам озера». Ориентиры Cloud Native (не ваша смета): S ≤100, M ≤200, L ≤500, XL ≤1000. Для S вендор рисует Webservice **6–9** подов, Sidekiq **8–12**, Gitaly **3** шарда без автоскейла. Linux-package HA «с завода» от **~60 RPS / 3000 users**.

**Docker-in-Docker (DinD)** — Docker **внутри** джоба. На kubernetes-executor почти всегда нужен **`privileged = true`**. Вендор: это снимает изоляцию контейнера. На общем пуле в проде не делать.

**BuildKit rootless** — сборка образа без privileged DinD (`moby/buildkit:rootless`). В доке GitLab — замена Kaniko. На Kubernetes + AppArmor может понадобиться annotation `unconfined` на **этот** контейнер — дырка меньше, чем privileged DinD.

**Kaniko** — старый способ собирать образ в поде без Docker socket. Репозиторий Google **архивирован 3 июня 2025**. В новый прод не закладывать.

**KAS / agentk** — GitLab Agent for Kubernetes: канал GitLab↔кластер (GitOps, cluster access). Порт KAS по умолчанию **8150**. Это **не** Runner и не обязательно для «просто CI».

**Container Registry** — реестр образов GitLab (порты 5050/5000, свой бакет). Не анонимный push.

**RPO / RTO** — сколько данных готовы потерять / как быстро поднять CI снова. Без Premium честный DR координатора = бэкап + restore (RTO часами, не «переключили DNS»). Цель «пережить 2 из 3» не обещать.

**PostgreSQL** — метаданные проектов, пайплайнов, джобов. GitLab БД не кластеризует. HA writer — ваша (Patroni / CloudNativePG). Версию смотреть в installation requirements 19.3. Реплика read-only GitLab как живой primary **не** заменяет штатный failover БД. Для HA БД Praefect вендор просит third-party PostgreSQL.

**Redis / Valkey** — сессии, Sidekiq, куски incremental logs. **Standalone + Sentinel (HA)**. **Redis Cluster не поддерживается** (в т.ч. incremental logging, issue 224171). Serverless вендор не поддерживает. Два инстанса в референсах Cloud Native: cache (можно eviction) и persistent (eviction нельзя «как кэш»).

**Sentinel** — процесс выбора Redis primary. Порт **26379**.

**Free / CE vs Premium vs Ultimate** — ядро CI (координатор + Runner) живёт в Free. Geo — платная редакция. Merge trains и часть «больших» CI-фич — нет / урезанно в Free.

**HPA** — больше подов Webservice по метрикам. Не лечит медленный clone и полный диск бакета.

**PDB** — лимит одновременного снятия подов Webservice/Sidekiq при drain.

**topologySpreadConstraints / `topology.kubernetes.io/zone`** — размазать реплики по ЦОДам.

**NetworkPolicy / PSA Restricted** — firewall подов и фильтр privileged. Job-namespace: default-deny + allow GitLab, Registry, Vault, Sonar, нужные API. Privileged пул — отдельный namespace и ноды, не ноды Kafka/Camunda.

**ResourceQuota** — потолок CPU/RAM/подов в namespace джобов.

**IdP / 2FA** — вход людей; закрыть публичную регистрацию; root не «из установки».

**SoT** — источник истины кода. Три полноценных GitLab = три SoT. Обычно хуже.

**Quality Gate / SonarScanner / SonarQube** — проверка качества **в джобе**, не внутри GitLab. Отдельный документ.

**Vault** — короткие credentials в джобах, не вечный секрет в UI.

**shallow clone** — неполный clone репозитория. Вендор: сотни concurrent CI jobs for large repositories насыщают канал; лечится shallow clone, LFS, не «ещё один брокер Kafka».

**Swap** — в референс-архитектурах **не рекомендуется**.

**Zero-downtime upgrade Cloud Native Hybrid** — *not supported*. Планировать окно.

**Вариант A** — один Kubernetes на 3 зоны, один Hybrid-инстанс. Только после замера RTT стабильно **< 5 ms**.

**Вариант B1** — primary Hybrid + Geo secondary (Premium). Failover ручной. Unified URL закладывать сразу.

**Вариант B2 (Free)** — один Hybrid в «домашнем» ЦОДе. Падение домашнего ЦОДа = нет CI, пока restore.

**Вариант B3** — три полноценных GitLab. Только если готовы три SoT кода.

**pending** — джоб создан, Runner не взял. Все Runner мертвы → пайплайны в pending; код в Git целый.

**`securityContext.privileged: false`** — дефолт чарта менеджера. `runners.kubernetes.privileged = true` — только осознанный пул.

**ГОСТ / СКЗИ / 152-ФЗ** — не заявлено для vanilla TLS. Job log и cache легко уносят токены и куски ПДн из фикстур.

Источники формулировок: глоссарий и тело `GitLab CI.md`. Новых порогов RTT и размеров диска здесь нет.
