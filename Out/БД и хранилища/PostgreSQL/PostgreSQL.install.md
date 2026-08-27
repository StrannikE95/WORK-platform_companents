# PostgreSQL 18.6 — установка (учебный контур)

Допущение: закрытый стенд на Linux-машине разработчика, один контейнер Docker, порт 5432 только на `127.0.0.1`. Это не бой и не отказоустойчивость.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Community PostgreSQL **18.6** (релиз 13 августа 2026) — **контейнер Docker** на Linux-машине разработчика. **Docker** — программа, которая запускает готовый **образ** (упакованная программа с зависимостями) как **контейнер**. Образ: `postgres:18.6` (теги `18` и `latest` на эту дату тоже указывают на 18.6 — **не** используйте их: ярлык завтра сдвинется).

Альтернатива учёбы: однонодовый Kubernetes + оператор **CloudNativePG 1.30.0**. Тогда образ инстанса — `ghcr.io/cloudnative-pg/postgresql` линии **18.6**, не `postgres:18.6` (оператор подменяет точку входа своим instance manager). Завод оператора 1.30.0 — **18.4**; его не оставлять. Windows-хост под CNPG не предполагается.

Карточки клиентов, метаданные Grafana и Camunda Identity — **разные** кластеры, даже на учёбе не склеивать в одну базу «на всех».

**Сколько железа.** Цифр «хватит N ядер / M ГБ» в мануале PostgreSQL **нет** (страница Requirements описывает компилятор и библиотеки для сборки из исходников, не смету VM). Не путать «процесс жив» со сметой боя.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум, чтобы процесс поднялся | **в доке вендора нет** | Заводской кэш страниц `shared_buffers` обычно **128 МБ** (может быть меньше, если ядро не даст — решает `initdb`) | Каталог данных `PGDATA` + журнал WAL; размера «хватит учебному стенду» **в доке нет** | [19.4 Resource Consumption](https://www.postgresql.org/docs/18/runtime-config-resource.html), [17.1 Requirements](https://www.postgresql.org/docs/18/install-requirements.html) |
| Учебный ориентир | 1 vCPU достаточно, чтобы контейнер отвечал на `SELECT 1` | Несколько сотен МБ контейнеру; потолок сессий `max_connections` обычно **100** | Именованный том Docker на `/var/lib/postgresql` (в 18 путь данных — `/var/lib/postgresql/18/docker`) | `max_connections`: [19.3 Connections](https://www.postgresql.org/docs/18/runtime-config-connection.html). Путь `PGDATA`: [Docker Hub postgres](https://hub.docker.com/_/postgres) |
| `/dev/shm` в Docker | — | У контейнера завод Docker часто **64 МБ** shm; при ошибке `could not resize shared memory segment` образ советует `--shm-size=256MB` | — | [docker-library/docs postgres](https://github.com/docker-library/docs/blob/master/postgres/README.md) |

Нагрузки платформы нет — **нет** фразы «хватит для терабайтов». `shared_buffers` для боя поднимают **после замера**, не «25% с потолка». Порог задержки сети (RTT) для синхронного `COMMIT` **в доке PostgreSQL нет** — поэтому живой кластер на несколько дата-центров здесь не собираем.

**Сильная сторона:** официальный образ, минуты, совпадает с практикой разработки.  
**Слабая сторона:** один процесс; падение машины = нет базы; успешный `INSERT` не доказывает смену главной и sync.

**Критично:**

- Порт **5432/TCP** не публиковать в интернет и не на все интерфейсы хоста (`-p 5432:5432` без `127.0.0.1` часто слушает Wi-Fi).
- `POSTGRES_HOST_AUTH_METHOD=trust` **запрещён** даже на стенде: метод `trust` пускает любого, кто дошёл до порта, с любым именем роли, включая суперпользователя.
- Не `latest`, не прыжок через minor.
- Каталог данных — локальный диск, не NFS как единственный диск главной (CNPG: лучше local; репликация СХД не заменяет копию PostgreSQL).
- Один `Cluster` не растягивать на 2–3 дата-центра. Автосмена главной между разными Kubernetes у оператора **не в scope**.

---

## Установка для новичка

Официальная страница образа: https://hub.docker.com/_/postgres  
Документация линейки 18: https://www.postgresql.org/docs/18/  
Установка из исходников (если когда-нибудь без Docker): https://www.postgresql.org/docs/18/tutorial-install.html → глава 17.

### Что должно быть до установки

Есть:

- Docker на Linux (или Docker Desktop с Linux-VM; команды ниже — bash, не PowerShell).
- На localhost свободен TCP **5432**.
- Закрытая сеть; стенд не торчит в интернет.

Нет (и не должно появиться):

- `-p 5432:5432` без `127.0.0.1`.
- `POSTGRES_HOST_AUTH_METHOD=trust`.
- Тег `latest` / `18` вместо `18.6`.
- Учётки приложения = роль `postgres`.

### Этап 1. Docker на месте

**Что делаем:** проверяем, что программа Docker запущена и умеет скачивать образы.

```bash
docker version
```

Успех: клиент и сервер Docker отвечают версией, без `Cannot connect to the Docker daemon`.

### Этап 2. Каталог данных на диске хоста

**Что делаем:** заводим именованный том, чтобы файлы базы (`PGDATA`) переживали удаление контейнера. В PostgreSQL 18 образ кладёт данные в `/var/lib/postgresql/18/docker`; том монтируют на **`/var/lib/postgresql`**, не на старый путь `/var/lib/postgresql/data`.

```bash
docker volume create pg-dev-data
```

Успех: `docker volume inspect pg-dev-data` показывает том.

### Этап 3. Запускаем 18.6

**Что делаем:** поднимаем один процесс `postgres`. `POSTGRES_PASSWORD` — **обязателен** (пароль суперпользователя `postgres`; заводского пароля у образа нет). `POSTGRES_DB=app` создаёт базу `app` при первом `initdb` (первичное создание пустого каталога данных). Пароль ниже — **только закрытый стенд**, не секрет и не в git.

```bash
docker run -d --name pg-dev \
  --shm-size=256mb \
  -e POSTGRES_PASSWORD=dev \
  -e POSTGRES_DB=app \
  -p 127.0.0.1:5432:5432 \
  -v pg-dev-data:/var/lib/postgresql \
  postgres:18.6
```

Успех: `docker ps` показывает `pg-dev` в состоянии running. Привязка к `127.0.0.1` обязательна.

Внутри контейнера образ слушает шире localhost (иначе другие контейнеры не подключились бы). Снаружи хоста порт должен быть только loopback.

### Этап 4. Версия 18.6

**Что делаем:** `psql` — консольный клиент SQL. Изнутри контейнера к Unix-сокету образ пускает роль `postgres` без пароля (`trust` **локально**). С другого хоста пароль нужен. Функция `version()` печатает строку версии сервера.

```bash
docker exec -it pg-dev psql -U postgres -c 'SELECT version();'
```

Успех: в строке есть **PostgreSQL 18.6**. Если другая патч-версия — образ не тот.

### Этап 5. Роль приложения, не `postgres`

**Что делаем:** суперпользователь (`postgres`) может читать файлы ОС и запускать программы из сессии — приложению его не дают. Роль — учётка внутри PostgreSQL. `LOGIN` разрешает войти. `GRANT CONNECT` — право открыть сессию к базе. С PostgreSQL 15 у всех больше нет `CREATE` на схеме `public` в новых базах — без `GRANT CREATE` роль `app` не создаст таблицы.

```bash
docker exec -it pg-dev psql -U postgres -d app -c "
CREATE ROLE app LOGIN PASSWORD 'dev-app';
GRANT CONNECT ON DATABASE app TO app;
GRANT CREATE, USAGE ON SCHEMA public TO app;
"
```

Успех: команды без ошибки. Проверка входом роли `app` (пароль `dev-app` — **стенд-only**):

```bash
docker exec -it pg-dev psql -U app -d app -c 'SELECT current_user;'
```

Должно напечатать `app`. Суперпользователь из приложения — отказ (так и должно быть).

### Этап 6. Данные на томе

**Что делаем:** убеждаемся, что каталог на диске живой, а не «только память контейнера».

```bash
docker exec -it pg-dev psql -U app -d app -c "CREATE TABLE ping(id int); INSERT INTO ping VALUES (1);"
docker restart pg-dev
docker exec -it pg-dev psql -U app -d app -c "SELECT * FROM ping;"
```

Успех: после рестарта строка `1` на месте.

### Чего этот стенд не доказывает

Отказ машины или зала, смену главной, лаг копии, синхронный `COMMIT`, точку восстановления из архива WAL, TLS, нагрузку, выборы лидера (их у самого `postgres` нет — нужен оператор). Один контейнер официально нормален для разработки; в бой его не копируют.

Однонодовый Kubernetes (не основной путь): манифест оператора `cnpg-1.30.0.yaml` из ветки `release-1.30`, затем объект `Cluster` с `instances: 1` и явно пиненым образом линии **18.6**. Сервис записи: `*-rw:5432`. Этот YAML в бой не копировать. Страница установки: https://cloudnative-pg.io/docs/1.30/installation_upgrade/ (URL `…/documentation/1.30/` из карточки консультанта на проверке дал **404**).

---

## Первый запуск — URL, порт, учётка, смена пароля

| Что | Учебный стенд |
|---|---|
| Хост | `127.0.0.1` / `localhost` |
| Порт | **5432/TCP** (завод PostgreSQL; TLS на том же порту, если `ssl=on`; завод `ssl` — **off**) |
| База | `app` |
| Стартовая роль образа | суперпользователь `postgres`, пароль из `POSTGRES_PASSWORD` (`dev` ниже — **стенд-only**) |
| Роль приложения | `app` / `dev-app` (**стенд-only**) |
| URI (libpq / psql) | `postgresql://app:dev-app@127.0.0.1:5432/app` |

Подключение с хоста (если `psql` установлен локально):

```bash
psql "postgresql://app:dev-app@127.0.0.1:5432/app"
```

**Смена пароля.** Обычная роль может сменить свой пароль. `ALTER ROLE … PASSWORD` передаёт пароль открытым текстом в SQL и может попасть в историю `psql`; предпочтительно метакоманда `\password` (клиент шифрует обмен).

```bash
docker exec -it pg-dev psql -U app -d app
```

Внутри `psql`:

```text
\password
```

Либо от суперпользователя:

```sql
ALTER ROLE app PASSWORD 'новый-стендовый';
```

Пароли стенда в бой не копируют. В бою — свой секрет в Vault (не git), TLS + SCRAM-SHA-256, список `pg_hba` только со своих сетей. Завод хранения паролей с версии 14 — `password_encryption = scram-sha-256`. **MD5 в 18 устарел.** Метода `trust` на TCP с не-loopback не будет.

Kubernetes учёбы: хост сервиса записи `pg-dev-rw`, порт 5432, роль-владелец из `bootstrap.initdb` (в примере оператора — база `app`, владелец `app`).

---

## Подключение к своей системе

**Протокол и порт.** Клиент открывает **TCP 5432** драйвером PostgreSQL. Отдельного «порта кластера» нет: и клиенты, и потоковая репликация ходят на 5432. UDP нет.

**Кто клиент в этой платформе.** Микросервисы и интеграционное API (истина карточки — `COMMIT` в PostgreSQL). Job workers Camunda — состояние процесса, которое не в Zeebe. Grafana / Superset — **свой** кластер, не эталон клиентов. После `COMMIT` факт «карточка изменилась» можно отдать в Kafka (таблица outbox в той же транзакции). Писать только на главную (в CNPG — DNS `*-rw`). Читать с копии (`*-ro`) — только если сервис переживает отставание.

**Что класть в секрет (не git):** хост, порт, имя базы, роль **не** `postgres`, пароль. В бою — ещё CA и `sslmode=verify-full`. Строка JDBC: `jdbc:postgresql://127.0.0.1:5432/app` + user/password. Python: `psycopg.connect("postgresql://app@127.0.0.1:5432/app")` (пароль из окружения/Vault, не из репозитория). .NET: npgsql, тот же хост/порт/база.

**Чем продукт не является**

| Сосед | Чем отличается |
|---|---|
| ClickHouse | Отчёты и сканы больших таблиц, не транзакционная карточка |
| MongoDB | Документы; роль озера в этой платформе за Mongo не закреплена |
| Redis / Valkey | Кэш и локи, не истина |
| Kafka | Лог событий, не `COMMIT` строки |

PostgreSQL — не шина, не аналитический склад, не файловое озеро (вложения — объектное хранилище). Горизонтали записи нет: ещё копия **не** ускоряет `INSERT`. Community 18.6 — не Postgres Pro и не EDB.

---

## Ссылки на материал

| Факт | Страница |
|---|---|
| Документация PostgreSQL **18.6** | https://www.postgresql.org/docs/18/ |
| Релиз 18.6 (13 авг 2026; пропуск 18.5; 28 CVE) | https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/ |
| Установка (tutorial → глава 17) | https://www.postgresql.org/docs/18/tutorial-install.html |
| Requirements (сборка; **нет** CPU/RAM стенда) | https://www.postgresql.org/docs/18/install-requirements.html |
| Порт 5432, `listen_addresses`, `max_connections` | https://www.postgresql.org/docs/18/runtime-config-connection.html |
| `shared_buffers` (обычно 128 МБ) | https://www.postgresql.org/docs/18/runtime-config-resource.html |
| `ssl` завод **off** | https://www.postgresql.org/docs/18/runtime-config-connection.html#GUC-SSL |
| `password_encryption` = `scram-sha-256`; MD5 deprecated | https://www.postgresql.org/docs/18/auth-password.html |
| Метод `trust` | https://www.postgresql.org/docs/18/auth-trust.html |
| `CREATE ROLE` / `ALTER ROLE` / `GRANT` | https://www.postgresql.org/docs/18/sql-createrole.html · https://www.postgresql.org/docs/18/sql-alterrole.html · https://www.postgresql.org/docs/18/sql-grant.html |
| Схема `public`, `CREATE` с PG 15 | https://www.postgresql.org/docs/18/ddl-schemas.html |
| `version()` | https://www.postgresql.org/docs/18/functions-info.html |
| URI `postgresql://` | https://www.postgresql.org/docs/18/libpq-connect.html#LIBPQ-CONNSTRING-URIS |
| Образ `postgres:18.6`, `POSTGRES_*`, `trust`, `PGDATA` 18 | https://hub.docker.com/_/postgres |
| Полный README образа (shm 64 МБ / `--shm-size`) | https://github.com/docker-library/docs/blob/master/postgres/README.md |
| JDBC URL | https://jdbc.postgresql.org/documentation/use/ |
| psycopg `connect` | https://www.psycopg.org/psycopg3/docs/basic/usage.html |
| CNPG 1.30 (рабочий префикс `/docs/`; `/documentation/1.30/` — 404) | https://cloudnative-pg.io/docs/1.30/ |
| Установка оператора 1.30.0 | https://cloudnative-pg.io/docs/1.30/installation_upgrade/ |
| Матрица k8s **1.36 / 1.35 / 1.34**, PG 14–18, завод **18.4** | https://cloudnative-pg.io/docs/1.30/supported_releases/ · https://github.com/cloudnative-pg/cloudnative-pg/releases/tag/v1.30.0 |
| Диск: local vs NFS/СХД | https://cloudnative-pg.io/docs/1.30/storage/ |
| Автосмена только внутри одного Kubernetes | https://cloudnative-pg.io/docs/1.30/architecture/ |
| `initdb`: база `app`, владелец `app` | https://cloudnative-pg.io/docs/1.30/bootstrap/ |
| Образы оператора (не `postgres:18.6`) | https://cloudnative-pg.io/docs/1.30/container_images/ |
| Правила платформы, порты, роль | `PostgreSQL.md` |
| Словарь | `PostgreSQL.info.md` |
| Стыковка с Kafka / Camunda / ClickHouse | `PostgreSQL.shema.md` |

**В доке вендора нет (не угадано):** сколько CPU/RAM «чтобы процесс поднялся» и «хватит учебной нагрузке»; порог RTT для stretch-кластера; гарантированный размер диска стенда; пароль по умолчанию (его нет — Docker требует `POSTGRES_PASSWORD`).
