# OpenSearch Dashboards 3.8.0 — установка и конфигурирование

Связанный документ (глоссарий, cookie, saved objects, почему так): `OpenSearch Dashboards.md`.  
Кластер поиска: `OpenSearch.md` и `OpenSearch.install.md`.

Этот файл — **как поставить и настроить UI**. Stretch «одного OSD на все ЦОДы» при кластере только в ЦОД-1 **не заменяет** HA поиска: OSD без 9200 — красный экран. Деплоим **вместе с OpenSearch того ЦОДа**, куда указывает `opensearch.hosts`.

Версии: **OpenSearch Dashboards 3.8.0**, образ `opensearchproject/opensearch-dashboards:3.8.0`. На Kubernetes — секция `spec.dashboards` **оператора** вместе с кластером; Helm `opensearch/opensearch-dashboards` — запасной путь.  
Документация: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/

OSD — **другое ПО**, чем OpenSearch: Node.js UI, порт **5601**. Saved objects живут **в индексах** `.kibana*` / `.opensearch_dashboards*` кластера, не на диске пода.

---

## Допущения этой инструкции

1. **Stretch запрещён** в том же смысле, что у кластера: не смешивать URL leader и follower в одном `opensearch.hosts` «как один кластер». Реплики OSD — stateless, смотрят на **локальный** кластер своего ЦОДа.
2. Версия OSD = версия OpenSearch (**3.8.0**). Плагины — major.minor.patch.
3. Прод — Kubernetes в каждом ЦОДе отдельно. Если OpenSearch только в ЦОД-1 — OSD тоже в ЦОД-1 (иначе каждый клик Discover едет через город без пользы для HA данных).
4. Dev — изолированная сеть; HTTP и demo `kibanaserver` допустимы только там.
5. Нагрузки (число аналитиков) нет — минимум прода: **≥ 2** реплики в том ЦОДе, где кластер.
6. Для 2 ЦОДов: OSD у leader в ЦОД-1; у CCR-follower в ЦОД-2 — **отдельный** OSD на follower (read-only ожидание) **или** нет UI там. Для 3 ЦОДов: то же. Wazuh indexer — **не** data source этого OSD.
7. Сессия — cookie; Redis для HA OSD официально не требуется. Одинаковый `cookie.password` (≥ 32 символов) на всех репликах **этого** Deployment.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** Discover, index pattern, Security UI. **Не** цель: отказ ЦОДа и N сессий.

### Предпосылки

- Уже запущен OpenSearch 3.8.0 (см. `OpenSearch.install.md`), сеть Docker `os-net` или localhost.
- Порт 5601 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

Файл `opensearch_dashboards.yml` как в install-гайде: `opensearch.hosts` на контейнер кластера, `opensearch.ssl.verificationMode: none` допустим **только здесь**, учётка `kibanaserver`.

```bash
docker run -d --name osd-dev \
  --network os-net \
  -p 127.0.0.1:5601:5601 \
  -v "$PWD/opensearch_dashboards.yml:/usr/share/opensearch-dashboards/config/opensearch_dashboards.yml:ro" \
  opensearchproject/opensearch-dashboards:3.8.0
```

Привязка к `127.0.0.1` обязательна. Тег **3.8.0**, не `latest` (гайд OpenSearch иногда показывает `latest` — не копировать).

Проверка: браузер `http://127.0.0.1:5601`, логин admin кластера, index pattern, документ **виден**.

На Kubernetes Dev: `spec.dashboards.enable: true`, `replicas: 1`, HTTP. **Не** этот values в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Реплики | 1 | Нет HA-цели |
| TLS 5601 | HTTP | Иначе cert раньше Discover |
| `verificationMode` | `none` как в docker-гайде | Self-signed кластера |
| `kibanaserver` | demo **только в изоляции** | Кластер с 2.12 всё равно требует свой admin |
| cookie.password | дефолт | Одна реплика |
| Snapshot `.kibana*` | можно отложить на день | На препроде нельзя забыть |

Чего **не** упрощать: версия = кластеру; картинки живут в `.kibana*`, не в контейнере OSD; 5601 не в интернет.

### Проверка Dev

1. Логин, Discover показывает тестовый индекс.
2. Удалили volume OSD — дашборды на месте. Удалили `.kibana*` — нет (это урок).
3. Helm-привычку `updateStrategy: Recreate` не считать нормой прода.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Совпадает с официальным Docker-гайдом | Нет доказательства cookie на нескольких репликах |
| Один процесс | Demo `kibanaserver` и дефолтный cookie.password приучают «так оставим» |
| | Успех на одном поде ≠ Ingress+SSO |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода OSD внутри ЦОДа** кластера. Отказ ЦОДа с OpenSearch = нет UI, сколько ни ставьте OSD в «пустых» площадках. Падение OSD ≠ падение 9200.

### Почему не stretch

`opensearch.hosts` — ноды **одного** кластера (разъяснение maintainer на форуме проекта). Список leader+follower как fallback = неподдерживаемая лотерея. Latency UI = RTT до 9200. Реплики OSD в ЦОД-2 при мёртвом 9200 ЦОД-1 дают ложное чувство HA.

### Топология

OSD **stateless** (Deployment, PVC поиска не нужен). Saved objects — replica/awareness/snapshot **OpenSearch**.

**Внутри ЦОДа, где живёт кластер (ЦОД-1):**

- `replicas ≥ 2` (лучше 3 по нодам **этого** ЦОДа);
- `opensearch.hosts` — Service coordinating **этого** кластера, HTTPS, `verificationMode: full`;
- свой `kibanaserver` (не demo); роль `kibana_server`;
- свой `opensearch_security.cookie.password` ≥ 32, один Secret на Deployment;
- `cookie.secure: true`, если браузер ходит HTTPS;
- Ingress TLS; cookie.secure согласовать с терминацией (классика: Ingress HTTPS + `cookie.secure` при HTTP до пода — логин «не работает»);
- PDB; если Helm — сменить дефолт **`Recreate` на `RollingUpdate`**;
- `NODE_OPTIONS=--max-old-space-size=…` **ниже** memory limit;
- дефолт Helm 512M **не** копировать; README чарта говорит про 4–8 ГиБ *available* на хосте — это не ваша смета QPS.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | OSD только у кластера ЦОД-1. Если CCR в ЦОД-2 — **отдельный** OSD на follower, свой `.kibana*`, не админка Security «как на leader» | Нет UI leader. OSD ЦОД-2, если есть, видит только follower (read-only) |
| **3 ЦОДа** | Не ставить OSD «во все зоны для HA», если 9200 только в ЦОД-1 | То же |

`data_source.enabled` — отдельное решение (второй Security realm в одной сессии). Timeline с этим режимом **не** поддерживается. Wazuh в этот UI не подключать.

### Предпосылки прода

- Живой OpenSearch 3.8.0 в том же ЦОДе, HTTP TLS на 9200, пользователь процесса создан.
- Заголовок `securitytenant` в `opensearch.requestHeadersAllowlist` вместе с `Authorization` — иначе OSD стартует **red** (дока).
- Snapshot кластера включает `.kibana*`. Проверять restore этих индексов.
- SSO (OIDC/SAML) + запасной `basicauth` (`?auto_login=false`, если redirect). `admin` в UI ежедневно запрещён.

### Установка (оператор, ЦОД кластера)

В CR OpenSearch:

```yaml
spec:
  dashboards:
    enable: true
    version: "3.8.0"
    replicas: 2
    # tls: лучше валидный cert на Ingress; внутри — generate или Secret
```

Секреты: пароль kibanaserver, cookie.password, OIDC `client_secret`. Оператор без вашего секрета **генерирует** пароль в `<cluster>-dashboards-password` — лучше demo, но Secret включить в эксплуатацию. Смена Secret OIDC **без** рестарта подов не подхватывается (дока оператора).

Helm-запас: `replicaCount ≥ 2`, `updateStrategy.type: RollingUpdate`, ресурсы не 100m/512M из values.

### Конфигурирование прода

| Слой | Правило |
|---|---|
| Сеть | 5601 не в интернет без SSO. NetworkPolicy: Ingress → OSD → 9200 coordinating |
| kibanaserver | Не пускать людей этим логином |
| Multi-tenancy | Задать явно, согласовать с `config.yml`. Private tenant × сотни пользователей = сюрприз heap manager |
| SAML ACS | Стабильный URL LB, не Pod IP; в `server.xsrf.allowlist` |
| Аноним | Не включать на ПДн |
| Probes | TCP 5601 в чарте = «порт открыт», не «`.kibana` зелёный» |

Боевые дашборды уметь накатить из **NDJSON**, иначе единственная копия — в кластере.

### Масштабирование (когда появятся цифры)

1. Больше людей → реплики OSD **в том же ЦОДе**.
2. Тяжелее Discover → **кластер** OpenSearch, не пятый Dashboards.
3. HPA только после профиля: иначе реплики вырастут из-за aggregation, который лечат шардами.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия OSD = 3.8.0 = кластеру.
2. Логин, index pattern. Убить один под: LB не 502, cookie принимается другой репликой.
3. `.kibana*` в snapshot, restore прогнан.
4. Учение «под OSD убит» vs «9200 недоступен»: второе — честно красный UI.
5. Нет demo cookie.password, нет `kibanaserver`/`kibanaserver`.

### Сильные / слабые стороны прод-схемы (OSD рядом с локальным кластером)

| Сильное | Слабое |
|---|---|
| Короткий путь до 9200 | Падение ЦОДа кластера = нет UI |
| Stateless горизонталь без Redis | Утечка cookie.password = подделка сессий |
| Saved objects в snapshot кластера | Реплики OSD в чужом ЦОДе без 9200 бесполезны |
| | Дефолт Helm Recreate = простой UI на каждом выкате, если забыли сменить |

**Не готов к проду**, если: одна реплика; Helm Recreate; `verificationMode: none`; дефолтный cookie.password; demo kibanaserver; 5601 опубликован; версии разъехались; OSD размазан «для HA» при кластере в одном ЦОДе; Wazuh и бизнес-поиск в одном UI; нет snapshot `.kibana*`.

---

## Источники

- Установка OSD: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/
- Docker demo yml: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/
- TLS, cookie.secure: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/tls/
- Оператор dashboards: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/operator/operator-dashboards-config/
- Multi-tenancy, `.kibana*`: https://docs.opensearch.org/latest/security/multi-tenancy/multi-tenancy-config/
- Правила: `OpenSearch Dashboards.md`
