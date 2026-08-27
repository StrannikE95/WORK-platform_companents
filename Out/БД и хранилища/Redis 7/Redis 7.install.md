# Redis Community Edition 7.4.11 — установка (учебный контур)

**Допущение:** один контейнер Docker на этой машине, порт **6379** только на `127.0.0.1`. Sentinel, Cluster и бой сюда не входят.

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Linux-машина (или Docker Desktop с **Linux-контейнерами**) и официальный образ **`redis:7.4.11`**. Не `redis:7.4`, не `redis:7`, не `latest` (`latest` на дату проверки указывает на **8.10.1**). Kubernetes и community-оператор на учёбе не нужны.

**Сколько железа.** В мануале **Community 7.4 нет** таблицы «N ядер / N ГБ, чтобы процесс поднялся». Цифры 4 ГБ / 8 ядер и т.п. относятся к **Redis Software** (другой продукт) — их сюда не переносить. Порог RTT для растяжки набора на несколько дата-центров в доке Community **тоже нет**.

| Зачем цифра | CPU | RAM | Диск |
|---|---|---|---|
| Минимум, чтобы процесс поднялся | **в доке нет** | **в доке нет**; ключи живут в RAM | **в доке нет**; заводской AOF выключен (`appendonly no`) |
| Учебный ориентир | 1 ядро хватит: команды исполняются в основном на **одном** потоке | Задайте свой `maxmemory` под игрушечный набор. Без него Redis ест свободную RAM, пока его не убьют. «Хватит 256 МБ» в доке вендора **нет** | Том не обязателен: рестарт без тома = пустой кэш. Если том — локальный каталог `/data`, не NFS |

**Сильная сторона:** официальный образ, минуты до `PONG`.  
**Слабая сторона:** один процесс; успешный `SET` не доказывает смену master, лаг replica и вытеснение.

**Критично:**

- **Не публиковать 6379 в интернет.** В образе protected mode **выключен**: `-p 6379:6379` без `127.0.0.1` открывает инстанс **без пароля** всем, кто достучится до хоста.
- Не `latest`. Не NFS/NAS под RDB/AOF (бенчмарки OSS: бьёт задержку и канал). Не растягивать Sentinel/Cluster на несколько дата-центров.

## Установка для новичка

Официальная страница Docker: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/  
Образ и предупреждение про порт: https://hub.docker.com/_/redis

Пример вендора слушает **все** интерфейсы и не пинит патч. Ниже — тот же путь, но тег **7.4.11** и bind на loopback.

### Что должно быть до установки

Есть:

- Docker (программа, которая запускает **образ** — упакованную программу с зависимостями — как **контейнер**). На Windows — Linux-контейнеры.
- Свободен порт **6379** на localhost.
- Сеть стенда не торчит в интернет.

Нет (и не должно появиться):

- `-p 6379:6379` без `127.0.0.1`.
- Тег `latest` / `7` / `7.4`.
- Учебный пароль в git и в бой.

### Этап 1. Проверяем Docker

**Что делаем:** убеждаемся, что демон Docker отвечает.

```bash
docker version
```

Успех: печатается версия клиента и сервера, без `Cannot connect to the Docker daemon`.

### Этап 2. Запускаем Redis 7.4.11

**Что делаем:** скачиваем образ (если его ещё нет) и стартуем один процесс. `-p 127.0.0.1:6379:6379` пробрасывает порт контейнера только на loopback **этой** машины (127.0.0.1 — «я сам», не LAN и не интернет).

```bash
docker run -d --name redis-dev \
  -p 127.0.0.1:6379:6379 \
  redis:7.4.11
```

Успех: `docker ps` показывает контейнер `redis-dev` в статусе Up.

### Этап 3. PING и версия

**Что делаем:** из того же образа запускаем `redis-cli` (утилита: говорит с сервером командами Redis) и проверяем, что это именно **7.4.11**.

```bash
docker exec -it redis-dev redis-cli PING
docker exec -it redis-dev redis-cli INFO server
```

Успех: первая команда отвечает `PONG`. Во второй строке `redis_version:7.4.11`.

### Этап 4. Пользователь приложения (ACL)

**Что делаем:** заводской пользователь `default` — без пароля и с правом на все команды. Заводим учётку `app` только на ключи `app:*`, затем гасим `default` для **новых** соединений.

```bash
docker exec -it redis-dev redis-cli ACL SETUSER app on '>dev-app' '~app:*' '+@read' '+@write' '+ping' '+info'
docker exec -it redis-dev redis-cli ACL SETUSER default off
```

`ACL` — именованные пользователи и права на команды/ключи (с Redis 6). Префикс `>` в правиле — это **пароль**, не редирект оболочки; кавычки обязательны.

Успех: оба раза `OK`. Проверка:

```bash
docker exec -it redis-dev redis-cli --user app -a dev-app --no-auth-warning PING
```

Ответ `PONG`. Без `--user` / `AUTH` новые соединения получают ошибку (default выключен).

Пароль **`dev-app` — только закрытый стенд**, не в бой и не в git. После `docker rm` без тома ACL пропадает: в памяти, пока не задан `aclfile` и `ACL SAVE` (или строки `user …` в `redis.conf` и `CONFIG REWRITE`).

### Этап 5. SET / GET с префиксом приложения

**Что делаем:** пишем и читаем ключ так, как будет микросервис.

```bash
docker exec -it redis-dev redis-cli --user app -a dev-app --no-auth-warning SET app:demo 1
docker exec -it redis-dev redis-cli --user app -a dev-app --no-auth-warning GET app:demo
```

Успех: `OK` и `"1"`. Ключ без префикса `app:` учётка `app` не трогает.

**Этот стенд не доказывает:** отказ машины, выборы master (Sentinel: минимум **3** процесса, два официально *DON'T DO THIS*), шарды Cluster, нагрузку, вытеснение при `maxmemory`, восстановление с диска, TLS, работу за NAT/пробросом портов (Sentinel autodiscovery так ломается).

## Первый запуск — URL, порт, учётка, смена пароля

Это **не HTTP**: в браузере URL нет. Клиенты открывают TCP.

| Что | Значение на стенде |
|---|---|
| Хост | `127.0.0.1` / `localhost` |
| Порт | **6379/TCP** (завод `redis.conf`; IANA) |
| Протокол | RESP (двоичный протокол Redis), не REST |
| Заводская учётка | `default`, правило `on nopass ~* &* +@all` — пароля **нет** |
| Учебная учётка | пользователь `app`, пароль `dev-app` (**стенд-only**) |
| UI | нет; консоль — `redis-cli` |

Вход:

```text
AUTH app dev-app
```

Один аргумент `AUTH пароль` — это пользователь `default`. После этапа 4 `default` выключен: нужна форма с именем.

Сменить пароль `app` (сбросить старые и поставить новый):

```bash
docker exec -it redis-dev redis-cli ACL SETUSER app resetpass '>новый-длинный'
```

`resetpass` обнуляет список паролей; `>…` добавляет новый. Сгенерировать длинный секрет: `ACL GENPASS` (вендор: Redis очень быстрый, короткий пароль перебирается).

Старый путь `requirepass` задаёт пароль только пользователю `default` и **несовместим** с `aclfile`. На стенде не нужен. Без TLS команда `AUTH` идёт открытым текстом, как остальные.

## Подключение к своей системе

Микросервис (и Camunda worker) — **клиент** Redis: Java **Lettuce** или **Jedis**, Python **redis-py**, .NET **StackExchange.Redis**. На стенде в клиенте: `127.0.0.1:6379`, username `app`, password из секрета, ключи с префиксом сервиса и **TTL** на кэше.

В секрет (Vault / Secret Kubernetes, **не git**): пароль (и имя пользователя ACL). Хост/порт учебного стенда в git можно. В бою клиент должен уметь **Sentinel** (порт **26379** — «кто сейчас master?»); захардкоженный IP master переживает только одиночку, как этот стенд.

Redis **не** эталон карточки (это PostgreSQL/озеро: промах кэша → прочитать эталон, не «клиента нет»). **Не** шина событий (это Kafka). **Не** один процесс и на кэш UI, и на локи, которые нельзя вытеснять.

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз **7.4.11** (17 Aug 2026, SECURITY) | https://github.com/redis/redis/releases/tag/7.4.11 |
| Образ `redis:7.4.11`; protected mode в образе выкл.; `latest` ≠ 7.4.11 | https://hub.docker.com/_/redis |
| Установка Docker (официальные шаги) | https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/ |
| Порт **6379**, `protected-mode yes` в бинарнике, `appendonly no`, `maxmemory` не задан | https://github.com/redis/redis/blob/7.4/redis.conf |
| Заводской `default`: `nopass` + все команды; `ACL SETUSER`; `ACL GENPASS`; `aclfile` / `ACL SAVE` vs `CONFIG REWRITE` | https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ |
| `AUTH` / `AUTH user password` | https://redis.io/docs/latest/commands/auth/ |
| `ACL SETUSER` (правила `on`, `>`, `resetpass`, `~`) | https://redis.io/docs/latest/commands/acl-setuser/ |
| Модель доверия; 6379 не в интернет; `requirepass` = пароль `default`; AUTH без TLS | https://redis.io/docs/latest/operate/oss_and_stack/management/security/ |
| Без `maxmemory` процесс ест свободную RAM | https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/ |
| Не класть RDB/AOF на NFS/NAS | https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/ |
| Репликация async; пустой master без persistence стирает replica | https://redis.io/docs/latest/operate/oss_and_stack/management/replication/ |
| Sentinel: не два процесса; Docker/NAT ломает discovery | https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ |
| Клиенты Lettuce / Jedis / redis-py / StackExchange.Redis | https://redis.io/docs/latest/develop/clients/ |
| Лицензия линии 7.4: RSALv2 или SSPLv1 (формулировка страницы образа) | https://hub.docker.com/_/redis |
| Community-оператор (не Redis Ltd; для учёбы не нужен) | https://github.com/OT-CONTAINER-KIT/redis-operator |
| Зачем продукт, порты, роль в платформе | `Redis 7.md` |
| Словарь | `Redis 7.info.md` |
| Стыковка со схемой платформы | `Redis 7.shema.md` |
| Эта карточка консультанта | `Redis 7.consultant.md` |

**В доке вендора Community 7.4 нет:** минимум CPU/RAM/диска «чтобы процесс поднялся»; учебный размер RAM в гигабайтах; порог RTT, при котором можно растянуть Sentinel/Cluster на несколько дата-центров. Цифры железа Redis Software на Community не переносить.
