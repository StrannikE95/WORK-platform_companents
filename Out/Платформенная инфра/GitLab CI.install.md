# GitLab 19.3.0 — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `GitLab CI.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

GitLab CI — конвейер поставки. Отдельного продукта «только CI» нет: нужен координатор (сам GitLab: репозитории, очередь работ, артефакты) **и** агент сборки (Runner), который спрашивает «есть работа?» и исполняет её. Ставите **свою** копию, не GitLab.com.

Версия в этой инструкции: **GitLab 19.3.0**, **GitLab Runner 19.3.0**, Helm **`gitlab/gitlab` 10.3.0** (mapping → 19.3.0). Чарт Runner пинить по колонке **APP VERSION = 19.3.0**, не по «последнему тегу чарта» (на дату соседнего файла виден `v0.91.0` под 19.2).

Документация: https://docs.gitlab.com/ci/ · чарт: https://docs.gitlab.com/charts/  
Патчи линии **19.3.x** ставить по мере выхода, не замирать на `.0`.

Обычный путь боя — Cloud Native Hybrid: Webservice и Sidekiq в Kubernetes; хранилище Git (Gitaly Cluster / Praefect) на виртуальных машинах; PostgreSQL, Redis и объектное хранилище — **снаружи**. Встроенные в чарт Postgres/Redis/MinIO — для знакомства. Цитата чарта: *The default Helm chart configuration is not intended for production.*

Учебный путь — пакет Linux (Omnibus) на одну машину. Windows как хост Gitaly в схеме с Kubernetes не предполагается.

Один environment GitLab, размазанный на несколько дата-центров, здесь **не собираем**. Поставщик для синхронного HA просит задержку **< 5 мс**; Praefect — *single location* и миллисекунды. *It is not supported to deploy a single GitLab environment across different regions.* Поэтому живой координатор — внутри одной площадки. Вторая площадка — Geo (если Premium) или восстановление из бэкапа. Runner'ы могут жить в других площадках и ходить на URL координатора.

---

## О чём эта инструкция молча договорилась

1. Gitaly Cluster, writer PostgreSQL и Redis **одного** инстанса живут в **одном** дата-центре. Между площадками — Geo (Premium/Ultimate, failover **ручной**) **или** бэкап + restore. Не открывайте между дата-центрами порты Gitaly 8075, Postgres и Redis Sentinel, если не строите осознанно другой контур.
2. Бой — Kubernetes в каждом дата-центре отдельно (см. `Kubernetes.install.md`). Совместимость чарта 10.3.0 с вашей версией Kubernetes смотреть в prerequisites **на момент установки**.
3. Учебный стенд — закрытая сеть. Omnibus all-in-one и bundled chart допустимы **только там**.
4. Цифр вашей нагрузки и RPS нет — нет фразы «хватит N подов Webservice». Дефолтный `helm install` с bundled Postgres — **не** размер S из таблицы референса (у S уже 6–9 Webservice).
5. Бакет артефактов в бою **есть или будет**. Копии Gitaly бэкап не заменяют: `git push --force` они послушно повторят.
6. Два дата-центра: координатор в первом; во втором — Runner'ы + Geo (если лицензия) или только restore. Три дата-центра: то же + Runner'ы в третьей. Третья площадка **не** член Praefect первой.
7. Сборка образов в бою — BuildKit rootless, без privileged Docker-in-Docker на общем пуле. Kaniko Google архивирован 3 июня 2025 — как стратегию не берём.
8. Zero-downtime upgrade Hybrid поставщик **не поддерживает** — окно выката закладываем.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** написать `.gitlab-ci.yml`, увидеть работу в UI, понять теги Runner и kubernetes-executor. **Не зачем:** доказывать отказ площадки и Praefect.

### Что должно быть до установки

- Linux-машина **или** однонодовый Kubernetes.
- HTTPS хотя бы с внутренним CA, если стенд дальше localhost.
- Токен Runner — **`glrt-`**, не legacy registration token.
- Сеть стенда не торчит в интернет.

### Установка (Omnibus — основной путь для учёбы)

Пакет **GitLab 19.3.0** (CE или EE) на один хост, встроенные PostgreSQL/Redis. Сразу свой пароль `root`, не `5iveL!fe` из старых гайдов.

```bash
# после gitlab-ctl reconfigure
curl -sI https://gitlab.example.local | head -1
gitlab-rake gitlab:env:info | grep -E 'GitLab|GitLab Shell'
# версия координатора 19.3.x
```

Один Runner 19.3.0: executor `docker` или `kubernetes`. Регистрация через **`glrt-`** (Admin → Runners → New instance runner). Legacy registration token не создавать «на будущее в бой» — снятие запланировано на GitLab **20.0**.

Проверка: UI открывается по HTTPS; pipeline с `echo` зелёный; убитый Runner оставляет UI живым и пайплайны в `pending`.

### Установка (Kubernetes — если стенд уже в кластере)

```bash
helm repo add gitlab https://charts.gitlab.io
helm search repo gitlab/gitlab --versions | grep 10.3.0
# appVersion должен быть 19.3.0
```

Чарт **10.3.0** как PoC. Bundled Postgres/Redis/MinIO на неделю допустимы; месяцами — вынести сразу, иначе миграция будет болью.

`privileged: false`. Сборка образа — BuildKit rootless или «работа без Docker». Встроенный `gitlab-runner` чарта на тесте можно оставить, в бою пересоберёте.

Praefect и Geo для знакомства с YAML **не нужны**.

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Praefect / Geo | нет | Не учимся переживать падение машины Git |
| Bundled PG/Redis | можно на короткий стенд | Не копия в бой |
| HPA Webservice | выкл | Нет RPS |
| Runner managers | 1 | Нет очереди |
| `privileged` | false | Иначе Docker-in-Docker станет нормой |
| Registration token | **запрещён** | Привычка уедет в бой |

Чего **не** упрощаем: версия 19.3.0 + Runner 19.3.0; HTTPS; `glrt-`; хотя бы одна работа реального сервиса, не только `echo`.

### Как понять, что стенд живой

1. UI 19.3.x; Runner 19.3.x в Admin → Runners.
2. Pipeline зелёный; лог виден после работы.
3. `privileged: false` на executor.
4. Снесли машину Omnibus — репозитории пропали. Всё лежало на одном диске. Так и должно быть на тесте.

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Часы, официальный Try GitLab / chart PoC | Нет HA объектного хранилища, нет Praefect |
| | Bundled MinIO приучает «данные в кластере» |
| | Зелёный `hello` ≠ clone монорепозитория и Hybrid |

Перед боем полезен **препрод**: внешние PG/Redis/бакет, 2 Webservice, Gitaly с PVC не emptyDir, 2 Runner manager, HTTPS, `glrt-`, без privileged — даже без боевого RPS. Всё это **в одном** дата-центре.

---

## Бой: один живой дата-центр, второй — запас (Geo или restore)

**Зачем:** пережить отказ **машины или пода внутри площадки координатора**; пережить отказ **целого дата-центра** ценой остановки CI до Geo-failover (Premium) или restore. Переживание **двух** площадок не обещать.

### Почему кластер не растягиваем на несколько дата-центров

Синхронный HA GitLab официально: *network latency … lower than 5 ms*; Praefect — *ideally single-digit milliseconds*, *single location*. Support multi-DC: *generally at your own risk*. Задержка между площадками это запрещает: Gitaly 8075, Postgres, Redis Sentinel **не** размазываем.

### Как расставить машины

**В активном дата-центре** — один Hybrid-инстанс:

- Webservice ≥ 2, Sidekiq не в одном поде «на всё», incremental logging **включён**;
- PostgreSQL — внешний HA **внутри этого дата-центра** (`PostgreSQL.install.md`). GitLab базу не кластеризует;
- Redis: **два** standalone HA (cache + persistent), **Sentinel**, не Redis Cluster (запрещён, в том числе для incremental logging);
- Объектное хранилище снаружи; бакет должен переживать отказ этой площадки **или** вы принимаете, что артефакты умрут вместе с ней;
- **Gitaly Cluster на VM**, Praefect + отдельная база Praefect, replication factor 3, все ноды **этого** дата-центра. Praefect в Kubernetes — **beta**, в референс Cloud Native не входит. Sharded Gitaly в Kubernetes = точка отказа **своих** репозиториев;
- Helm **10.3.0**, bundled Postgres/Redis/MinIO **выключены**. `gitlab-runner.install: false` + отдельный чарт Runner (или полностью свои values, не дефолт туториала);
- Балансировщик: **least connections**, не round-robin;
- PDB Webservice/Sidekiq: выкат не снимает все реплики сразу;
- Pin **10.3.x / 19.3.x** и digest образов.

**Между площадками:**

| Сколько площадок | Что где | Если умер активный дата-центр |
|---|---|---|
| **Две** | Первая: координатор HA. Вторая: Runner managers + job-поды. **Geo secondary** — только при Premium/Ultimate. Иначе бэкап Gitaly (официальные Rake) + PG + бакет и restore | CI стоит, пока failover Geo (ручной) или restore. RPO > 0 без Geo. Время простоя замерить |
| **Три** | То же + Runner'ы в третьей площадке | То же; третья площадка не даёт второго Gitaly-writer |

Три независимых GitLab = три правды кода — обычно ошибка.

Клиенты Runner в других площадках в штате ходят на **URL координатора первой** (TLS). Падение первой: живые Runner'ы спрашивают API в пустоту, пока Geo/restore.

### Что должно быть до боевой установки

- Kubernetes в каждом дата-центре (не stretch-кластер).
- Сначала HA PostgreSQL, Redis Sentinel ×2 роли, бакет — **потом** GitLab, потом Runner. Failover базы проверить **без** GitLab.
- Ingress/LB свой; PKI на Runner'ах.
- Редакция: нужен ли Geo — решить до установки.
- Unified URL закладывать сразу, если берёте Geo (гайд Geo+Helm).

### Порядок установки в активном дата-центре

1. Внешние PostgreSQL / Redis / объектное хранилище. Failover базы проверить **без** GitLab.
2. Gitaly Cluster на VM в **этом** дата-центре. Не emptyDir, не NFS как диск репозиториев. Прогнать clone под нагрузкой.
3. Чарт `gitlab/gitlab` **10.3.0**, Hybrid values:

```bash
helm upgrade --install gitlab gitlab/gitlab --version 10.3.0 \
  -f values-hybrid.yaml
# values: postgresql.install=false, redis.install=false, minio.install=false,
# gitlab-runner.install=false
```

4. HTTPS, свои секреты, incremental logging **до** второго Rails.
5. IdP, закрыть публичную регистрацию, создать Runner'ы через UI (`glrt-`).
6. Поставить managers (чарт `gitlab/gitlab-runner`, APP VERSION **19.3.0**) в каждой площадке, где есть job-ноды. Executor **kubernetes**. `securityContext.privileged: false`. `runners.kubernetes.privileged = true` — только изолированный пул, не ноды Kafka/Camunda.
7. Теги пулов: `build` / `test` / `deploy-prod` (`deploy-prod` — **protected**). `concurrent` — от CPU job-подов, не дефолт 20. Job-namespace: requests/limits, ResourceQuota, NetworkPolicy default-deny + allow GitLab, Registry, Vault, Sonar, нужные API.
8. Сборка образов: **BuildKit rootless**. Distributed cache — S3 того же класса, что артефакты.
9. Подключить один боевой сервис: test → Quality Gate (Sonar) → image → deploy на не-бой.
10. Retention артефактов (`expire_in` + Sidekiq delete; `Ci::ScheduleBulkDeleteJobArtifactCronWorker` раз в 30 мин чистит expired — если TTL не задали, бакет растёт вечно).
11. Бэкап Gitaly официальными Rake. Snapshot одной Praefect-ноды **не** бэкап. Restore прогнать на стенде.

Известный баг: гонка `PostReceiveHook` vs реплика → CI не находит `refs/merge-requests/...`. Лечение поставщика: retry работы / fetch stage. Это не «сломали YAML».

### Правила конфигурации боя

| Параметр | В бою | Зачем |
|---|---|---|
| Bundled PG/Redis/MinIO | false | Не PoC и не размер S |
| Redis Cluster | нельзя | Поставщик не поддерживает |
| Incremental logging | вкл | Иначе лог теряется между Rails |
| Praefect | VM, один дата-центр | Kubernetes — beta |
| `gitlab-runner.install` в чарте GitLab | false + отдельный чарт | Не дефолтный Runner рядом с Gitaly |
| Registration token | нет | Снятие в 20.0; `glrt-` |
| DinD privileged на общем пуле | нет | Цитата поставщика: отключает изоляцию |
| `CI_JOB_TOKEN` | allowlist проектов | Иначе токен ходит по группе шире, чем думаете |
| Секреты боя | Protected + Masked, лучше Vault | Masked ≠ недоступен Developer'у |
| Образы работ | свой Registry, pin digest | Не «всегда тянуть с Docker Hub» |
| 443 / 8075 / Postgres / Redis | 443 только с нужных сетей; остальное не с мира | Job-namespace не видит etcd и не видит шину Kafka |

**Geo** (только Premium/Ultimate): secondary — отдельный Kubernetes/база, failover **ручной**, консистентность eventual. Без лицензии не обещать «переключили DNS и git жив». Известные шероховатости Praefect+Geo — смотреть known issues поставщика.

Sharded Gitaly в Kubernetes (GA с 18.11) допустим, если сознательно принимаете: рестарт StatefulSet = простой **этих** репозиториев; cgroups и memory buffer обязательны, иначе OOM убивает под и все git-процессы.

### Как расти, когда появятся цифры нагрузки

1. Ось CI №1 — job-поды + `concurrent` + ноды Kubernetes, не «ещё Gitaly в чужом дата-центре».
2. Тяжёлый clone — Gitaly CPU/диск; shallow clone, не второй координатор.
3. Артефакты — бакет + `expire_in` + Sidekiq delete; терабайты озера сюда сами не попадают.
4. Больше UI/API → Webservice (HPA), PostgreSQL, Redis cache. HPA Webservice не лечит монорепозиторий.
5. Таблица S/M/L/XL по RPS — ориентир, не смета. Для S поставщик уже рисует 6–9 Webservice.

### Проверки, без которых это ещё не бой

1. Версии 19.3.x на GitLab и Runner.
2. Запись в UI, clone в работе, артефакт из объектного хранилища (не emptyDir).
3. Выключили один Webservice — Runner переподключается. Выключили один Runner manager — другие берут работы. Выключили один Gitaly при Praefect — CI жив **внутри первой площадки**.
4. Restore Gitaly официальным Rake на стенде.
5. Если Premium: учение Geo failover (ручное) на препроде. Если Free: замер RTO restore во второй площадке.
6. Работа без ACL/без `glrt-` не регистрирует чужой Runner. Privileged на общем пуле выкл.
7. Учение «первая площадка выключена»: живые Runner'ы poll'ят в пустоту — и это **ожидаемо**, пока Geo/restore.

### Что хорошо и что плохо в схеме «мозг в одном дата-центре + Runner'ы + Geo или restore»

| Хорошо | Плохо |
|---|---|
| Gitaly и Postgres не зависят от задержки между городами | Падение первой площадки = нет CI, пока Geo/restore |
| Согласовано с запретом stretch и с Praefect *single location* | RPO > 0 без синхронного git на две площадки |
| Runner в других площадках переживает отказ **исполнения** | Runner не переживает мёртвый координатор |
| | Geo — платная редакция; failover ручной |

**Не готово к бою**, если: дефолтный Helm с bundled Postgres; Redis Cluster; один sharded Gitaly выдан за Praefect; Praefect/Patroni на 2–3 дата-центра; privileged DinD на нодах приложений; registration token; `latest`; ждут, что Runner переживёт падение GitLab; Kaniko как стратегия.

---

## Откуда цифры и имена образов

- GitLab 19.3 / Runner 19.3: https://docs.gitlab.com/releases/19/gitlab-19-3-released/
- Чарт 10.3.0 → 19.3.0: https://docs.gitlab.com/charts/installation/version_mappings/
- Референс-архитектуры, < 5 мс, no cross-region: https://docs.gitlab.com/administration/reference_architectures/
- Дефолтный Helm ≠ прод: https://docs.gitlab.com/charts/
- Praefect, Geo: https://docs.gitlab.com/administration/gitaly/praefect/
- Incremental logging, Redis Cluster unsupported: https://docs.gitlab.com/administration/cicd/job_logs/
- BuildKit rootless: https://docs.gitlab.com/ci/docker/using_buildkit/
- Правила и схема компонентов: `GitLab CI.md`
