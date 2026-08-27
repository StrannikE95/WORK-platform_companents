# Backstage 1.54.0 — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `Backstage.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

Backstage — внутренний портал для разработчиков: карта сервисов, шаблоны, документация рядом с кодом. Это **фреймворк**: вы собираете *свой* app командой `create-app` и кладёте **свой** образ. Готового «скачал с Docker Hub — работает в бою» нет. Чарт Helm **прямо пишет**: vanilla demo image для боя скорее всего не подходит.

Версия в этой инструкции: **Backstage 1.54.0** (релиз 18 августа 2026), Helm **`backstage/backstage` 2.8.2**. Базовый runtime шаблона 1.54 — `node:24-trixie-slim`.

Документация: https://backstage.io/docs/  
Чарт: `https://backstage.github.io/charts` (OCI `ghcr.io/backstage/charts/backstage`).

Обычный путь боя — stateless Deployment в Kubernetes + внешний PostgreSQL. Учебный путь — `yarn start` на ноутбуке (SQLite in-memory). Windows как хост боевых подов в схеме с Kubernetes не предполагается.

Один портал, чьи поды и **одна** PostgreSQL размазаны на несколько дата-центров, здесь **не собираем**. Состояние в PostgreSQL; каждый запрос каталога бьёт в базу. Порог задержки документация **не задаёт**. Поэтому writer базы — внутри одной площадки. Вторая площадка — восстановление Postgres + выкат Deployment **или** независимый экземпляр с тем же Git.

---

## О чём эта инструкция молча договорилась

1. PostgreSQL каталога — один кластер **внутри одного дата-центра**. Поды Backstage не ходят SQL через город «как кластер». Несколько подов не выбирают лидера.
2. Бой — Kubernetes в каждом дата-центре отдельно (см. `Kubernetes.install.md`). Prerequisites чарта: Kubernetes 1.25+; прогон 2.8.2 на вашей версии этим файлом **не** сертифицирован.
3. Self-hosted app, не Red Hat Developer Hub. Helm 2.8.2 описывает Deployment, не заменяет сборку.
4. Учебный стенд — закрытая сеть. SQLite, Guest и Lunr допустимы **только там**.
5. Цифр вашей нагрузки нет — нет фразы «хватит N реплик». `replicas: 3` в golden-path — пример.
6. Две честные боевые схемы: **один** портал в одном дата-центре **или** независимый Backstage **на площадку** с общим catalog в Git. Три портала без договорённости = три карты сервисов.
7. Два дата-центра: актив в первом или по экземпляру с общим git. Три — то же. Третий writer Postgres каталога **не** появляется.
8. Релиз 1.54.0 содержит critical fixes плагина Kubernetes — без этого патча линейки плагин в бой не включать.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** увидеть каталог, `catalog-info.yaml`, логин, один шаблон. **Не зачем:** доказывать отказ дата-центра.

### Что должно быть до установки

- Node **22 или 24** (политика: две соседние чётные; с 1.46.0 это 22 и 24), Yarn **4.4.1**, порты 3000 и 7007.
- Guest в Docker-контейнерах проект **не** предназначен.
- Сеть стенда не торчит в интернет.

### Установка (ноутбук — основной путь для учёбы)

```bash
npx @backstage/create-app@latest
# зафиксировать app на 1.54.0 (yarn backstage-cli versions:bump по документации)
yarn start
```

UI: `http://localhost:3000`, backend: **7007**. Дефолт базы — SQLite `:memory:`: рестарт = пустой каталог.

### Установка (Docker / Kubernetes — ближе к бою, всё ещё учёба)

Host build (`yarn tsc` + `yarn build:backend`), образ своего app:

```bash
docker run --rm -p 127.0.0.1:7007:7007 \
  -e POSTGRES_HOST=127.0.0.1 \
  <registry>/backstage-app:1.54.0
curl -s http://127.0.0.1:7007/.backstage/health/v1/liveness
```

Привязка к `127.0.0.1` обязательна. Нужен Postgres и не-Guest auth (тестовый OIDC/GitLab). Гостевой провайдер в контейнерах **не предназначен**.

Helm 2.8.2 с demo-образом — только пощупать чарт, не эталон app. Учебный Postgres `13.2-alpine` в k8s-гайде — **не** версия боя. `/metrics` в свежем app **нет**, пока сами не настроите OpenTelemetry (это отмечает Helm README).

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Реплики | 1 | Не учимся переживать выкат |
| База | SQLite или один Postgres | Нет HA |
| Вход | Guest на хосте **или** тестовый IdP; Guest не в контейнере | Документация: Guest не для prod UI |
| Search | Lunr | Индекс умрёт с процессом — на стенде ок |
| TechDocs | `builder: local` | S3 ещё нет |
| Permission | выкл / allow-all | Иначе не увидите сущности |
| Split backend | нет | Учит продукт, не DiscoveryService |

Чего **не** упрощаем: **1.54.0** и совместимость плагинов; один настоящий `catalog-info.yaml` type `url`; `app.baseUrl` / `backend.baseUrl` совпадают с браузером; breaking 1.54 — allowlist OAuth redirect.

### Как понять, что стенд живой

1. Каталог показывает сущность из Git.
2. Рестарт при SQLite `:memory:` — каталог пуст (ожидаемо). Не считать это нормой боя.
3. Demo Helm `latest` не уезжает в GitOps.

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Часы, официальный Getting Started | Нет SSO, нет общего поиска между процессами |
| | Успешный `yarn start` ≠ TechDocs-S3 и permission |
| | Guest приучает открытый 7007 |

Перед боем полезен **препрод**: свой образ 1.54.0, Postgres, SSO без Guest, Search не Lunr, TechDocs external если так в бою, `replicas ≥ 2` — в **одном** дата-центре.

---

## Бой: один живой дата-центр, второй — запас

**Зачем:** пережить отказ **пода** при живом Postgres; пережить отказ **дата-центра портала** ценой восстановления Postgres + выкат Deployment или работой второго независимого app. Цифр RPS нет.

### Почему кластер не растягиваем на несколько дата-центров

Каждый запрос каталога бьёт в PostgreSQL. Поды в трёх площадках на один writer через город — скрытый stretch SQL, не «HA Backstage». Документация порога задержки **не задаёт**.

### Как расставить машины

Backstage — **stateless Deployment**. PVC нужен Postgres и бакету TechDocs, не «диску каталога в поде».

Два допустимых рисунка (выберите один):

**P1 — один портал в одном дата-центре.**  
Deployment `replicas ≥ 2` (для rolling update; golden-path пример — **3**), PDB (`maxUnavailable: 1`; в чарте PDB по умолчанию `create: false` — включить самим), Ingress TLS, PostgreSQL HA **этого** дата-центра. Service `sessionAffinity: None` — sticky **не** требуется. Пользователи других площадок ходят по городу на Ingress. Падение первой площадки = нет UI, пока restore PG + поды во второй (RTO мерить).

**P2 — независимый Backstage на площадку, общий catalog в Git.**  
В каждом дата-центре свой Deployment + **своя** PostgreSQL в этом дата-центре (не общая база через город). Источник сущностей — одни и те же `catalog-info.yaml` / locations в Git. Это **несколько** порталов с одной картой из git, не один writer. Scaffolder, права и пользователи могут разъехаться — закладывать до боя. Не путать с «три эталона клиентов»: каталог — про ПО, не про озеро.

| Сколько площадок | P1 | P2 |
|---|---|---|
| **Две** | Актив в первой; вторая — DR Postgres + манифесты | По экземпляру в первой и второй, общий git |
| **Три** | То же + пользователи третьей на Ingress первой | Третий экземпляр или только git-mirror без UI |

**Не берём:** поды всех площадок на **одну** PostgreSQL (stretch SQL). Bitnami `postgresql.enabled` чарта как боевая база.

Кэш Redis/Valkey — если включили: тоже HA, иначе после failover кэша просто холоднее (не потеря каталога). Auth keys: static keystore или смириться с ротацией при рестарте.

### Что должно быть до боевой установки

- Свой образ в своём registry, immutable tag, Node 24 как в runtime.
- PostgreSQL 14…18 (окно «последние 5 major» на август 2026); учебный 13.2 из k8s-гайда не брать.
- IdP, SCM (GitLab). Секреты — Secret/Vault, не `app-config` в git.
- Encryption at rest etcd: Kubernetes Secret — base64, не шифрование.
- Замер сети: под↔Postgres, под↔Git/IdP, пользователь↔Ingress. Решение P1 vs P2.

### Порядок установки в активном дата-центре

1. App **1.54.0**: плагины, SignInPage **без Guest**, `app-config.production.yaml`, CORS под реальный URL.
2. `client: pg`, SSL; плагины создают `backstage_plugin_*` (или схемы).
3. Auth provider + sign-in resolver. Прогнать логин и OAuth redirect (ужесточение 1.54).
4. `catalog.locations` / providers, **catalog.rules** (User/Group не из произвольного pull request). `catalog.readonly: true`, если каталог — зеркало Git и UI-регистрация локаций запрещена.
5. Search **не Lunr** (старт — модуль Postgres). TechDocs: `builder: external` + S3/MinIO (`endpoint` + `s3ForcePathStyle: true` для MinIO). Кэш TechDocs требует ещё `backend.cache`.
6. `permission.enabled: true`, убрать allow-all. Политика по группам из каталога.
7. Образ, probes `/.backstage/health/v1/liveness` и `readiness` (≥ 1.29). Не выдумывать `/healthcheck`, если не включали старый путь.
8. Выложить `replicas ≥ 2` с spread по нодам, PDB, Ingress TLS. Желательно auth proxy **перед** 7007.

Helm 2.8.2 — свой `backstage.image.repository` + tag:

```bash
helm repo add backstage https://backstage.github.io/charts
helm install backstage backstage/backstage --version 2.8.2 \
  -f values-prod.yaml
# postgresql.enabled: false
# PDB в чарте по умолчанию create: false — включить самим
```

HPA чарта (`maxReplicas: 100`, CPU 80%) — **дефолт чарта, не расчёт**. Включать после метрик, иначе thrashing.

9. Метрики: OpenTelemetry, latency API, `catalog.processing.duration`, очередь scaffolder, пул базы. Алерт на отставание processing и 5xx.
10. Только потом — scaffolder с write в Git, kubernetes plugin к боевым кластерам, proxy к внутренним API, split backend.

`processingInterval` не ставить слишком коротким: сожрёте rate limit GitLab. Документация: это *желаемый минимум*, фактически может быть длиннее.

Выкат образа: сначала backend-плагины, потом frontend (skew policy), особенно при split.

### Правила конфигурации боя

| Параметр | В бою | Зачем |
|---|---|---|
| Образ | свой, не `ghcr.io/backstage/backstage:latest` | Чарт: demo не для prod |
| Guest / `dangerouslyDisableDefaultAuthPolicy` | нет | Иначе аноним/суперпользователь |
| Lunr на нескольких репликах | нет | Свой индекс в RAM у каждого процесса |
| TechDocs local + emptyDir | нет | Реплики не делят доки |
| Auth keys | понять рестарт; static keystore если нужно | Дефолт ключей теряется при рестарте |
| Kubernetes plugin | 1.54.0 | Critical security fixes линейки |
| `skipTLSVerify` | нет | Warning в 1.54 |
| UrlReader allow | минимум | Не metadata 169.254.169.254 |
| Proxy plugin | не инжектить Authorization во upstream | Иначе любой, кто дошёл до портала, бьёт секретами |
| `$text` в YAML | понимать риск | Пользователь может прочитать то, до чего у него нет прав в Git, *через* backend |
| Scaffolder | после политики; permission на `execute` | Job на хосте пода; по возможности user OAuth на запись в Git, не org-admin токен в поде |
| `automountServiceAccountToken` | сузить RBAC SA | В чарте по умолчанию `true` — для плагина Kubernetes может быть нужно |
| Контур | внутренний; WAF/auth proxy | Threat model: не публичный интернет; DoS-защита — ваша |

NetworkPolicy: 7007 с Ingress; egress в Postgres (локальный дата-центр), IdP, SCM, бакет. Не в весь кластер.

Split auth/catalog/scaffolder — когда есть требование ИБ или нагрузка; из коробки Discovery «два Deployment и заработало» **не обещано**. Для high-security threat model советует отдельный auth и **свою** базу. Frontend-бандл по умолчанию скачивается даже без логина; если карта плагинов секретна — experimental public entry.

`User`/`Group` — только из доверенного provider (LDAP/GitLab org). `Template` — узкий allow-list локаций. Смешивать индекс портала с бизнес-логами/персональными данными в одном OpenSearch — плохая идея даже если «оба OpenSearch».

### Как расти, когда появятся цифры нагрузки

1. Сначала одинаковые реплики за балансировщиком (`sessionAffinity` не требуется).
2. Растёт `catalog.processing.duration` — interval, rate limit Git, мощность Postgres, затем вынести catalog.
3. Очередь scaffolder — отдельный backend (изоляция CPU и секретов Git write).
4. Медленный UI — вынести frontend на CDN/NGINX; конфиг тогда не инжектится backend'ом в рантайме.
5. Поиск: старт на Postgres; когда индекс вырастет — OpenSearch (отдельный документ), не тот кластер, что логи/персональные данные.
6. Портал не хранит озеро: ёмкость = сущности + индекс + TechDocs.

### Проверки, без которых это ещё не бой

1. Версия app 1.54.0; образ свой; PostgreSQL writer в том же дата-центре, что поды P1.
2. SSO без Guest; permission не allow-all; нет `dangerouslyDisableDefaultAuthPolicy`.
3. Выключили один под — UI жив. Restore Postgres на стенде (реплики Backstage бэкапом не являются).
4. Для P2: изменение `catalog-info.yaml` появляется на обоих порталах (лаг processing — ожидаем).
5. Lunr не используется; TechDocs не с emptyDir.
6. Учение «площадка портала выключена»: время подъёма запаса замерить. Учение «Git недоступен»: UI открывается, каталог не обновляется — так и должно быть.

### Что хорошо и что плохо в схеме «портал в одном дата-центре» / «по экземпляру»

| Схема | Хорошо | Плохо |
|---|---|---|
| P1 один дата-центр | Один SSO, один набор плагинов | Падение площадки = нет UI до восстановления |
| P2 по экземпляру | Postgres локальный, нет stretch SQL | Несколько app; scaffolder и права разъедутся без дисциплины git |
| Обе | Согласовано с запретом stretch | Ошибка `app-config` на всех репликах сразу |

**Не готово к бою**, если: SQLite `:memory:`; Guest в контейнере; Helm demo `latest`; Lunr как поиск организации; `replicas: 1` без запаса; TechDocs local на emptyDir; permission выкл при разных ролях; поды трёх площадок на один Postgres через город; каталог назначен эталоном клиентских персональных данных.

---

## Откуда цифры и имена образов

- Релиз 1.54.0: https://github.com/backstage/backstage/releases/tag/v1.54.0
- Docker, Guest не для контейнеров: https://backstage.io/docs/deployment/docker
- Kubernetes, stateless + Postgres: https://backstage.io/docs/deployment/k8s/
- Scaling, replicas: https://backstage.io/docs/golden-path/deployment/scaling
- Search Lunr не для prod: https://backstage.io/docs/features/search/search-engines
- Helm 2.8.2, demo не для prod: https://artifacthub.io/packages/helm/backstage/backstage
- PostgreSQL каталога: `PostgreSQL.install.md`
- Правила и схема компонентов: `Backstage.md`
