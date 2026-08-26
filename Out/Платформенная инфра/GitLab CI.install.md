# GitLab 19.3.0 — установка и конфигурирование

Связанный документ (глоссарий, Gitaly/Praefect, Geo, почему так): `GitLab CI.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного GitLab (Gitaly, PostgreSQL, Redis) на несколько ЦОДов **не делаем**: вендор для HA просит задержку **< 5 ms**, Praefect — *single location* и миллисекунды; межЦОДовый RTT это запрещает. Отдельного продукта «только CI» нет: нужен координатор (GitLab) **и** Runner.

Версии: **GitLab 19.3.0**, **GitLab Runner 19.3.0**, Helm **`gitlab/gitlab` 10.3.0** (mapping → 19.3.0). Чарт Runner пинить по колонке **APP VERSION = 19.3.0**, не по «последнему тегу чарта» (на дату соседнего файла виден `v0.91.0` под 19.2).  
Документация: https://docs.gitlab.com/ci/ · чарт: https://docs.gitlab.com/charts/  
Патчи линии **19.3.x** ставить по мере выхода, не замирать на `.0`.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Один environment GitLab **между ЦОДами** не размазываем. Gitaly Cluster и writer PostgreSQL живут **внутри одного ЦОДа**.
2. Прод — Kubernetes 1.36.x в каждом ЦОДе (`Kubernetes.install.md`). Совместимость чарта 10.3.0 с 1.36 — смотреть prerequisites **на момент установки**, не этот абзац как вечную таблицу.
3. Self-managed, не GitLab.com. Cloud Native Hybrid: Webservice/Sidekiq в Kubernetes; **Gitaly Cluster (Praefect) на VM**; PostgreSQL, Redis, object storage — **снаружи**. Bundled Postgres/Redis/MinIO чарта — **не** reference architecture S.
4. Dev — изолированная сеть.
5. Нагрузки/RPS нет — нет числа подов Webservice «хватит».
6. Geo — **Premium/Ultimate**, не Free. Без лицензии межплощадочный DR координатора = бэкап + restore.
7. Для 2 ЦОДов: координатор HA в ЦОД-1, в ЦОД-2 — Runner'ы + Geo (если лицензия) **или** только restore. Для 3 — то же + Runner'ы в ЦОД-3, не третий writer Gitaly.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** `.gitlab-ci.yml`, джоб в UI, kubernetes-executor. **Не** цель: отказ площадки и Praefect.

### Предпосылки

- Linux VM **или** Dev-Kubernetes.
- HTTPS хотя бы с внутренним CA, если стенд дальше localhost.
- Токен Runner — **`glrt-`**, не legacy registration token.

### Установка (Omnibus — основной путь Dev)

Пакет **GitLab 19.3.0** (CE или EE) на один хост, встроенные PostgreSQL/Redis. Сразу свой пароль `root`, не `5iveL!fe`.

```bash
# после gitlab-ctl reconfigure
curl -sI https://gitlab.example.local | head -1
gitlab-rake gitlab:env:info | grep -E 'GitLab|GitLab Shell'
# версия координатора 19.3.x
```

Один Runner 19.3.0: executor `docker` или `kubernetes`. Регистрация через **`glrt-`** (Admin → Runners → New instance runner). Legacy registration token не создавать «на будущее в прод» — снятие запланировано на GitLab **20.0**.

Проверка: UI открывается по HTTPS; pipeline с `echo` зелёный; убитый Runner оставляет UI живым и пайплайны в `pending`.

### Установка (Helm Dev, если стенд уже в K8s)

```bash
helm repo add gitlab https://charts.gitlab.io
helm search repo gitlab/gitlab --versions | grep 10.3.0
# appVersion должен быть 19.3.0
```

Чарт **10.3.0** как PoC. Вендор: *The default Helm chart configuration is not intended for production.* Bundled Postgres/Redis/MinIO на неделю допустимы; месяцами — вынести сразу.

`privileged: false`. Сборка образа — BuildKit rootless или «джоб без Docker». Встроенный `gitlab-runner` чарта на тесте можно оставить, в проде пересоберёте.

Praefect и Geo для знакомства с YAML **не нужны**.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Praefect / Geo | нет | Нет требования пережить узел git |
| Bundled PG/Redis | можно на короткий стенд | Не копия в прод |
| HPA Webservice | выкл | Нет RPS |
| Runner managers | 1 | Нет очереди |
| `privileged` | false | Иначе DinD станет нормой |
| Registration token | **запрещён** | Привычка уедет в прод |

Чего **не** упрощать: версия 19.3.0 + Runner 19.3.0; HTTPS; `glrt-`; хотя бы один джоб реального сервиса, не только `echo`.

### Проверка Dev

1. UI 19.3.x; Runner 19.3.x в Admin → Runners.
2. Pipeline зелёный; лог виден после джоба.
3. `privileged: false` на executor.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, Try GitLab / chart PoC | Нет object storage HA, нет Praefect |
| | Bundled MinIO приучает «данные в кластере» |
| | Зелёный `hello` ≠ clone монорепы и Hybrid |

Препрод: внешние PG/Redis/бакет, 2 Webservice, Gitaly с PVC не emptyDir, 2 Runner manager, HTTPS, `glrt-`, без privileged — даже без боевого RPS.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа координатора**; пережить отказ **целого ЦОДа** ценой остановки CI до Geo-failover (Premium) или restore. Переживание **двух** ЦОДов не обещать.

### Почему не stretch

Синхронный HA GitLab официально: *network latency … lower than 5 ms*; Praefect — *ideally single-digit milliseconds*, *single location*. *It is not supported to deploy a single GitLab environment across different regions.* Support multi-DC: *generally at your own risk*. МежЦОДовый ping это запрещает: Gitaly 8075, Postgres, Redis Sentinel **не** размазываем.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один Hybrid-инстанс:

- Webservice ≥ 2, Sidekiq не в одном поде «на всё», incremental logging **включён**;
- PostgreSQL — внешний HA **внутри этого ЦОДа** (`PostgreSQL.install.md`: CNPG в одном Kubernetes, не stretch). GitLab БД не кластеризует;
- Redis: **два** standalone HA (cache + persistent), **Sentinel**, не Redis Cluster (запрещён, в т.ч. для incremental logging);
- Object storage снаружи; бакет должен переживать отказ ЦОД-1 **или** вы принимаете, что артефакты умрут вместе с ним;
- **Gitaly Cluster на VM**, Praefect + отдельная БД Praefect, replication factor 3, все ноды **этого** ЦОДа. Praefect в Kubernetes — **beta**, в референс Cloud Native не входит. Sharded Gitaly в K8s = SPOF **своих** репозиториев;
- Helm **10.3.0**, bundled Postgres/Redis/MinIO **выключены**. Это не размер S из таблицы RPS.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: координатор HA. ЦОД-2: Runner managers + job-поды. **Geo secondary** — только при Premium/Ultimate. Иначе бэкап Gitaly (официальные Rake) + PG + бакет и restore | CI стоит, пока failover Geo (ручной) или restore. RPO > 0 без Geo |
| **3 ЦОДа** | То же + Runner'ы в ЦОД-3 | То же; третий ЦОД не даёт второго Gitaly-writer |

Три независимых GitLab = три SoT кода — обычно ошибка.

Дефолтный `helm install` с bundled Postgres **не** reference architecture S (у S уже 6–9 Webservice, внешние PG/Redis, 3 Gitaly).

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не stretch-кластер).
- Сначала HA PostgreSQL, Redis Sentinel ×2 роли, бакет — **потом** GitLab, потом Runner.
- Ingress/LB свой; PKI на Runner'ах.
- Редакция: нужен ли Geo — решить до установки.
- Zero-downtime upgrade Hybrid вендор **не поддерживает** — окно выката закладываем.

### Установка координатора (ЦОД-1)

1. Внешние PG / Redis / object storage. Failover БД проверить **без** GitLab.
2. Gitaly Cluster на VM в **этом** ЦОДе. Не emptyDir, не NFS как диск репозиториев.
3. Чарт `gitlab/gitlab` **10.3.0**, Hybrid values: external PG/Redis/object storage, Gitaly external.

```bash
helm upgrade --install gitlab gitlab/gitlab --version 10.3.0 \
  -f values-hybrid.yaml
# values: postgresql.install=false, redis.install=false, minio.install=false,
# gitlab-runner.install=false (или полностью свои values, не дефолт туториала)
```

4. HTTPS, свои секреты, incremental logging **до** второго Rails.
5. IdP, закрыть публичную регистрацию, Runner'ы через UI (`glrt-`).

### Установка Runner (каждый ЦОД, где есть job-ноды)

Чарт `gitlab/gitlab-runner`, APP VERSION **19.3.0**. Executor **kubernetes**. `securityContext.privileged: false`. `runners.kubernetes.privileged = true` — только изолированный пул, не ноды Kafka/Camunda.

Теги: `build` / `test` / `deploy-prod` (`deploy-prod` — **protected**). `concurrent` — от CPU job-подов, не дефолт 20. Job-namespace: requests/limits, ResourceQuota, NetworkPolicy default-deny.

Сборка образов: **BuildKit rootless** (дока GitLab; Kaniko Google архивирован 3 июня 2025). Distributed cache — S3 того же класса, что артефакты.

### Конфигурирование (смысл, не полный values)

| Параметр | Прод | Зачем |
|---|---|---|
| Bundled PG/Redis/MinIO | false | Не PoC и не размер S |
| Redis Cluster | нельзя | Вендор не поддерживает |
| Incremental logging | вкл | Иначе лог теряется между Rails |
| Praefect | VM, один ЦОД | K8s — beta |
| `gitlab-runner.install` в чарте GitLab | false + отдельный чарт | Не дефолтный Runner рядом с Gitaly |
| Registration token | нет | Снятие в 20.0; `glrt-` |
| DinD privileged на общем пуле | нет | Цитата вендора: отключает изоляцию |

Клиенты Runner в ЦОД-2/3 в штате ходят на **URL координатора ЦОД-1** (TLS). Падение ЦОД-1: живые Runner'ы poll'ят в пустоту, пока Geo/restore.

**Geo** (только Premium/Ultimate): secondary — отдельный Kubernetes/БД, failover **ручной**, консистентность eventual. Unified URL закладывать сразу (гайд Geo+Helm). Без лицензии не обещать «переключили DNS и git жив». Известные шероховатости Praefect+Geo — смотреть known issues вендора, не игнорировать.

Безопасность координатора и job-подов:

- нет shared registration token; `glrt-` в Secret; scope `CI_JOB_TOKEN` (allowlist проектов);
- 443 только с нужных сетей; 8075/Praefect/PG/Redis не с мира;
- job-namespace не видит etcd и не видит шину Kafka;
- секреты прода — Protected + Masked, лучше Vault short-lived, не вечный ключ в UI;
- образы джобов из своего Registry, pin digest.

### Масштабирование (когда появятся цифры)

1. Ось CI №1 — job-поды + `concurrent` + ноды Kubernetes, не «ещё Gitaly в чужом ЦОДе».
2. Тяжёлый clone — Gitaly CPU/диск; shallow clone, не второй координатор.
3. Артефакты — бакет + `expire_in` + Sidekiq delete; терабайты озера сюда сами не попадают.
4. HPA Webservice не лечит монорепу.

### Проверка прода (пока это не пройдено — это не прод)

1. Версии 19.3.x на GitLab и Runner.
2. Запись в UI, clone в джобе, артефакт из object storage (не emptyDir).
3. Убить 1 Webservice; убить 1 Runner manager; убить 1 Gitaly при Praefect — CI жив **внутри ЦОД-1**.
4. Restore Gitaly официальным Rake на стенде. Snapshot одной Praefect-ноды **не** бэкап.
5. Если Premium: учение Geo failover (ручное) на препроде. Если Free: замер RTO restore в ЦОД-2.
6. Джоб без ACL/без `glrt-` не регистрирует чужой Runner. Privileged на общем пуле выкл.

### Сильные / слабые стороны (мозг в одном ЦОДе + Runner'ы + Geo или restore)

| Сильное | Слабое |
|---|---|
| Gitaly/PG не зависят от межЦОДового RTT | Падение ЦОД-1 = нет CI, пока Geo/restore |
| Согласовано с запретом stretch и с Praefect *single location* | RPO > 0 без синхронного git на две площадки |
| Runner в других ЦОДах переживает отказ **исполнения** | Runner не переживает мёртвый координатор |
| | Geo — платная редакция; failover ручной |

**Не готов к проду**, если: дефолтный Helm с bundled Postgres; Redis Cluster; один sharded Gitaly выдан за Praefect; Praefect/Patroni на 2–3 ЦОДа; privileged DinD на нодах приложений; registration token; `latest`; ждут, что Runner переживёт падение GitLab; Kaniko как стратегия.

---

## Источники

- GitLab 19.3 / Runner 19.3: https://docs.gitlab.com/releases/19/gitlab-19-3-released/
- Чарт 10.3.0 → 19.3.0: https://docs.gitlab.com/charts/installation/version_mappings/
- Референс-архитектуры, < 5 ms, no cross-region: https://docs.gitlab.com/administration/reference_architectures/
- Дефолтный Helm ≠ прод: https://docs.gitlab.com/charts/
- Praefect, Geo: https://docs.gitlab.com/administration/gitaly/praefect/
- Incremental logging, Redis Cluster unsupported: https://docs.gitlab.com/administration/cicd/job_logs/
- BuildKit rootless: https://docs.gitlab.com/ci/docker/using_buildkit/
- Правила: `GitLab CI.md`
