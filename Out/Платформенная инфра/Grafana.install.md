# Grafana 13.2.0 — установка и конфигурирование

Связанный документ (зачем система, из каких программ состоит, порты, железо): `Grafana.md`.

Этот файл — **как поставить и настроить**. Настройки с учебной машины в бой не копируйте.

## Что вы ставите

Grafana — платформа дашбордов и алертов. Она **рисует** то, что лежит в Prometheus (и других источниках), сама ряды не хранит.

Версия в этой инструкции: **Grafana OSS 13.2.0**, образ `grafana/grafana:13.2.0`. На Kubernetes — Helm **`grafana-community/grafana` 12.11.2** (`appVersion` 13.2.0). Репозиторий: `https://grafana-community.github.io/helm-charts`.  
Документация: https://grafana.com/docs/grafana/latest/  
HA: https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/

Один логический Grafana (одна БД + одни `ha_peers`), размазанный на несколько дата-центров, здесь **не собираем**. Синхронный Postgres и gossip **9094 TCP+UDP** чувствительны к задержке сети; в документации Grafana **нет** цифры «сколько миллисекунд ещё можно». Поэтому живые реплики и их БД — внутри одной площадки. Вторая площадка — свой экземпляр с тем же git **или** люди ходят в UI первой.

Windows как сервер Grafana в схеме с Kubernetes не ставим. На учёбе достаточно Docker.

---

## О чём эта инструкция молча договорилась

1. **Один живой «мозг» в одном дата-центре.** Реплики Grafana, общая PostgreSQL и gossip 9094 живут **в одной площадке**. Если растянуть БД и 9094 на несколько городов, порога задержки нет — поэтому здесь так не делаем.
2. Бой — Kubernetes в каждом дата-центре отдельно (см. `Kubernetes.install.md`).
3. Учебный стенд — одна машина в закрытой сети. SQLite допустим **только там**.
4. Цифр вашей нагрузки нет. Ориентир вендора: минимум 1 ядро / 512 МиБ; Small — 2 ядра / 2–4 ГиБ / 10–20 ГБ SSD; Medium — 4–8 ядер / 8–16 ГиБ и **2** инстанса; Large — 8–16+ ядер / 16–32+ ГиБ и **3+**. Это не смета вашего железа.
5. Общая БД в бою — **PostgreSQL 12+** (отдельная база `grafana`). HA этой БД — `PostgreSQL.install.md`, не этот файл.
6. Дашборды в бою — **provisioning из git**, не только клики в UI. Иначе две площадки = две правды.
7. Два дата-центра: активный Grafana в первом; второй — доступ к UI первого **или** независимый экземпляр (своя БД, тот же git). Три — то же. Третья площадка **не** второй writer в ту же БД.
8. Grafana **не ходит** в государственные сервисы. Цифры она берёт из Prometheus/Mimir, которые сняли ваши экспортёры.

---

## Учебный стенд: одна площадка, без нагрузки

**Зачем:** открыть UI, подключить Prometheus, понять Explore и алерт. **Не зачем:** доказывать отказ дата-центра и 200 зрителей.

### Что должно быть до установки

- Docker Engine **или** однонодовый Kubernetes.
- На localhost свободен порт 3000.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь для учёбы)

```bash
docker run -d --name grafana-dev \
  -p 127.0.0.1:3000:3000 \
  grafana/grafana:13.2.0
```

Привязка к `127.0.0.1` обязательна: `-p 3000:3000` без адреса часто слушает все интерфейсы.

Первый вход: `admin` / `admin`, вендор просит сменить пароль. Данные — SQLite внутри контейнера. Удалили контейнер без volume — удалили дашборды. Пароль **не** оставляйте `admin` / `admin` как привычку — в жизнь это копировать нельзя.

Проверка:

```bash
curl -s http://127.0.0.1:3000/api/health
docker exec grafana-dev grafana cli --version
```

В ответе health — JSON со статусом БД. Версия процесса — **13.2.0**, не `latest`.

Сохраните JSON дашборда в git сразу: без volume и без provisioning стенд не переживает пересоздание.

### Установка (Kubernetes, если стенд уже в кластере)

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
helm install grafana-dev grafana-community/grafana --version 12.11.2 \
  --set image.tag=13.2.0 \
  --set replicas=1
```

`replicas: 1`, SQLite. **Не** этот values в бой. Port-forward только на localhost.

### Какие настройки на тесте упрощаем

| Параметр | На тесте | Зачем так |
|---|---|---|
| Один процесс | да | Не учимся переживать выкат |
| БД | SQLite | Нет требования пережить падение пода |
| `ha_peers` | не нужен | Один Alertmanager |
| TLS | нет, сеть закрыта | Иначе сертификаты раньше PromQL |
| Image Renderer / Live Redis | выкл | Не цель |
| reporting / gravatar / snapshots | можно оставить на изолированном стенде | В чек-лист боя не переносить |

Чего **не** упрощаем: тег **13.2.0**; хотя бы один источник и панель, которая не пустая; дашборд в git; 3000 не с мира; unsigned plugins нет; пароль сменить с заводского `admin`.

### Как понять, что стенд живой

1. `/api/health` отвечает, UI открывается с localhost.
2. Источник (например Prometheus) в Explore живой, панель обновилась.
3. Рестарт контейнера без volume: дашборды пропали — это ожидаемо, не баг стенда.

### Что хорошо и что плохо в такой схеме

| Хорошо | Плохо |
|---|---|
| Минуты, официальный образ | Нет failover, нет общей БД, нет gossip |
| Совпадает с quickstart Grafana | Успешный график **не** доказывает бой |
| | Открытый 3000 на Wi-Fi = дыра; привычка `admin`/`admin` уедет в бой |

Перед боем полезен **препрод**: Postgres, 2 реплики, `ha_peers`, свои пароли — всё **в одном** дата-центре.

---

## Бой: один живой дата-центр, второй — запас или свой UI

**Зачем:** пережить отказ **одной машины или пода внутри площадки** (вторая реплика, общая Postgres, gossip чтобы письма не дублировались). Отказ **всего дата-центра, где лежит Postgres Grafana** = нет UI и нет встроенных алертов, пока вручную не поднимете запасную копию БД и не переключите адрес. Терабайты рядов хранит Prometheus/Mimir, **не** Postgres Grafana.

### Почему кластер не растягиваем на несколько дата-центров

HA Grafana = несколько процессов + **одна** Postgres + gossip **9094 TCP и UDP**. Commit в Postgres не быстрее сети до запасной копии; Memberlist не рассчитан на WAN. Закрытый UDP между городами = дубли алертов. Документация Grafana **не задаёт** порог миллисекунд.

### Как расставить в активном дата-центре

Один логический Grafana:

- Deployment **≥ 2** реплик (для Medium вендор пишет 2, для Large 3+) на **разных нодах** одного дата-центра.
- **Не** SQLite: внешний PostgreSQL **этой** площадки.
- `headlessService: true`; в `grafana.ini`: `ha_listen_address` / `ha_advertise_address` = `${POD_IP}:9094`, `ha_peers` = headless:9094.
- Дашборды, источники, правила алертов — **provisioning из git**. Клики в UI будут перезатёрты — выбрать один источник правды.
- Перед 3000 — HTTPS LB (HAProxy/Ingress). Sticky **не** обязателен.
- Образ **13.2.0**, чарт **12.11.2**, не `latest`.
- `persistence` чарта для sqlite **выкл**.
- PDB: не убивать все реплики сразу.

**Между площадками:**

| Сколько площадок | Что где | Если умер активный дата-центр |
|---|---|---|
| **Две** | Первая: Grafana HA + Postgres Grafana. Вторая: люди ходят в UI первой **или** независимый Grafana (своя БД, тот же git), **без** `ha_peers` и без общей БД через город | Нет UI первой, пока DR/второй экземпляр. Встроенные алерты молчат вместе с этим Grafana |
| **Три** | Третья как вторая | То же. Третий дата-центр не даёт второго writer в ту же БД |

Метрики с других площадок — **удалённый Prometheus / federation** (см. `Prometheus.install.md`), не второй Grafana «как правда».

Не растягивать: один Postgres Grafana на 2–3 Kubernetes; gossip 9094 между дата-центрами; «три реплики Grafana в трёх ЦОДах, одна SQLite».

### Что должно быть до боевой установки

- Kubernetes активной площадки.
- HA Postgres для базы `grafana` **внутри этой площадки**; WAL archive для DR. Бакет/архив должен **переживать** гибель этой площадки.
- NetworkPolicy: 3000 только от LB; 9094 только между подами Grafana **этого** дата-центра; 5432 только Grafana → Postgres.
- Секреты (`secret_key`, пароль БД, SSO) — в Vault. Свой `secret_key` **до** заведения источников. Дефолт `SW2YcwTIb9zpOOhoPsMm` известен всем.
- NTP на всех машинах.

### Порядок установки в активном дата-центре

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm upgrade --install grafana grafana-community/grafana --version 12.11.2 \
  --namespace monitoring --create-namespace \
  -f grafana-prod-values.yaml
```

Смысл values (не полный файл — сверять с README чарта 12.11.2):

```yaml
image:
  repository: grafana/grafana
  tag: "13.2.0"
replicas: 2
headlessService: true
persistence:
  enabled: false
grafana.ini:
  database:
    type: postgres
    host: pg-grafana-rw:5432
    name: grafana
    ssl_mode: verify-full
  unified_alerting:
    ha_peers: grafana-headless:9094
  security:
    admin_user: break-glass
    cookie_secure: true
  analytics:
    reporting_enabled: false
    check_for_updates: false
  snapshots:
    external_enabled: false
  server:
    root_url: https://grafana.example.internal/
```

Пароль admin и `secret_key` — Secret, не `admin`/`admin`. Оператор Grafana 5.25.0 — допустимая альтернатива с теми же правилами (общая БД + `ha_peers` **внутри** площадки).

Дальше:

1. Выключить опасные заводские значения: reporting, gravatar, внешние snapshots, anonymous; `/metrics` с basic auth.
2. Подключить источник (Prometheus). Provisioning 2–3 дашбордов. Контролируемый алерт доехал.
3. SSO + RBAC. Убрать ежедневный вход под Admin.
4. Бэкап **ядра** Postgres Grafana: дашборды, пользователи, состояние алертов, зашифрованные пароли источников. Git дашбордов **не** заменяет backup ролей и алертов.

### Правила конфигурации боя

| Слой | Правило |
|---|---|
| Сеть | 443 — из сети единого входа; 9094, 5432 — не из пользовательской VLAN |
| Gossip | в тот же день, когда `replicas > 1`. Метрика `grafana_alertmanager_cluster_members` |
| Оценка алертов | каждый под считает **все** правила → нагрузка на Prometheus × N. Третью реплику «для красоты» без спроса к источнику не ставить |
| Критичные алерты без Grafana | дублировать во **внешнем** Alertmanager Prometheus |
| `ssl_mode` Postgres | `verify-full`, не `disable` |
| Image Renderer | если нужен — свой токен ≠ `-`, NetworkPolicy только от Grafana. Иначе не включать |
| Плагины | только подписанные, из внутреннего registry |
| SELinux / мандатный доступ | не выключать «чтобы установилось» без решения ИБ |

Независимый Grafana на второй площадке — отдельный Helm, **своя** Postgres, тот же git. Не `ha_peers` на сервис чужого дата-центра.

### Как расти, когда появятся цифры нагрузки

1. Замерить зрителей, число дашбордов, refresh, число alert rules.
2. Больше людей → реплики Grafana + CPU/RAM **внутри** площадки. Не умножать оценку алертов бездумно.
3. Терабайты рядов → масштаб Prometheus/Mimir, не `replicas` Grafana.
4. Тиры вендора (Small/Medium/Large) — стартовые точки, не смета. При >1000 правил или interval < 1 мин — отдельная схема оценки, не «ещё один под».

### Проверки, без которых это ещё не бой

1. UI **13.2.0**, `/api/health`, запись в Postgres (созданный folder жив после рестарта пода).
2. `grafana_alertmanager_cluster_members` = числу реплик **этой** площадки.
3. Убить под Grafana: LB жив, UI жив, алерт не задублировался.
4. Контролируемый алерт дошёл до contact point.
5. Provisioning из git перезатирает клик в UI (если git — правда).
6. Учение «активный дата-центр выключен»: либо UI второй площадки с git поднялся, либо честный простой. Restore Postgres Grafana на стенде.
7. Нет `admin`/`admin`, нет дефолтного `secret_key`.

### Что хорошо и что плохо в схеме «мозг в одном дата-центре»

| Хорошо | Плохо |
|---|---|
| Gossip и Postgres не зависят от межгородского RTT | Падение этой площадки = нет этого Grafana, пока не поднимете запас |
| Один набор дашбордов через git | Два независимых Grafana без git = две правды алертов |
| Официальная схема HA внутри площадки | Admin Grafana = ключ к паролям источников |
| | Клиентский 3000 из другого города идёт через сеть — TLS ваш |

**Не готово к бою**, если: одна реплика с SQLite; три реплики без `ha_peers`; `admin`/`admin` или дефолтный `secret_key`; `latest`; Grafana в интернет; reporting/snapshots наружу; дашборды только в UI без git и без бэкапа Postgres; stretch БД/`ha_peers` на 2–3 ЦОДа; учебный манифест скопирован в бой.

---

## Откуда цифры и имена образов

- Релиз 13.2.0: https://github.com/grafana/grafana/releases/tag/v13.2.0
- Установка, SQLite vs Postgres 12+: https://grafana.com/docs/grafana/latest/setup-grafana/installation/
- HA: https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/
- Alerting HA, 9094 TCP+UDP: https://grafana.com/docs/grafana/latest/alerting/set-up/configure-high-availability/
- Helm 12.11.2: https://github.com/grafana-community/helm-charts/releases/tag/grafana-12.11.2
- Правила и схема компонентов: `Grafana.md`
