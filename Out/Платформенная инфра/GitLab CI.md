# GitLab CI/CD 19.3.0 — развёртывание и настройка

Версия координатора: **GitLab 19.3.0** (minor-релиз 20 августа 2026).  
Версия агента сборки: **GitLab Runner 19.3.0** (выходит в тот же день, что и GitLab 19.3).  
Helm GitLab (сам инстанс на Kubernetes): чарт **`gitlab/gitlab` 10.3.0** → GitLab **19.3.0** (таблица version mappings вендора).  
Helm Runner: чарт **`gitlab/gitlab-runner`**, версия чарта **не равна** версии Runner. На дату документа в репозитории чарта виден тег `v0.91.0` (16 июля 2026, линия 19.2). Чарт под Runner **19.3.0** нужно снять командой `helm search repo -l gitlab/gitlab-runner` и пинить по колонке **APP VERSION = 19.3.0**, не по «последнему тегу, который запомнили».

Документация CI: https://docs.gitlab.com/ci/  
Runner: https://docs.gitlab.com/runner/  
Референс-архитектуры: https://docs.gitlab.com/administration/reference_architectures/  
Чарт GitLab: https://docs.gitlab.com/charts/

Политика поддержки на дату документа: **19.3** — bug + security; **19.2** и **19.1** — security. Критический патч GraphQL 17 августа 2026 закрыт в **19.2.4 / 19.1.6 / 19.0.8 / 18.11.11**; 19.3.0 вышел позже (20 августа). Патчи 19.3.x ставить по мере выхода, не замирать на `.0`, если появился security-патч.

Этот текст — не мануал «скопируй `.gitlab-ci.yml`», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

GitLab CI **не было** в исходном описании архитектуры (Kafka, Camunda, озеро данных, интеграционное API). Ниже — как поставить конвейер поставки на ту же инфраструктуру с Kubernetes и тремя ЦОДами. Это не шина событий, не процессный движок и не «кнопка DevOps включена».

**Главное, что путают с названием «GitLab CI»:** отдельного продукта «поставить только CI» нет. CI/CD — подсистема GitLab. Без инстанса GitLab (репозитории, очередь джобов, артефакты, API) Runner не к чему регистрироваться. Документ покрывает **два контура**, которые вместе и есть GitLab CI: координатор (GitLab) и исполнители (Runner).

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **GitLab CI/CD** | Встроенный конвейер: файл `.gitlab-ci.yml` в репозитории описывает, *что* собрать/протестировать/выкатить. Это не отдельный сервер рядом с GitLab. |
| **Координатор** | Сам GitLab: принимает push/MR, создаёт pipeline, отдаёт джобы Runner'ам, принимает логи и артефакты. |
| **GitLab Runner** | Программа, которая **спрашивает** у GitLab «есть работа?» и **исполняет** джоб. Без Runner пайплайны висят в `pending`. |
| **Pipeline** | Один прогон CI по событию (push, MR, tag, schedule): набор джобов, связанных стадиями/правилами. |
| **Job (джоб)** | Одна единица работы (`build`, `test`, `deploy`). Один джоб = как правило один под Kubernetes (при kubernetes-executor). |
| **Stage** | Группа джобов, которые логически идут пачкой (`build` → `test` → `deploy`). |
| **Executor** | Как Runner запускает джоб: `kubernetes` (под на джоб), `docker`, `shell`, … Для вашего Kubernetes целевой — **kubernetes**. |
| **Runner manager** | Процесс Runner, который ходит в API GitLab и создаёт поды джобов. Сам код приложения он обычно **не** собирает — это делают эфемерные поды. |
| **Shared / group / project runner** | Общий на инстанс / на группу / только на проект. Shared на всём инстансе удобен и опасен: любой проект может поставить джоб на ваши ноды. |
| **Tag (тег Runner)** | Ярлык (`k8s`, `docker-build`, `prod-deploy`). Джоб с `tags:` попадёт только на Runner с этими тегами. Без тегов легко «уедет» на чужой Runner. |
| **Protected runner** | Runner, который берёт джобы **только** с protected-веток/тегов. Нужен, чтобы прод-секреты не утекли из MR с форка. |
| **Authentication token (`glrt-`)** | Современный токен регистрации Runner. Legacy **registration token** deprecated, снятие запланировано на **GitLab 20.0**. |
| **`.gitlab-ci.yml`** | YAML в корне (или include) репозитория: что запускать. Это **код**, его ревьюят как код. |
| **CI/CD variable** | Переменная для джобов (URL, флаги). **Masked** — не светить в логе; **Protected** — только protected-ветки. Не путать с секретом в Vault. |
| **`CI_JOB_TOKEN`** | Короткоживущий токен джоба: клонировать репо, пушить в Registry, дергать API в рамках прав. Утечка = доступ от имени того, кто запустил пайплайн, пока джоб жив. |
| **Artifact** | Файлы джоба, которые GitLab сохраняет (отчёт, бинарь, покрытие). На проде это **object storage**, не диск пода GitLab. |
| **Cache** | Зависимости между джобами (npm/maven). На нескольких Runner без **distributed cache** (S3) кэш «прилипает» к одному менеджеру и не шарится. |
| **Job log / incremental logging** | Лог сборки. В мультинодовом GitLab без NFS логи надо вести через **incremental logging** (куски в Redis → object storage). Иначе лог может потеряться, если его принял один Rails, а Sidekiq живёт на другом. |
| **Sidekiq** | Фоновые задачи GitLab: архивация логов, пайплайны, почта, Geo. Очередь CI без живого Sidekiq деградирует. |
| **Webservice (Puma/Rails)** | HTTP API и UI. Runner ходит сюда за джобами и шлёт трейс лога. |
| **Workhorse** | Прокси перед Rails: git HTTP, загрузка артефактов, крупные тела. Клиенты снаружи обычно видят его, не «голый» Puma. |
| **Gitaly** | Хранилище Git-репозиториев. Джоб **клонирует** код отсюда (через GitLab). Упал Gitaly — CI не из чего собирать. |
| **Gitaly Cluster (Praefect)** | Несколько Gitaly + маршрутизатор Praefect: копии репозитория, автоfailover. Latency: **< 1 с, желательно единицы миллисекунд**, официально **одна площадка**. |
| **Sharded Gitaly** | Не кластер: каждый репозиторий живёт на **одном** Gitaly. Потеря этой ноды/пода = эти репо недоступны. В Cloud Native на Kubernetes вендор кладёт именно шарды, не Praefect. |
| **Geo** | Вторая (и далее) площадка GitLab: реплика «на чтение» + ручной failover. Это **Premium/Ultimate**, не Free. Для нескольких **регионов**, не замена Gitaly Cluster. |
| **Object storage** | S3-совместимое хранилище артефактов, LFS, uploads, packages, часто Registry. Для распределённого GitLab **обязателен**. |
| **Cloud Native** | Референс: GitLab-компоненты в Kubernetes; PostgreSQL, Redis, object storage — **снаружи**. Gitaly в K8s — шарды, Praefect в K8s **beta**. |
| **Cloud Native Hybrid** | Stateless (Webservice, Sidekiq) в Kubernetes; Gitaly Cluster / тяжёлое состояние — на VM или внешней службе. Для HA git в проде вендор указывает сюда. |
| **Omnibus (Linux package)** | Один (или несколько) пакет `gitlab-ee`/`gitlab-ce` на VM. Удобен для стенда и для «железа» Gitaly/Praefect. |
| **RPS** | Запросов в секунду к GitLab. Вендор **сайзит координатор по RPS**, не по «терабайтам озера». |
| **`concurrent`** | Сколько джобов один Runner manager готов вести сразу. Дефолт чарта GitLab для встроенного runner: **20**. Это не ёмкость кластера. |
| **Docker-in-Docker (DinD)** | Собирать образ, подняв Docker **внутри** джоба. На kubernetes-executor почти всегда нужен **`privileged = true`**. Вендор прямо: это снимает изоляцию контейнера. |
| **BuildKit rootless** | Сборка образа без privileged DinD (`moby/buildkit:rootless`). В доке GitLab — замена Kaniko. |
| **Kaniko** | Старый способ собирать образ в поде без Docker socket. Репозиторий Google **архивирован 3 июня 2025**, проект не развивается. В новый прод его не закладывать. |
| **KAS / agentk** | GitLab Agent for Kubernetes: канал GitLab↔кластер (GitOps, cluster access). Порт KAS по умолчанию **8150**. Это **не** Runner и не обязательно для «просто CI». |
| **RPO / RTO** | Сколько данных готовы потерять / как быстро поднять CI снова. |

---

## Основные элементы системы и зависимости

### Что входит в «одно ПО» GitLab CI (и что — нет)

| Роль | Зачем | Как масштабируется |
|---|---|---|
| **Webservice / Workhorse** | UI, API, приём логов Runner, git HTTP | Горизонтально: больше подов. HPA в Cloud Native |
| **Sidekiq** | Очереди: архив логов, пайплайны, крючки | Больше worker'ов; отдельные queues под CI при нагрузке |
| **Gitaly** | Git, из которого джоб клонирует код | Вертикаль диска/CPU; HA git = **Praefect на VM**, не «ещё один Deployment» |
| **PostgreSQL** | Метаданные проектов, пайплайнов, джобов | Своя HA (Patroni / управляемый сервис). GitLab БД не кластеризует |
| **Redis / Valkey** | Сессии, Sidekiq, куски incremental logs | **Standalone + Sentinel (HA)**. **Redis Cluster не поддерживается** |
| **Object storage** | Артефакты, логи после архива, LFS, пакеты, Registry | Ёмкость и HA бакета, не подов GitLab |
| **GitLab Runner manager** | Забрать джоб, создать под, вернуть статус | Несколько менеджеров; `concurrent`; отдельные пулы по тегам |
| **Job pod** | Исполнение `.gitlab-ci.yml` | Горизонтально: ноды Kubernetes + лимиты namespace |

Плюс то, что **вы обязаны поставить сами** (в проде чарт это не заменяет):

- внешние **PostgreSQL** и **Redis/Valkey**;
- **S3-совместимый** object storage (MinIO/Ceph RGW/облако) с HA, переживает тот же отказ ЦОДа, что вы обещаете CI;
- Ingress/LB **свой** перед HTTPS;
- PKI / корпоративный CA на Runner'ах (иначе clone и Registry не доверят сертификату);
- образы сборки в **вашем** registry (не «всегда тянуть с Docker Hub в госконтур»).

Встроенные в Helm GitLab **PostgreSQL / Redis / MinIO** — для PoC. Цитата чарта: *The default Helm chart configuration is not intended for production.*

### Редакции — это не «тариф UI»

| Нужно | Free / CE | Premium | Ultimate |
|---|---|---|---|
| Сам CI/CD, Runner, kubernetes-executor, артефакты | да | да | да |
| Gitaly Cluster (Praefect) как **фича** | да (tier Free) | да | да |
| Техподдержка Praefect у GitLab Support | ограничена | да | да |
| **Geo** (вторая площадка, DR инстанса) | **нет** | да | да |
| Merge trains, часть «больших» CI-фич | нет / урезанно | да | да |

**Следствие для «3 ЦОДа, без сбоев»:** ядро CI (координатор + Runner) живёт в Free. **Межплощадочный DR инстанса штатно — это Geo, и это платная редакция.** Без Premium честный DR координатора = бэкап + restore (RTO часами, не «переключили DNS»).

### Официальные порты (менять можно, это контракт сети)

| Порт (default Linux package) | Назначение |
|---|---|
| **443 / 80** | HTTPS/HTTP UI и API (сюда же Runner) |
| **22** | Git SSH |
| **8075** (TLS **9999**) | Gitaly |
| **2305** (TLS **3305**) | Praefect |
| **5432** | PostgreSQL |
| **6379** | Redis; Sentinel **26379** |
| **5050 / 5000** | Container Registry (если включён) |
| **8150** | KAS |
| **8080 / 8181** | внутренние Puma / Workhorse в пакете (снаружи обычно не торчат) |

Runner с kubernetes-executor ходит: **к API GitLab (443)** и **к API Kubernetes (обычно 6443)**. Джоб-под ходит: clone (443/22), object storage, Registry, ваши сервисы по NetworkPolicy.

### Чего в GitLab CI нет (частая путаница)

| Нужно системе | Это не GitLab CI | Зачем помнить |
|---|---|---|
| Шина бизнес-событий, гарантии Kafka | Apache Kafka | Пайплайн — про **поставку кода**, не про runtime заявок |
| Исполнение BPMN | Camunda | YAML CI ≠ процессный движок |
| Озеро эталона / SoT клиентских данных | ваша БД / объектное хранилище озера | Артефакт сборки — не эталон клиента |
| Интеграции с госслужбами | ваше интеграционное API | CI может **собрать** этот сервис, не ходить в ведомства |
| HA git внутри Kubernetes «как Praefect» | Praefect на VM; в K8s Praefect **beta** | Cloud Native: каждый Gitaly-под — SPOF **своих** репозиториев |
| Active/Active GitLab на 3 ЦОДа | нет такого режима | Geo = primary + secondary, failover **ручной** |
| Порог RTT «городской = можно stretch» | нет | Для HA вендор просит **< 5 ms**; Praefect — *ideally single-digit milliseconds*, *single location* |
| Redis Cluster «потому что HA» | **не поддерживается** (в т.ч. incremental logging) | Sentinel / managed standalone HA |
| Zero-downtime upgrade Cloud Native Hybrid | *not supported* | Планируйте окно |
| Сборка образов без изоляционных дыр «из коробки» | DinD + privileged | Нужен отдельный путь (BuildKit rootless и т.п.) |
| ГОСТ TLS / СКЗИ | не заявлено | Штатный TLS — не криптография КИИ |
| Гарантия «терабайты артефактов влезут» | нет | Это бакет + retention политики, не «поставили GitLab» |
| Замена бэкапа репликами Gitaly | нет | Praefect спасает от падения **ноды**. Не спасает от `git push --force` по ошибке и от гибели всех копий + бакета |

### Зависимости окружения (обязательны)

- **ОС координатора / Gitaly:** Linux x86_64 (прод). Windows-host для Gitaly в вашей схеме с Kubernetes не предполагается.
- **PostgreSQL:** внешний в проде; версию смотреть в *installation requirements* выбранного 19.3, не из памяти «всегда 14». HA writer — ваша.
- **Redis или Valkey:** standalone (+ Sentinel). Serverless-варианты вендор **не** поддерживает. Eviction policy: cache и persistent — **разные** инстансы в референсах Cloud Native.
- **Диск Gitaly:** не burstable «по кредитам»; требования к диску Gitaly отдельные от PVC веб-подов. NFS как диск репозиториев в проде — путь страдания; object storage — для артефактов, не замена git.
- **Сеть HA:** *Network latency should be as low as possible … Generally this should be lower than 5 ms.* Несколько своих ЦОДов: *synchronous capable latency*, избыточные линки против split-brain, **один geographic region**, **нечётное** число площадок под кворум. Цитата: *it is not supported to deploy a single GitLab environment across different regions.* Support по инфраструктуре multi-DC: *generally at your own risk*.
- **Kubernetes:** дистрибутив, удовлетворяющий prerequisites чарта. Поведение CSI/CNI — **вне** поддержки GitLab.
- **Swap:** в референс-архитектурах **не рекомендуется**.

### Как GitLab CI стыкуется с вашей архитектурой

```
Разработчик / MR
        │  git push (код микросервисов, Camunda-as-code, Helm, интеграционное API)
        ▼
   GitLab (Webservice)  ──► PostgreSQL + Redis
        │                      Gitaly (git)
        │                      Object storage (artifacts, logs, packages)
        │  джобы
        ▼
   GitLab Runner manager(s)  ──► Kubernetes API ──► Job pods
        │                                              │
        │                                              ├─ clone с GitLab
        │                                              ├─ test / SAST (SonarScanner — ваш отдельный документ)
        │                                              ├─ build image (BuildKit rootless → Registry)
        │                                              └─ deploy (Helm/GitOps) в кластеры приложений
        ▼
   Кластер приложений: Kafka, Camunda, озеро, интеграционное API
        (рантайм 24/7; CI его собирает, но не заменяет)
```

Честные роли в *вашей* картине:

1. **Поставка** микросервисов, манифестов Camunda, Helm интеграционного API.
2. **Качество до выкладки** — тесты + Quality Gate (SonarQube живёт **в джобе**, не внутри GitLab).
3. **Сборка образов** в закрытый Registry, без privileged на общих нодах приложений.

Нечестная роль: «CI переживёт два ЦОДа, потому что Runner в Kubernetes». Runner без живого GitLab/Gitaly/бакета — очередь мёртвых `pending`.

---

## Краткие вводные

### Зачем вам GitLab CI в этой архитектуре

У вас много сервисов, 30+ интеграций, процессный движок, Kubernetes, ожидание непрерывной работы **бизнеса**. Без конвейера выкладка — ручной риск: «на стенде собралось, в прод скопировали образ».

GitLab CI закрывает: единый YAML как код, MR-пайплайны, артефакты, Registry, крючок на Quality Gate. Он **не** держит runtime Kafka. Падение GitLab **не должно** ронять уже работающие сервисы. Падение GitLab **должно** быть заложено в процесс: либо поставка встаёт, либо (хуже) люди катят в обход.

### Как устроена отказоустойчивость (идея, не магия)

Три разных контура. Путать их — главная ошибка.

**1) Исполнение джобов (Runner)**

| Что падает | Что происходит |
|---|---|
| Один job-под | Этот джоб красный; retry; остальные Runner'ы живы |
| Один Runner manager из нескольких | `concurrent` этого менеджера пропал; другие менеджеры продолжают poll API |
| Весь ЦОД с job-подами | Джобы **в полёте на этой площадке** погибли. Новые джобы могут взять менеджеры в живых ЦОДах, **если** координатор жив |
| Все Runner'ы | Пайплайны в `pending`. Код в Git целый |

Эфемерный kubernetes-executor здесь сильное место: нет «грязной VM, на которой вчерашний джоб оставил malware». Слабое: без resource requests джобы съедят ноды приложений.

**2) Координатор (GitLab)**

| Что падает | Что происходит |
|---|---|
| Один под Webservice из нескольких | LB уводит трафик; Runner переподключается |
| Sidekiq | UI может быть жив, архив логов/часть пайплайнов — нет |
| PostgreSQL writer | Инстанс «глухой». Реплика read-only GitLab как живой primary **не** заменяет штатный failover БД |
| Redis | Сессии, Sidekiq, куски incremental log. Это не кэш «можно прогреть» |
| Object storage | Нет устойчивых артефактов/логов в мультиноде; Registry/LFS |
| Один sharded Gitaly | Репозитории **этого** шарда недоступны; CI этих проектов не клонирует |
| Praefect: 1 Gitaly из 3 (replication factor 3, одна площадка) | Автоfailover; RPO вендора для одного узла: *less than 1 minute* (асинхронная реплика записи; strong consistency уменьшает потери в части сценариев); RTO *less than 10 seconds* при 10 неудачных health check |
| Весь кластер Gitaly / ЦОД, где лежали все копии | Git нет. Нужен бэкап (официальные Rake), не snapshot одной ноды Praefect |

**3) Площадки (ваши 3 ЦОДа)**

Gitaly Cluster официально: **несколько нод, одна location**, задержка *< 1 s, ideally ms*. Geo: **несколько location**, задержка *up to one minute*, failover **ручной**, консистентность eventual, покрывает **весь** инстанс.

Следствие: «размазать один Praefect на 3 ЦОДа с неизвестным RTT» **не** следует из гайда. «Поднять Geo на второй ЦОД» — штатный DR, но это **Premium** и не Active/Active.

### Как устроено масштабирование

Несколько независимых осей:

1. **Больше параллельных пайплайнов** → больше **Runner managers** + `concurrent` + **ноды Kubernetes** под job-поды + Quota. Это главная ось CI. Координатор при этом может быть ещё «маленьким».
2. **Больше клонов больших репо** → Gitaly CPU/диск/сеть. Вендор отдельно предупреждает: *Hundreds of concurrent CI jobs for large repositories* — дополнительная нагрузка сверх таблиц RPS; полный clone монорепы на каждый джоб насыщает канал. Лечится shallow clone, LFS, не «ещё один брокер Kafka».
3. **Больше UI/API/MR** → Webservice (HPA), PostgreSQL, Redis cache.
4. **Больше артефактов и логов** → object storage + retention (`expire_in` в YAML + админские TTL) + Sidekiq delete workers. Терабайты озера сюда **не** попадают сами — попадают zip'ы джобов, которые вы не чистите.
5. **Sidekiq CI** → отдельные процессы/поды под тяжёлые очереди, если общий Sidekiq забит архивацией логов.

Референс Cloud Native (это **ориентир вендора**, не ваша смета; нагрузки у вас нет):

| Size | Target RPS | CI в формулировке вендора |
|---|---|---|
| S | ≤100 | Light concurrent pipeline execution; **not suitable for actively used monorepos** |
| M | ≤200 | Moderate pipeline concurrency |
| L | ≤500 | Heavy pipeline usage with proper Sidekiq scaling |
| XL | ≤1000 | Intensive |

Для S вендор уже рисует Webservice **6–9 подов**, Sidekiq **8–12**, Gitaly **3** шарда без автоскейла. Дефолт `helm install` с bundled Postgres — **не** этот размер.

Linux-package HA «с завода» вендор в целом рекомендует от **~60 RPS / 3000 users**. Меньше — часто дешевле **бэкап**, чем полный HA. У вас ожидание непрерывной поставки + 3 ЦОДа: это аргумент **за HA координатора**, даже если людей меньше 3000. Это продуктовое решение, не формула.

### Безопасность самого GitLab CI

Компрометация CI в контуре с госинтеграциями = компрометация **исходников интеграционного API, секретов деплоя, образов, карты сервисов**. Токен Runner и `CI_JOB_TOKEN` — ключи к этому.

Официально и практически:

- **Registration token** (legacy) позволяет зарегистрировать **любой** Runner, кто его украл. Переход на **`glrt-`**. Снятие legacy — план **20.0**.
- **Privileged / DinD:** *By enabling privileged mode, you are effectively disabling all the container’s security mechanisms*. Если без privileged нельзя — изолированные эфемерные VM и **protected** Runner только на protected-ветки, не общий пул MR.
- **Shell executor** на общей машине — джобы видят друг друга. Для прода микросервисов не ваш путь.
- **`CI_JOB_TOKEN`:** маскируется в логах, живёт пока джоб. Включайте **ограничение scope** (allowlist проектов), иначе токен ходит по группе шире, чем думаете.
- Переменные: секреты прода — **Protected + Masked**, лучше Vault (у вас отдельный документ) по short-lived credentials, не вечный `AWS_SECRET` в UI.
- Shared runner на всём инстансе + Developer может запустить произвольный `script:` = чужой код на ваших нодах. Для госконтура: отдельные пулы `build` / `prod-deploy`, NetworkPolicy, PSA Restricted на job-namespace (privileged пул — отдельный namespace и ноды).
- Helm GitLab по умолчанию ставит **вложенный** `gitlab-runner` (`gitlab-runner.install` default **true**). На проде его либо жёстко конфигурируют, либо выключают и ставят отдельный чарт с своими токенами — не оставляют «как в туториале».

«Открыли LoadBalancer GitLab в интернет, DinD privileged, registration token в Confluence» — это новый attack surface, не платформа поставки.

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Self-managed GitLab**, не GitLab.com и не Dedicated. Три своих ЦОДа SaaS вендора не заменяют.
2. **Прод координатора — Cloud Native Hybrid:** Webservice/Sidekiq в Kubernetes; **Gitaly Cluster (Praefect) на VM** (или эквивалент вне K8s); PostgreSQL, Redis, object storage — внешние. Это не «всё в Helm values из demo».
3. **Прод Runner — kubernetes-executor**, официальный чарт `gitlab/gitlab-runner`, менеджеры в Kubernetes. Совместимость: Runner не старше GitLab **более чем на одну major** (рекомендация вендора); целевой пин — **та же minor 19.3**.
4. **Версия — 19.3.0 + патчи линии 19.3.** Не 18.x «потому что привыкли». Не `latest` без пина.
5. **Сборка образов в проде — без privileged DinD на общем пуле.** Путь: BuildKit rootless (дока GitLab: замена Kaniko). Kaniko Google **не** использовать как стратегический инструмент (архив с июня 2025). Если DinD всё же нужен — отдельный isolated пул, не ноды Kafka/Camunda.
6. **Три ЦОДа = три зоны отказа в одном городе / одном region.** Один GitLab environment **между регионами** вендор не поддерживает.
7. **Цель отказа координатора: пережить 1 ЦОД только если** сеть и кворум PG/Redis/Praefect это позволяют **и** object storage не умер вместе с этим ЦОДом. Цель «пережить 2 из 3» **не** обещать.
8. **Пока RTT не измерен, stretch Praefect/Patroni/Redis на 3 ЦОДа — гипотеза, не план.** Если RTT ≥ 5 ms — координатор в 1–2 площадках, третья: Runner + (если Premium) Geo secondary.
9. **Geo в базовый Free-план не входит.** Если лицензии Premium нет, DR инстанса = бэкапы Gitaly/PG/object storage + прогон restore. Это слабее «без сбоев», и это надо сказать руководству явно.
10. **Нагрузки и RPS нет** — нет цифры «N webservice и M concurrent». Есть сигналы (pending jobs, очередь Sidekiq, CPU Gitaly, latency 443, saturating clone) и рычаги (Runner, ноды, Gitaly, RPS-класс).
11. **Формального SLA на CI нет.** «Система 24/7» про бизнес-контур. Политика: блокирует ли мёртвый GitLab выкладку в гособмен.
12. **Тестовый стенд изолирован.** На нём допустимы Omnibus all-in-one и bundled chart. В прод их не копируем.
13. **Kafka, Camunda, озеро, интеграционное API** — **репозитории и runtime**, не HA-зависимости GitLab. Джобы не должны иметь NetworkPolicy «весь кластер».
14. **SonarQube, Vault, Registry** — соседние платформенные сервисы. Сканер и секреты живут в джобах, не внутри процесса Runner.
15. **Zero-downtime upgrade Hybrid не используем как обещание.** Окно выката координатора закладываем.
16. **Шифрование канала — обычный TLS.** Требования ГОСТ/СКЗИ не озвучены.

---

## Критически важные условия, которых нет в исходном контексте

Их нужно закрыть **до** «helm install gitlab и забыли».

| Пробел | Почему это ломает решение |
|---|---|
| **RTT и потери между ЦОДами** (p50/p95/p99, все пары), отдельно до PostgreSQL, Redis, Gitaly **8075**, object storage | Вендор: HA sync **< 5 ms**. Praefect — ms, одна location. Без замера схема «3 ЦОДа = 3 AZ» — лотерея split-brain и медленный git clone в каждом джобе |
| **Один Kubernetes на 3 ЦОДа или три кластера** | Runner может быть в каждом кластере. Координатор и Praefect — нет. От этого зависит Hybrid vs Geo vs «CI лежит, если домашний ЦОД лёг» |
| **Где writer PostgreSQL, Redis primary, бакет артефактов** | Падение ЦОДа с writer/бакетом = падение CI, сколько ни плодь Runner'ов |
| **Редакция (Free vs Premium) и нужен ли Geo** | Без Geo нет штатного межплощадочного DR инстанса |
| **Профиль CI:** монорепа или 30 мелких репо; сколько параллельных джобов; полный clone vs fetch; размер артефактов | Таблица RPS «пользователи» врёт, если каждый MR клонирует гигабайты |
| **Где собирать образы** | Privileged на общих worker-нодах приложений = вы отдаёте root ноды любому `script:` |
| **Политика при недоступном GitLab / красном pipeline** | Не спроектировали — в инциденте либо встанет релизный поезд, либо все катят руками |
| **Куда смотрит UI/API** (только внутренняя сеть?) | Публичный 443 + слабый root + открытая регистрация Runner — инцидент |
| **152-ФЗ / секреты в логах и артефактах** | Job log и cache легко уносят токены и куски ПДн из фикстур |
| **Нужен ли Container Registry / KAS** | Это дополнительные порты, бакеты и RBAC, не «включено само» |
| **Совместимость чарта 10.3.0 с вашим Kubernetes 1.36** | Смотреть prerequisites чарта на момент установки, не этот абзац как вечную таблицу |

---

## Краткое описание ключевых этапов и элементов развертывания

Общий каркас (и тест, и прод):

1. Зафиксировать **GitLab 19.3.x** и **Runner 19.3.x**, чарт GitLab **10.3.x**, чарт Runner по APP VERSION 19.3.x. Не смешивать major.
2. Сначала **PostgreSQL + Redis + бакет**, потом GitLab, потом Runner. Обратный порядок даёт «UI есть, джобы pending / артефакты на emptyDir».
3. Выпустить **свои** пароли root, токены `glrt-`, ключи object storage, сертификаты **до** первого боевого проекта.
4. Прогнать **один** репозиторий: pipeline зелёный, лог виден после завершения джоба, артефакт скачивается, образ (если есть) в Registry.
5. Включить метрики: pending jobs, `ci_running_builds`, очередь Sidekiq, latency clone, диск Gitaly, 5xx Workhorse, saturating network.
6. Только потом — защищённые ветки, раздельные пулы Runner, Vault, блок merge по Quality Gate.

Дальше — два режима.

---

### 1 инстанс: тестовый стенд, 1 ЦОД, без нагрузки

**Цель стенда:** чтобы команда написала `.gitlab-ci.yml`, увидела джоб в UI, поняла теги Runner и kubernetes-executor. **Не** цель: доказать отказ ЦОДа и ёмкость «сотни параллельных сборок интеграционного API».

Два официальных минимальных пути (выберите один):

**Путь A — Omnibus на одной VM** (быстрее понять продукт):

- Пакет GitLab CE или EE, один хост, встроенные PostgreSQL/Redis.
- HTTPS хотя бы с внутренним CA, не оставлять HTTP, если стенд маршрутизируется дальше localhost.
- Один Runner: `docker` или `kubernetes` executor на тот же/соседний хост.
- Токен **`glrt-`**, не registration token «на будущее в прод».

**Путь B — Helm `gitlab/gitlab` 10.3.0 как PoC** — ближе к оркестратору, **опаснее привычкой**:

- Вендор прямо: дефолтный чарт — **не прод** (bundled Postgres/Redis/MinIO).
- Для стенда на неделю допустимо; если стенд живёт месяцами — вынести Postgres/Redis/бакет сразу, иначе миграция будет болью.
- Встроенный `gitlab-runner` чарта можно оставить на тесте, понимая, что в проде его пересоберёте.
- `privileged: false`. Сборку образа на тесте — BuildKit rootless или вообще «джоб echo hello», чтобы не учить DinD как норму.

Режим «Praefect на 3 VM + Geo» для знакомства с YAML **не нужен**: создаст ложное чувство HA и сгорит по людям.

#### Что сознательно упрощаем

| Параметр | На тесте | Почему можно |
|---|---|---|
| Praefect / Geo | нет | Нет требования пережить узел git |
| Внешний Redis Sentinel | часто нет | Один процесс Redis на Omnibus |
| Incremental logging | можно не трогать на одном узле | Нет второго Rails |
| HPA Webservice | выкл | Нет RPS |
| Несколько Runner managers | 1 | Нет очереди |
| PSA Restricted на job-ns | желательно, но не блокер дня 1 | Иначе команда утонет в политиках раньше YAML |
| IdP | локальные пользователи | На тесте важнее контур джоба |

#### Чего на тесте **не** стоит упрощать

- HTTPS и свой пароль `root` (не `5iveL!fe` из старых гайдов, если до стенда есть сеть).
- Регистрация Runner через **`glrt-`**.
- Хотя бы один джоб **реального** сервиса (интеграционное API или маленький Java/Helm), не только `echo`.
- Понимание: **Runner ≠ GitLab**. Убитый Runner оставляет UI зелёным и пайплайны серыми.
- `privileged: false` как дефолт. Кто включил DinD на тесте — пусть пишет, зачем, в README стенда.
- Запомнить: зелёный pipeline на одном Omnibus **не** доказывает Hybrid, Praefect и RTT между ЦОДами.

#### Сильные стороны такой схемы

- Поднимается за часы.
- Совпадает с Try GitLab / chart PoC.
- Дёшево показывает, как 30 репозиториев превратятся в 30 YAML и в очередь джобов.

#### Слабые стороны (обязательно понимать)

- Нет модели отказа ЦОДа, нет object storage HA, нет Praefect.
- Bundled MinIO/Postgres в чарте приучают «данные живут в кластере как у приложения».
- Shared runner без тегов на тесте легко уезжает в прод «как было».
- Успех `hello world` **не** доказывает clone монорепы 20 GB × N джобов.

Практическая рекомендация: препрод = маленький **прод-профиль** (внешние PG/Redis/бакет, 2 Webservice, 1 Gitaly с PVC не emptyDir, 2 Runner manager, HTTPS, `glrt-`, без privileged), даже без боевого трафика.

---

### Прод: 3 ЦОДа, нагрузка

Цифр «ядер под терабайты озера» **нет** — нагрузки и RPS нет. Ниже правила, без которых экземпляр не считается готовым.

#### Шаг 0. Макроархитектура (сделать до установки)

GitLab не выбирает за вас «один Kubernetes или три». Он плохо переживает **три независимых GitLab** (три правды репозиториев, три Registry, три пайплайна «что в проде») и плохо переживает **слепой stretch Praefect**.

**Вариант A — один Kubernetes на 3 зоны, один Hybrid-инстанс.**

Имеет смысл **только после замера**: RTT между зонами к PG/Redis/Gitaly **стабильно < 5 ms**, линки избыточны, ЦОДы в одном region.

- Webservice/Sidekiq: несколько реплик, `topologySpreadConstraints` по `topology.kubernetes.io/zone`.
- PostgreSQL: кворумный HA writer (Patroni и т.п.), нечётное размещение. Размазать «поровну» без понимания кворума БД — отдельная ошибка.
- Redis: **два** standalone HA (cache + persistent), Sentinel, **не** Redis Cluster. Incremental logging включён.
- Object storage: бакет, который переживает падение **одного** ЦОДа (репликация erasure/другой ЦОД/внешняя СХД). Бакет «только диск ЦОДа 1» = CI умрёт вместе с ним.
- **Gitaly Cluster на VM**, 3 ноды, replication factor 3, Praefect + **отдельная** БД Praefect (для HA этой БД вендор просит third-party PostgreSQL). Latency Praefect — ms. Если RTT между ЦОДами не ms — **не** класть по Gitaly в каждый ЦОД «для галочки».
- Runner: **минимум по одному manager в каждой зоне**, kubernetes-executor, job-поды с requests/limits, Quota. Падение зоны убивает только тамошние джобы.

**Вариант B — координатор не растягивать.**

Когда RTT неизвестен или ≥ 5 ms, или сеть ЦОДов не «AZ облака».

- **B1 (ближе к вендору, нужен Premium).** Primary Hybrid в ЦОДе с лучшей сетью к дискам + **Geo secondary** в другом ЦОДе. Failover ручной. Runners в трёх ЦОДах смотрят на primary URL; после failover — на новый primary (unified URL в гайде Geo+Helm — закладывать сразу).
- **B2 (Free).** Один Hybrid в «домашнем» ЦОДе. Из других ЦОДов — только Runner'ы и агенты разработки. Падение домашнего ЦОДа = **нет CI**, пока restore из бэкапа. Честный RTO. Runbook бэкапа Gitaly **официальными Rake**, не snapshot одной Praefect-ноды.
- **B3.** Три полноценных GitLab — только если готовы три SoT кода. Обычно хуже.

**Пока топология K8s и RTT не зафиксированы, нельзя честно выбрать A или B.** Дальше — общее.

##### Сильные / слабые стороны (A, один инстанс на 3 зоны)

| Сильное | Слабое |
|---|---|
| Один реестр кода, один YAML, один Registry | Требование **< 5 ms** и Praefect *single location*; Support multi-DC *at your own risk* |
| Runner в каждой зоне переживает падение площадки **исполнения** | Кворум PG/Redis/Praefect не переживает **два** ЦОДа |
| Совпадает с «odd number of AZ» | PVC/диски Gitaly зональные: ошибка размещения = ложный HA |
| HPA Webservice | HPA не лечит медленный clone и полный диск бакета |

##### Сильные / слабые стороны (B1, Geo)

| Сильное | Слабое |
|---|---|
| Штатный DR всего инстанса | **Premium**; failover **ручной**; eventual consistency |
| Praefect не обязан жить на 8075 между всеми тремя ЦОДами постоянно | Пока DNS/люди не переключили — CI на primary мёртв |
| Secondary можно читать (clone ближе к площадке) | Известные шероховатости Praefect+Geo (см. known issues вендора) |

##### Слабое место любого stretch без замера RTT

Документация задаёт **5 ms** как общий ориентир HA и **миллисекунды** для Praefect. Утверждение «3 ЦОДа в одном городе, значит можно как AZ» — **непроверенное**, пока нет ваших графиков.

#### Gitaly / Praefect (ядро отказоустойчивости *исходников для CI*)

- Для прода с «git mission critical» вендор: если деградация git мешает выкатывать в прод — git надо считать mission critical → убирать SPOF.
- **Не** класть прод-Praefect в Kubernetes: в доке Praefect на K8s **beta / not GA**; Cloud Native референс: только **sharded** Gitaly, *each Gitaly pod is a single point of failure for the repositories it serves*.
- Sharded Gitaly в K8s (GA с 18.11) допустим, если сознательно принимаете: рестарт StatefulSet (апгрейд чарта, смена resources) = простой **этих** репо; cgroups и memory buffer обязательны, иначе OOM убивает под и все git-процессы.
- Бэкап Praefect: **не** snapshot одной ноды. Официальные backup/restore; incremental backup — способ ускорить.
- Известный баг: гонка `PostReceiveHook` vs реплика → CI не находит `refs/merge-requests/...`. Лечение вендора: retry джоба / fetch stage (`job stages attempts`). Это не «сломали YAML».

**Слабое место:** три Gitaly-пода в одной зоне + emptyDir + «у нас же Kubernetes». Выглядит как кластер, ведёт себя как один диск.

#### Webservice / Sidekiq (ядро отказоустойчивости *API для Runner*)

- Минимум **2** Webservice в разных зонах, health check на LB. Алгоритм LB: **least connections**, не round-robin (рекомендация референсов).
- Incremental logging **включён**, object storage для artifacts/logs/builds **до** этого. После включения записи логов на диск нет «защиты от кривого бакета».
- Sidekiq не в одном поде «на всё». Следить за очередями CI и `Ci::ScheduleBulkDeleteJobArtifactCronWorker` (раз в 30 мин чистит expired artifacts — если TTL не задали, бакет растёт вечно).
- Zero-downtime upgrade Hybrid **не обещать**.

Падение 1 Webservice: переживается. Падение ЦОДа, где жили **все** Webservice и writer БД: не переживается.

#### PostgreSQL и Redis (ядро отказоустойчивости *состояния CI*)

- Отдельные от GitLab хосты/кластеры.
- Redis Cluster **нельзя**. Incremental logging с Cluster — отдельный явный запрет (issue 224171 в доке).
- Два Redis в референсе Cloud Native: cache (можно eviction) и persistent (для Sidekiq/очередей — eviction нельзя «как кэш»).
- Учение: убили Redis primary / PG writer — Runner снова poll'ит, джобы не теряют метаданные.

Без этого учения у вас нет прода CI, есть надежда.

#### Object storage (ядро отказоустойчивости *артефактов и логов*)

- Consolidated object storage (один connection, разные bucket'ы: artifacts, lfs, uploads, packages, …).
- Helm: не локальные PVC Rails для артефактов.
- Distributed cache Runner — **тот же класс** S3, иначе каждый manager кэширует в пустоту.
- Класс отказа бакета ≥ класс отказа GitLab. Иначе «HA GitLab» = UI жив, артефакты взорвались.

#### Runner (ядро отказоустойчивости и масштаба *исполнения*)

- Несколько managers, токены `glrt-`, теги пулов: `build`, `test`, `deploy-prod`.
- `deploy-prod`: **protected**, узкий RBAC в целевых namespace, без права на Kafka-nodes.
- `concurrent` считать от реальных CPU job-подов, не от дефолта 20.
- Job-поды: `requests`/`limits`, отдельный namespace, ResourceQuota, NetworkPolicy default-deny + allow GitLab, Registry, Vault, Sonar, нужные API.
- Менеджер: `securityContext.privileged: false` (дефолт чарта). `runners.kubernetes.privileged = true` — только осознанный пул.
- Сборка образов: **BuildKit rootless** (дока GitLab). На Kubernetes + AppArmor может понадобиться annotation `unconfined` на **этот** контейнер — это дырка меньше, чем privileged DinD, но её надо согласовать с ИБ, не копировать молча.
- Не ставить Runner DaemonSet на ноды брокеров Kafka «чтобы было ближе к продю».

Падение 1 manager: переживается. Падение всех managers: pending. Падение GitLab при живых Runner: idle poll, пользы нет.

#### Безопасность прода (без этого контур не считается настроенным)

Три слоя:

1. **Секреты и идентичность** — нет legacy registration token; `glrt-` в Secret; root не «из установки»; IdP; 2FA админов; Protected/Masked variables; scope `CI_JOB_TOKEN`; секреты прода не в YAML.
2. **Сеть** — 443 только с нужных сетей; 8075/Praefect/PG/Redis не с мира; job-namespace не видит etcd и не видит шину Kafka; Registry не анонимный push.
3. **Исполнение** — эфемерные поды; без privileged на общем пуле; PSA; отдельные ноды/пул для исключений; образы джобов из своего Registry, pin digest.

Плюс чарт GitLab:

- выключить bundled Postgres/Redis/MinIO;
- pin **10.3.x / 19.3.x** и digest образов;
- `gitlab-runner.install`: либо false + отдельный чарт, либо полностью свои values (не дефолтный Runner в тот же namespace, что Gitaly);
- свой Ingress, свой cert.

##### Сильные / слабые стороны выбранной ИБ-схемы (Hybrid + kubernetes-executor + glrt- + BuildKit rootless + NetworkPolicy)

| Сильное | Слабое |
|---|---|
| Эфемерный под на джоб | Компрометация job = всё, что видит сервисный аккаунт пода |
| `glrt-` вместо shared registration token | Украденный `glrt-` регистрирует ещё менеджеры **этого** Runner |
| BuildKit без privileged | AppArmor unconfined; не 100% «как DinD по привычке» |
| Scope job token | Забыли allowlist — токен ходит по группе |
| Vault на короткие credentials | Пока не внедрили — секреты в UI GitLab (masked ≠ недоступен Developer'у) |
| Не ГОСТ | Если ИБ потребует СКЗИ — vanilla TLS это не закроет |

#### Kubernetes-специфика прода

- Чарт **`gitlab/gitlab` 10.3.x** в режиме Hybrid values (external PG/Redis/object storage, Gitaly external).
- Чарт **`gitlab/gitlab-runner`** с APP VERSION 19.3.x; RBAC create true; **не** `clusterWideAccess` без нужды.
- Job-поды не на control-plane (taint).
- PDB Webservice/Sidekiq: выкат не снимает все реплики зоны сразу.
- Мониторинг: экспортёры GitLab + метрики Runner (`listen_address`) в ваш Prometheus (отдельный документ), не «живёт под».

#### Порядок вывода в прод (этапы, не команда за командой)

1. Измерить RTT/потери между ЦОДами на 443, 5432, 6379, 8075, S3 API. Решить: A или B1/B2.
2. Зафиксировать редакцию (нужен ли Geo) и оценить RPS/параллельные джобы/размер репо — хотя бы грубо, не «терабайты озера».
3. Поднять HA PostgreSQL, Redis Sentinel ×2 роли, object storage. Проверить failover **без** GitLab.
4. Поднять Gitaly Cluster на VM (или сознательно sharded + принятый SPOF). Прогнать clone под нагрузкой.
5. Установить GitLab 19.3.x Hybrid, HTTPS, свои секреты, incremental logging, без bundled DB.
6. IdP, закрыть публичную регистрацию, создать Runner'ы через UI (`glrt-`).
7. Поставить managers в зонах, теги, Quota, NetworkPolicy, BuildKit на тестовом репо.
8. Подключить один боевой сервис: test → Quality Gate (Sonar) → image → deploy на не-прод.
9. Retention артефактов, запрет секретов в логах, protected `deploy-prod`.
10. Нагрузить пачкой пайплайнов. Смотреть pending, Gitaly, бакет, CPU нод джобов.
11. Учения: убить 1 Webservice; убить 1 Runner manager; убить 1 Gitaly (Praefect); убить ЦОД без writer; убить ЦОД с writer; для B1 — Geo failover; прогон restore из бэкапа на стенде.

Без пунктов 1, 3, 5 (incremental logging + бакет), 10 и 11 у вас нет платформы CI, есть namespace `gitlab`.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | Не требуется | Hybrid: ≥2 Webservice, HA **writer** PG, Redis Sentinel (не Cluster), бакет переживает 1 ЦОД, Praefect на VM **или** честный SPOF шарда; Runner managers в нескольких зонах; Geo только с Premium. Переживание **2** ЦОДов не обещать |
| Производительность / масштаб | Не требуется | Ось №1 — job-поды и `concurrent`; Gitaly под clone; Sidekiq под логи; retention артефактов; RPS-класс как старт, не смета; HPA не лечит монорепу |
| Безопасность | Дефолт паролей только в изоляции; без privileged | Нет registration token; `glrt-`; HTTPS; IdP; scope job token; NetworkPolicy; без DinD на общем пуле; pin 19.3.x; секреты не в Git |

**Не готов к проду**, если: дефолтный Helm с bundled Postgres/MinIO; Redis Cluster; NFS логов между Rails при нескольких нодах без incremental logging; один sharded Gitaly в K8s выдан за «HA как Praefect»; Praefect растянут на 3 ЦОДа без замера RTT; privileged DinD на нодах приложений; shared registration token; `latest`; три независимых GitLab «для HA»; ждут, что Runner переживёт падение GitLab; ждут, что CI заменит Kafka/Camunda/озеро; RTT не мерили; нет учения отказа writer и одной зоны Runner; нет бэкапа Gitaly официальным путём; Kaniko как стратегия на годы.

---

## Источники (чтобы не принимать на веру)

- Релиз GitLab 19.3 / Runner 19.3 (20 Aug 2026): https://docs.gitlab.com/releases/19/gitlab-19-3-released/  
- Политика поддержки (на дату документа: 19.3 / 19.2 / 19.1): https://docs.gitlab.com/policy/maintenance/  
- Критический патч 19.2.4 (17 Aug 2026): https://docs.gitlab.com/releases/patches/patch-release-gitlab-19-2-4-released/  
- Соответствие чарта 10.3.0 → GitLab 19.3.0: https://docs.gitlab.com/charts/installation/version_mappings/  
- Референс-архитектуры, RPS, HA от ~3k users, Geo, < 5 ms, multi-DC own risk, no cross-region single environment, least connections, no swap, Praefect DB: https://docs.gitlab.com/administration/reference_architectures/  
- Cloud Native: Gitaly в K8s только шарды, Praefect не в референсе, PG/Redis/object storage снаружи, размеры S/M/L/XL: https://docs.gitlab.com/administration/reference_architectures/cloud_native/  
- Дефолтный Helm ≠ прод, stateful наружу: https://docs.gitlab.com/charts/  
- Gitaly Cluster (Praefect): Free как фича, support Premium+; latency ms; single vs Geo; RPO/RTO одной ноды; K8s beta; snapshot одной ноды нельзя; гонка CI refs: https://docs.gitlab.com/administration/gitaly/praefect/  
- Gitaly on Kubernetes GA 18.11, ограничения disruption/OOM/cgroups: https://docs.gitlab.com/administration/gitaly/kubernetes/  
- Kubernetes executor (под на джоб, шаги prepare/build): https://docs.gitlab.com/runner/executors/kubernetes/  
- Чарт Runner, privileged default false: https://docs.gitlab.com/runner/install/kubernetes/ и values чарта `gitlab/gitlab-runner`  
- Безопасность Runner, privileged = breakout: https://docs.gitlab.com/runner/security/  
- DinD требует privileged: https://docs.gitlab.com/ci/docker/docker_in_docker/  
- BuildKit rootless как замена Kaniko: https://docs.gitlab.com/ci/docker/using_buildkit/  
- Архив Kaniko (3 Jun 2025): https://github.com/GoogleContainerTools/kaniko  
- Новый workflow токенов `glrt-`, снятие registration token в 20.0: https://docs.gitlab.com/ci/runners/new_creation_workflow/ и https://docs.gitlab.com/runner/register/  
- `CI_JOB_TOKEN`: https://docs.gitlab.com/ci/jobs/ci_job_token/  
- Артефакты, object storage, мультинода: https://docs.gitlab.com/administration/cicd/job_artifacts/  
- Incremental logging, Redis Cluster не поддерживается: https://docs.gitlab.com/administration/cicd/job_logs/  
- Object storage типы объектов: https://docs.gitlab.com/administration/object_storage/  
- Installation requirements (Redis Cluster unsupported, object storage для distributed): https://docs.gitlab.com/install/requirements/  
- Geo + Helm (два кластера, две БД): https://docs.gitlab.com/charts/advanced/geo/  
- Порты пакета: https://docs.gitlab.com/administration/package_information/defaults/  
- Опции чарта (`gitlab-runner.concurrent` default 20, `privileged` default false): https://docs.gitlab.com/charts/installation/command-line-options/  

Утверждения вида «GitLab CI переживёт два ЦОДа» или «N concurrent на интеграционный сервис» в документации вендора **отсутствуют** — поэтому в этом файле их нет. Порог **5 ms** есть как общее требование HA-сети; для Praefect отдельно — *ideally single-digit milliseconds* и *single location*. Их надо измерить у себя, а не заменить фразой «город один».
