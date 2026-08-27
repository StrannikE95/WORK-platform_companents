# MongoDB Community Server 7.0.40 — установка (учебный контур)

**Допущение:** закрытый стенд на машине разработчика, Docker, порт **27017** только на `127.0.0.1`. Лицензия сервера — **SSPL**; без решения юристов этот файл не разрешение ставить Community 7.0. Это не Atlas и не Enterprise.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Docker на машине разработчика (Linux или Docker Desktop). **Docker** — программа, которая запускает **образ** (упакованная программа с зависимостями) как контейнер.

Образ: `mongodb/mongodb-community-server:7.0.40-ubi9-slim` (тег есть на Docker Hub, linux/amd64 и linux/arm64). Не `library/mongo`, не `latest`, не `7.0-ubi9-slim`. Образы 5.0+ требуют **AVX** на CPU.

Kubernetes + оператор **MCK 1.10.0** и объект `MongoDBCommunity` (тип официально только `ReplicaSet`) — путь боя этой платформы, не этот файл.

| Зачем цифра | CPU | RAM | Диск |
|---|---|---|---|
| Минимум, чтобы процесс поднялся | «два реальных ядра **или** один многоядерный CPU» на `mongod` | Цифры «хватит N ГБ, чтобы стартовал» **в доке нет**. Нижняя граница кэша WiredTiger — **256 МБ**. В контейнере кэш **обязан** быть задан явно и **меньше** лимита контейнера: иначе процесс считает RAM **хоста** | Минимума ГБ **в доке нет** |
| Учебный ориентир | та же машина разработчика | `--wiredTigerCacheSizeGB 0.5` (512 МБ; в пределах 0.256–10000 ГБ). Сколько RAM выделить Docker Desktop сверх кэша — **в доке нет** | именованный том; без `-v` данные живут только пока жив контейнер. Официальная Docker-страница том **не** показывает; заводской `dbPath` Linux — `/data/db` |

Цифр «хватит N шардов / терабайтов» нет: нагрузки в запросе платформы не было. Шарды этим стендом не ставим; `MongoDBCommunity` их всё равно не описывает.

**Сильная сторона:** официальный Community-образ, минуты, совпадает с патчем **7.0.40**.  
**Слабая сторона:** один процесс — нет выборов главного, нет `w: majority`, нет change stream (нужен replica set).

**Критично:**

- **27017 в интернет / на все интерфейсы рабочей станции не публиковать.** Заводской пакет слушает localhost; Docker `-p 27017:27017` это обходит. Только `127.0.0.1`.
- NFS как единственный диск данных: формально POSIX допустим, вендор предупреждает — обычно **медленнее**. На стенде — локальный том.
- Не `latest`. Не прыгать на 7.0.41 из changelog, пока нет того же патча в Release Notes **и** в тегах образа.
- Живой replica set **не растягивать** на 2–3 дата-центра: порога RTT **в доке нет**; пульс 2 с, таймаут выборов 10 с. Голосующие члены — одна площадка.

---

## Установка для новичка

Официальная страница Docker-установки Community: https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/  
Там в примерах `latest` и `-p 27017:27017` без привязки к loopback — **не копировать как есть**. Ниже тот же путь с пином **7.0.40** и `127.0.0.1`.

### Что должно быть до установки

Есть:

- Docker; на хосте свободен **27017**.
- CPU с **AVX** (иначе образ 7.0 не стартует).
- Закрытая сеть; `mongosh` на хосте (консольный клиент; вендор ставит его **отдельно**, не как часть slim-образа в Docker-странице).

Нет:

- Публикации 27017 на `0.0.0.0` / Wi-Fi.
- Образа `library/mongo` как единственного ориентира.
- Kubernetes для этого стенда.

### Этап 1. Docker и `mongosh`

**Что делаем:** проверяем, что Docker запущен, и ставим `mongosh`, если его ещё нет.

```bash
docker version
```

`mongosh`: https://www.mongodb.com/docs/mongodb-shell/install/

Успех: `docker version` печатает клиент и сервер; `mongosh --version` отвечает.

### Этап 2. Скачать зафиксированный образ

**Что делаем:** забираем патч 7.0.40, не «что скачается по latest».

```bash
docker pull mongodb/mongodb-community-server:7.0.40-ubi9-slim
```

Успех: тег `7.0.40-ubi9-slim` есть локально. Список тегов: https://hub.docker.com/r/mongodb/mongodb-community-server/tags?name=7.0.40

### Этап 3. Запустить контейнер

**Что делаем:** один процесс `mongod` (СУБД) с проверкой прав сразу. `--auth` и размер кэша — флаги `mongod`; официальная Docker-страница разрешает дописывать их после имени образа. Том нужен, чтобы пользователи не исчезли после `docker rm` (саму `-v` Docker-страница не показывает).

```bash
docker run -d --name mongo-dev \
  -p 127.0.0.1:27017:27017 \
  -v mongo-dev-data:/data/db \
  mongodb/mongodb-community-server:7.0.40-ubi9-slim \
  --auth --wiredTigerCacheSizeGB 0.5
```

Успех: `docker container ls` — статус `Up`, порт `127.0.0.1:27017->27017/tcp`.

### Этап 4. Проверка «жив» и версия

**Что делаем:** с хоста подключаемся к loopback. Пока пользователей нет, срабатывает **localhost exception**: с localhost можно создать **первого** пользователя в `admin`, затем исключение закрывается.

```bash
mongosh --port 27017 --eval "db.runCommand({ hello: 1 })"
mongosh --port 27017 --eval "db.version()"
```

Успех: `ok: 1` (или `isWritablePrimary: true`) и версия **7.0.40**. Команда `hello` — со страницы Docker-установки.

### Этап 5. Администратор и пользователь приложения

**Что делаем:** заводим администратора пользователей, затем отдельную учётку приложения (не ходить в базу как админ). Пароли ниже — **только закрытый стенд**.

```bash
mongosh --port 27017 --eval '
db.getSiblingDB("admin").createUser({
  user: "standAdmin",
  pwd: "stand-admin-only",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
'
```

Дальше — уже с паролем админа:

```bash
mongosh --port 27017 -u standAdmin -p stand-admin-only --authenticationDatabase admin --eval '
db.getSiblingDB("app").createUser({
  user: "app",
  pwd: "stand-app-only",
  roles: [ { role: "readWrite", db: "app" } ]
})
'
```

Проверка записи/чтения учёткой приложения:

```bash
mongosh --port 27017 -u app -p stand-app-only --authenticationDatabase app app --eval '
db.foo.insertOne({ x: 1 });
db.foo.findOne();
'
```

Успех: пользователь `app` вставлен в `app.foo`, без роли документ не пишется. Процедура пользователей: https://www.mongodb.com/docs/v7.0/tutorial/configure-scram-client-authentication/ и https://www.mongodb.com/docs/v7.0/tutorial/create-users/

**Этот стенд не доказывает:** отказ машины или зала, выборы primary, `w: majority`, полную синхронизацию на другой диск, шарды, TLS, нагрузку. Change stream на одиночке **нет**.

---

## Первый запуск — URL, порт, учётка, смена пароля

HTTP-консоли у Community Server **нет**. Клиент — протокол **`mongodb://`** на TCP **27017**. GUI Compass — отдельная программа, не кластер.

| Что | Значение на этом стенде |
|---|---|
| Адрес | `127.0.0.1:27017` (не публиковать наружу) |
| Учётка по умолчанию | **Нет.** Пока нет `--auth` и пользователей, любой, кто дотянулся до порта, — полный доступ. Переменных `MONGO_INITDB_*` / `MONGODB_INITDB_*` у образа `mongodb-community-server` на Docker Hub **нет** (это `library/mongo` и Atlas Local — другой образ) |
| Учебные учётки | `standAdmin` / `stand-admin-only` (база аутентификации `admin`); `app` / `stand-app-only` (база `app`). **Только закрытый стенд** |
| URI приложения | `mongodb://app:stand-app-only@127.0.0.1:27017/app` |

Смена пароля (под `standAdmin`; без TLS пароль едет открытым текстом — на loopback стенда приемлемо, в бой нет):

```bash
mongosh --port 27017 -u standAdmin -p stand-admin-only --authenticationDatabase admin --eval '
db.getSiblingDB("app").runCommand({ updateUser: "app", pwd: "stand-app-new" })
'
```

В `mongosh` тот же смысл у помощника `db.changeUserPassword()` (страница https://www.mongodb.com/docs/v7.0/reference/command/updateUser/). В бою — свой секрет (Vault), не эти строки.

---

## Подключение к своей системе

| | |
|---|---|
| Протокол / порт | Драйвер MongoDB, TCP **27017**. Строка `mongodb://…`. В бою — URI **replica set** (`replicaSet=` или SRV), не один IP пода; писать с `w: majority` и ненулевым `wtimeout`. Балансировщик «как HTTP на все 27017» ломает смену главного |
| Кто клиент здесь | Микросервисы, workers Camunda, своё интеграционное API. Шина событий — **Kafka**, не Mongo. Кэш — **Redis/Valkey**, не Mongo. Озеро эталонных карточек за Mongo в исходном запросе **не закреплено** |
| В секрет, не в git | Пароль (и в бою — URI, keyfile/X.509 между членами, клиентский TLS). Учебные `stand-*` в репозиторий не класть |
| Это не | **PostgreSQL** (транзакционные карточки с внешними ключами); **Kafka** (лог событий); **Redis/Valkey** (кэш); **Enterprise / Atlas** (at-rest движком, аудит, LDAP — другой продукт и лицензия) |

Лимит одного документа BSON — **16 МБ**. Нативный пакет без Docker слушает localhost; не открывать 27017 «для удобства» в офисную сеть.

---

## Ссылки на материал

| Факт | URL / файл |
|---|---|
| Патч 7.0.40 (11 Aug 2026) | https://www.mongodb.com/docs/v7.0/release-notes/7.0/ |
| Тег образа `7.0.40-ubi9-slim` | https://hub.docker.com/r/mongodb/mongodb-community-server/tags?name=7.0.40 |
| Docker Community: pull / run / `hello` / AVX | https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/ |
| Порт 27017 (27018 шард, 27019 config) | https://www.mongodb.com/docs/v7.0/reference/default-mongodb-port/ |
| CPU min; кэш; NFS; XFS; THP; права выкл. по умолчанию | https://www.mongodb.com/docs/v7.0/administration/production-notes/ |
| Формула кэша WiredTiger, контейнер ≠ RAM хоста | https://www.mongodb.com/docs/v7.0/core/wiredtiger/ |
| `dbPath` Linux `/data/db`; `--auth` | https://www.mongodb.com/docs/v7.0/reference/program/mongod/ |
| Bind localhost vs Docker map | https://www.mongodb.com/docs/v7.0/core/security-mongodb-configuration/ |
| SCRAM, первый пользователь, `--auth` | https://www.mongodb.com/docs/v7.0/tutorial/configure-scram-client-authentication/ |
| Localhost exception | https://www.mongodb.com/docs/v7.0/core/localhost-exception/ |
| Пользователь приложения, `insertOne` | https://www.mongodb.com/docs/v7.0/tutorial/create-users/ |
| Смена пароля (`updateUser`) | https://www.mongodb.com/docs/v7.0/reference/command/updateUser/ |
| URI `mongodb://` | https://www.mongodb.com/docs/v7.0/reference/connection-string/ |
| Установка `mongosh` | https://www.mongodb.com/docs/mongodb-shell/install/ |
| `MongoDBCommunity` только `ReplicaSet` | https://www.mongodb.com/docs/kubernetes/current/reference/k8s-operator-community-specification/ |
| Установка оператора MCK, манифесты 1.10.0 | https://www.mongodb.com/docs/kubernetes/current/tutorial/install-k8s-operator/ · https://github.com/mongodb/mongodb-kubernetes/tree/1.10.0/public |
| MCK 1.10.0 (30 Jul 2026) | https://www.mongodb.com/docs/kubernetes/current/release-notes/ |
| EOL линии 7.0: 31 Aug 2027 | https://www.mongodb.com/legal/support-policy/lifecycles |
| Зачем продукт, порты, роль в платформе | `MongoDB 7.md` |
| Словарь | `MongoDB 7.info.md` |
| Стыковка с Kafka / Camunda / ЦОД | `MongoDB 7.shema.md` |
| Карточка консультанта | `MongoDB 7.consultant.md` |

**В доке вендора этого нет** (не выдумано): минимум RAM/диска в ГБ «чтобы `mongod` стартовал»; порог RTT для растяжки replica set; заводская учётка/пароль Community; env `MONGO_INITDB_*` у образа `mongodb-community-server`; том `-v` на Docker-странице Community.
