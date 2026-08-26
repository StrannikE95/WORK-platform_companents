# Backstage 1.54.0 — установка и конфигурирование

Связанный документ (глоссарий, catalog, permission, почему так): `Backstage.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Backstage (поды + **одна** PostgreSQL на несколько ЦОДов) **не делаем**: состояние в PostgreSQL, а она живёт в одном ЦОДе (`PostgreSQL.install.md`). Несколько подов не выбирают лидера.

Версии: **Backstage 1.54.0** (релиз 18 августа 2026), Helm **`backstage/backstage` 2.8.2**. Образ — **ваш** app из `create-app`, не demo. Базовый runtime шаблона 1.54 — `node:24-trixie-slim`.  
Документация: https://backstage.io/docs/  
Чарт: `https://backstage.github.io/charts` (OCI `ghcr.io/backstage/charts/backstage`). Чарт **прямо**: vanilla demo image **для прода скорее всего не подходит**.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** PostgreSQL каталога — один CNPG Cluster **внутри одного ЦОДа**. Поды Backstage не ходят SQL через город «как кластер».
2. Прод — Kubernetes 1.36.x в каждом ЦОДе (`Kubernetes.install.md`). Prerequisites чарта: K8s 1.25+; прогон 2.8.2 на 1.36 этим файлом **не** сертифицирован.
3. Self-hosted app, не Red Hat Developer Hub. Helm 2.8.2 описывает Deployment, не заменяет сборку.
4. Dev — изолированная сеть. SQLite/Guest/Lunr — только Dev.
5. Нагрузки нет — нет числа реплик «хватит» (`replicas: 3` в golden-path — пример).
6. Две честные прод-схемы: **один** портал в одном ЦОДе **или** независимый Backstage **на ЦОД** с общим catalog в Git. Три портала без договорённости = три карты сервисов.
7. Для 2 ЦОДов: актив в ЦОД-1 или по экземпляру с общим git. Для 3 — то же. Третий writer Postgres каталога **не** появляется.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** каталог, `catalog-info.yaml`, логин, один шаблон. **Не** цель: отказ ЦОДа.

### Предпосылки

- Node **22 или 24** (политика: две соседние чётные; с 1.46.0 это 22 и 24), Yarn **4.4.1**, порты 3000 и 7007.
- Guest в Docker-контейнерах вендор **не** предназначен.

### Установка (ноутбук — основной путь Dev)

```bash
npx @backstage/create-app@latest
# зафиксировать app на 1.54.0 (yarn backstage-cli versions:bump по доке)
yarn start
```

UI: `http://localhost:3000`, backend: **7007**. Дефолт БД — SQLite `:memory:`: рестарт = пустой каталог.

### Установка (Docker / Kubernetes Dev)

Host build (`yarn tsc` + `yarn build:backend`), образ своего app:

```bash
docker run --rm -p 127.0.0.1:7007:7007 \
  -e POSTGRES_HOST=127.0.0.1 \
  <registry>/backstage-app:1.54.0
curl -s http://127.0.0.1:7007/.backstage/health/v1/liveness
```

Нужен Postgres и не-Guest auth (тестовый OIDC/GitLab). Гостевой провайдер в контейнерах **не предназначен**.

Helm 2.8.2 с demo-образом — только пощупать чарт, не эталон app. Учебный Postgres `13.2-alpine` в k8s-гайде — **не** версия прода (у вас якорь 18.6). `/metrics` в свежем app **нет**, пока сами не настроите OpenTelemetry (это отмечает Helm README).

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Реплики | 1 | Некому переживать выкат |
| БД | SQLite или один PG | Нет HA |
| Auth | Guest на хосте **или** тестовый IdP; Guest не в контейнере | Дока: Guest не для prod UI |
| Search | Lunr | Индекс умрёт с процессом — на стенде ок |
| TechDocs | `builder: local` | S3 ещё нет |
| Permission | выкл / allow-all | Иначе не увидите сущности |
| Split backend | нет | Учит продукт, не DiscoveryService |

Чего **не** упрощать: **1.54.0** и skew плагинов; один настоящий `catalog-info.yaml` type `url`; `app.baseUrl` / `backend.baseUrl` совпадают с браузером; breaking 1.54 — allowlist OAuth redirect.

### Проверка Dev

1. Каталог показывает сущность из Git.
2. Рестарт при SQLite `:memory:` — каталог пуст (ожидаемо). Не считать это нормой прода.
3. Demo Helm `latest` не уезжает в GitOps.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, Getting Started | Нет SSO, нет общего поиска между процессами |
| | Успешный `yarn start` ≠ TechDocs-S3 и permission |
| | Guest приучает открытый 7007 |

Препрод: свой образ 1.54.0, Postgres, SSO без Guest, Search не Lunr, TechDocs external если так в проде, `replicas ≥ 2` — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода** при живом Postgres; пережить отказ **ЦОДа портала** ценой DR Postgres + выкат Deployment или работой второго независимого app. Цифр RPS нет.

### Почему не stretch

Каждый запрос каталога бьёт в PostgreSQL. Sync-реплика PG между ЦОДами запрещена ping'ом (`PostgreSQL.install.md`). Поды в трёх ЦОДах на один writer через город — скрытый stretch SQL, не «HA Backstage». Документация порога RTT **не задаёт**.

### Топология

Backstage — **stateless Deployment**. PVC нужен Postgres и бакету TechDocs, не «диску каталога в поде».

Два допустимых рисунка (выбрать один):

**P1 — один портал в одном ЦОДе.**  
Deployment `replicas ≥ 2`, PDB, Ingress TLS, PostgreSQL HA **этого** ЦОДа. Пользователи других площадок ходят по городу на Ingress. Падение ЦОД-1 = нет UI, пока restore PG + поды в ЦОД-2 (RTO мерить).

**P2 — независимый Backstage на ЦОД, общий catalog в Git.**  
В каждом ЦОДе свой Deployment + **своя** PostgreSQL в этом ЦОДе (не общая БД через город). Источник сущностей — одни и те же `catalog-info.yaml` / locations в Git. Это **несколько** порталов с одной картой из git, не один writer. Scaffolder/permission/users могут разъехаться — закладывать до боя. Не путать с «три SoT клиентов»: каталог — про ПО, не про озеро.

| Площадок | P1 | P2 |
|---|---|---|
| **2 ЦОДа** | Актив в ЦОД-1; ЦОД-2 — DR PG + манифесты | По экземпляру в ЦОД-1 и ЦОД-2, общий git |
| **3 ЦОДа** | То же + пользователи ЦОД-3 на Ingress ЦОД-1 | Третий экземпляр или только git-mirror без UI |

**Не берём:** поды всех ЦОДов на **одну** PostgreSQL (stretch SQL). Bitnami `postgresql.enabled` чарта как прод-БД.

### Предпосылки прода

- Свой образ в своём registry, immutable tag, Node 24 как в runtime.
- PostgreSQL 14…18 (окно «последние 5 major» на август 2026); якорь платформы — **18.6**, не учебный 13.2.
- IdP, SCM (GitLab). Секреты — Secret/Vault, не `app-config` в git.
- Encryption at rest etcd: K8s Secret — base64, не шифрование.

### Установка

1. App **1.54.0**: плагины, SignInPage **без Guest**, `app-config.production.yaml`, CORS под реальный URL.
2. `client: pg`, SSL; плагины создают `backstage_plugin_*` (или схемы).
3. Auth provider + sign-in resolver. Прогнать логин и OAuth redirect (ужесточение 1.54).
4. `catalog.locations` / providers, **catalog.rules** (User/Group не из произвольного PR).
5. Search **не Lunr** (старт — `search-backend-module-pg`). TechDocs: `builder: external` + S3/MinIO (`s3ForcePathStyle` для MinIO).
6. `permission.enabled: true`, убрать allow-all.
7. Образ, probes `/.backstage/health/v1/liveness` и `readiness` (≥ 1.29). Не выдумывать `/healthcheck`, если не включали старый путь.

Helm 2.8.2 — свой `backstage.image.repository` + tag:

```bash
helm repo add backstage https://backstage.github.io/charts
helm install backstage backstage/backstage --version 2.8.2 \
  -f values-prod.yaml
# postgresql.enabled: false
# PDB в чарте по умолчанию create: false — включить самим
```

HPA чарта (`maxReplicas: 100`, CPU 80%) — **дефолт чарта, не расчёт**. Включать после метрик.

### Конфигурирование

| Параметр | Прод | Зачем |
|---|---|---|
| Образ | свой, не `ghcr.io/backstage/backstage:latest` | Чарт: demo не для prod |
| Guest / `dangerouslyDisableDefaultAuthPolicy` | нет | Иначе аноним/суперпользователь |
| Lunr на нескольких репликах | нет | Свой индекс в RAM у каждого процесса |
| TechDocs local + emptyDir | нет | Реплики не делят доки |
| Auth keys | понять рестарт; static keystore если нужно | Дефолт ключей теряется при рестарте |
| Kubernetes plugin | 1.54.0 | Critical security fixes линейки |
| `skipTLSVerify` | нет | Warning в 1.54 |
| UrlReader allow | минимум | Не metadata 169.254.169.254 |

NetworkPolicy: 7007 с Ingress; egress в Postgres (локальный ЦОД), IdP, SCM, бакет. Scaffolder — после политики: job на хосте пода, write в Git.

Безопасность (без этого портал — карта атаки):

- нет Guest; нет `backend.auth.dangerouslyDisableDefaultAuthPolicy`;
- `permission.enabled: true`; User/Group только из доверенного provider;
- `backend.reading.allow` минимальный (не 169.254.169.254);
- proxy plugin: не инжектить Authorization во upstream;
- Kubernetes plugin **1.54.0** (critical fixes); `skipTLSVerify: true` не оставлять;
- контур внутренний; DoS-защита — proxy/WAF оператора, в Backstage её почти нет;
- threat model: рассчитан на защищённый периметр, не публичный интернет.

Split auth/catalog/scaffolder — когда есть требование ИБ или нагрузка; из коробки Discovery «два Deployment и заработало» **не обещано**.

### Масштабирование (когда появятся цифры)

1. Сначала одинаковые реплики за LB (`sessionAffinity` не требуется).
2. Растёт `catalog.processing.duration` — interval, rate limit SCM, мощность PG, затем вынести catalog.
3. Очередь scaffolder — отдельный backend.
4. Портал не хранит озеро: ёмкость = сущности + индекс + TechDocs.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия app 1.54.0; образ свой; PG 18.6 в том же ЦОДе, что writer.
2. SSO без Guest; permission не allow-all.
3. Убить 1 под — UI жив. Restore Postgres на стенде (реплики Backstage бэкапом не являются).
4. Для P2: изменение `catalog-info.yaml` появляется на обоих порталах (лаг processing — ожидаем).
5. Lunr не используется; TechDocs не с emptyDir.

### Сильные / слабые стороны

| Схема | Сильное | Слабое |
|---|---|---|
| P1 один ЦОД | Один SSO, один набор плагинов | Падение площадки = нет UI до DR |
| P2 по экземпляру | PG локальный, нет stretch SQL | Несколько app; scaffolder/права разъедутся без дисциплины git |
| Обе | Согласовано с запретом stretch | Ошибка `app-config` на всех репликах сразу |

**Не готов к проду**, если: SQLite/` :memory:`, Guest в контейнере, Helm demo `latest`, Lunr как поиск организации, `replicas: 1` без DR, TechDocs local на emptyDir, permission выкл при разных ролях, поды трёх ЦОДов на один PG через город, каталог назначен SoT клиентских ПДн.

---

## Источники

- Релиз 1.54.0: https://github.com/backstage/backstage/releases/tag/v1.54.0
- Docker, Guest не для контейнеров: https://backstage.io/docs/deployment/docker
- Kubernetes, stateless + Postgres: https://backstage.io/docs/deployment/k8s/
- Scaling, replicas: https://backstage.io/docs/golden-path/deployment/scaling
- Search Lunr не для prod: https://backstage.io/docs/features/search/search-engines
- Helm 2.8.2, demo не для prod: https://artifacthub.io/packages/helm/backstage/backstage
- PostgreSQL каталога: `PostgreSQL.install.md`
- Правила: `Backstage.md`
