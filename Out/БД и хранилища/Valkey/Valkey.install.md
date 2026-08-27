# Valkey 9.1.1 — установка (учебный контур)

Допущение: закрытый учебный стенд, Docker, слушать только `127.0.0.1`. Это не бой.

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Один контейнер Docker на машине разработчика (Linux, WSL или Docker Desktop). Образ `valkey/valkey:9.1.1`. Порт **6379/TCP** публиковать только на `127.0.0.1`.

Windows как родная ОС сервера вендор **не поддерживает** (для разработки — WSL). В Kubernetes учебный путь — Helm `valkey` **0.11.0** (`appVersion` 9.1.1), одиночка; не Cluster, не Sentinel. Оператор `valkey-operator` в README: *not ready for production* — на стенд «как будущий бой» не ставим.

**Сколько железа.** Цифр «N ядер / N ГБ, чтобы процесс поднялся» и «хватит для учёбы» в мануале Valkey **нет**. Данные живут в **RAM**. Чарт Helm поле `resources` по умолчанию пустое (`{}`) — это не смета.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум, чтобы процесс поднялся | в доке нет | в доке нет; на ARM вендор пишет «малый footprint», без гигабайт | в доке нет | installation, ARM, Helm values |
| Учебный ориентир | 1 контейнер рядом с IDE | игрушечный датасет; `maxmemory` можно не ставить | persistence выкл. — ключи в RAM, рестарт = пусто | эта инструкция, не смета |
| Если `maxmemory` не задан | — | процесс **может съесть всю свободную RAM** машины | — | [Memory optimization](https://valkey.io/topics/memory-optimization/) |

Helm в примере persistence ставит **10 ГиБ** — заглушка чарта, не расчёт ваших ключей. На этом стенде том не обязателен.

**Сильная сторона:** официальный образ, минуты, тот же протокол, что у клиентов Redis.  
**Слабая сторона:** нет смены primary, нет replica, успех `GET` не доказывает отказ зала и нагрузку.

**Критично:**

- **6379 в интернет** — открытая память. В образе Docker *protected mode* **выключен**: `-p 6379:6379` без `127.0.0.1` = мир без пароля.
- **NFS/NAS** под RDB/AOF: вендор в бенчмарке пишет *Avoid putting RDB or AOF files on NAS or NFS shares*. На учёбе том с сети не нужен.
- Тег **`latest`** и **9.1.0** запрещены. Пин **9.1.1** (SECURITY: CVE-2026-56684, CVE-2026-63639).
- Один процесс **не** растягивать на 2–3 дата-центра. Порога RTT в доке **нет**. Между площадками — независимый Valkey или клиенты в «домашний» зал.

## Установка для новичка

Официальная страница установки: https://valkey.io/topics/installation/  
Образы: https://valkey.io/download/

**Docker** — программа, которая запускает **образ** (упакованная программа с зависимостями) как **контейнер** (запущенная копия). Команды — в Linux-shell / WSL, не «голый» Windows без Docker.

### Что должно быть до установки

Есть:

- Docker Engine (или Docker Desktop).
- Свободен порт **6379** на localhost.
- Сеть стенда не торчит в интернет; образ `valkey/valkey:9.1.1` можно скачать (или уже в локальном registry).

Нет (для этого стенда не нужно):

- Kubernetes, Helm, Sentinel, Cluster, TLS, PVC.
- Заводской пароль — его **нет**. Вендор: по умолчанию нет аутентификации.

### Этапы

**1. Скачать образ 9.1.1**

**Что делаем:** кладём на машину именно патч 9.1.1, не «девятку вообще».

```bash
docker pull valkey/valkey:9.1.1
```

Успех: в `docker images` тег `9.1.1`, не `latest`.

**2. Запустить контейнер с паролем и петлёй**

**Что делаем:** поднимаем процесс Valkey. Пароль в команде — слой совместимости `requirepass` (пароль пользователя `default`). Только стенд.

```bash
docker run -d --name valkey-dev \
  -p 127.0.0.1:6379:6379 \
  valkey/valkey:9.1.1 \
  valkey-server --requirepass 'stand-only-dev'
```

Успех: `docker ps` показывает `valkey-dev` в статусе Up. Без `127.0.0.1` порт часто слушает все интерфейсы хоста.

**3. Проверить PING и версию**

**Что делаем:** `valkey-cli` — консольный клиент из того же образа; `PING` — «живой ли сервер».

```bash
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning PING
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning INFO server
```

Успех: ответ `PONG`. В `INFO` есть **9.1.1** (поле `valkey_version`; совместимое `redis_version` — не путать с «это Redis 7»).

**4. SET / GET**

**Что делаем:** пишем и читаем ключ — минимальная проверка протокола.

```bash
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning SET stand:ping 1
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning GET stand:ping
```

Успех: `GET` возвращает `1`. Затем тот же SET/GET из **вашего** сервиса, не только cli.

**5. Пользователь приложения (ACL)**

**Что делаем:** **ACL** — пользователи и права на команды/ключи. Не ходить в бой как всесильный `default`.

```bash
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning ACL SETUSER app on '>dev-app' '~app:*' '+@read' '+@write' '+ping' '+info'
docker exec -it valkey-dev valkey-cli --user app -a 'dev-app' --no-auth-warning SET app:demo 1
```

Успех: `OK` на `ACL SETUSER` и на `SET app:demo`. Без файла `aclfile` этот пользователь **пропадёт после рестарта** контейнера — на стенде заведите снова или держите только `requirepass` в команде запуска.

### Если уже есть учебный Kubernetes

**Helm** — шаблон манифестов Kubernetes. Репозиторий чартов: https://valkey.io/valkey-helm/

```bash
helm repo add valkey https://valkey.io/valkey-helm/
helm repo update
helm install valkey-dev valkey/valkey --version 0.11.0 --set image.tag=9.1.1
```

Успех: под Running, Service `valkey` слушает **6379** (это **писатель**). Завод чарта: auth выкл., replica выкл., persistence выкл. Если сеть шире ноутбука — включите `auth.enabled` и **обязательно** пользователя `default` в ACL (иначе, цитата чарта: *anyone can access the database without credentials*). Replication **без** PVC не включайте: пустой primary после рестарта съест копии ([replication safety](https://valkey.io/topics/replication/#safety-of-replication-when-primary-has-persistence-turned-off)).

### Чего этот стенд не доказывает

Отказ пода/зала, выборы primary (Sentinel в чарт **0.11.0 не входит**), шарды Cluster, лаг replica, `WAIT`, вытеснение при полном `maxmemory`, TLS, нагрузку. Успешный Docker ≠ готовность боя.

## Первый запуск — URL, порт, учётка, смена пароля

HTTP-URL и веб-консоли **нет**. Клиенты ходят по TCP.

| | Значение |
|---|---|
| Хост | `127.0.0.1` (с хоста). Из другого контейнера — имя `valkey-dev` в общей Docker-сети, не публиковать `-p` на `0.0.0.0` |
| Порт | **6379/TCP** ([installation](https://valkey.io/topics/installation/), [valkey.conf](https://github.com/valkey-io/valkey/blob/9.1.1/valkey.conf)) |
| Заводская учётка | **нет**. Без `requirepass`/ACL любой, кто достучался, — хозяин |
| Стенд | пользователь `default`, пароль `stand-only-dev` — **только закрытый стенд** |
| Приложение | пользователь `app`, пароль `dev-app` — **только стенд**; ключи с префиксом `app:` |

Смена пароля `default` на уже запущенном процессе:

```bash
docker exec -it valkey-dev valkey-cli -a 'stand-only-dev' --no-auth-warning CONFIG SET requirepass 'stand-only-new'
```

Или ACL: `ACL SETUSER default on >новый-пароль`. `requirepass` — совместимость: пароль пользователя `default`. С `aclfile` / `ACL LOAD` директива `requirepass` **игнорируется**. Сменить пароль в команде `docker run` и пересоздать контейнер — надёжнее на стенде без файла конфига.

В бою (не эта инструкция): свой секрет в Vault/K8s Secret, длинный пароль (`ACL GENPASS`), не эти строки.

Проверка с хоста, если установлен клиент:

```bash
valkey-cli -h 127.0.0.1 -p 6379 -a 'stand-only-dev' --no-auth-warning PING
```

Клиент Redis-протокола тоже подходит: `AUTH stand-only-dev` или `AUTH default stand-only-dev`.

## Подключение к своей системе

| | |
|---|---|
| Протокол | TCP, протокол Valkey / Redis (**RESP**). Клиенты Redis OSS **7.2+** вендор считает совместимыми с Valkey 7.2+ |
| Порт | **6379** |
| Кто клиент здесь | микросервисы и интеграционное API (кэш ответа, лимит, лок); воркеры Camunda (лок, идемпотентность). Не Kafka, не озеро, не сам Valkey |
| В секрет (не git) | хост, порт `6379`, имя пользователя, пароль. Для Helm — Secret с паролями ACL, не values в репозитории |
| В git | префиксы ключей, TTL, имя Service; не пароль |

Подключение из приложения (идея, не конкретная библиотека): host `127.0.0.1`, port `6379`, username `app`, password из переменной окружения / Secret, ключи `app:…`, TTL. Список клиентов: https://valkey.io/clients/

**Это не:**

- источник истины карточек (эталон — СУБД/озеро);
- база «как PostgreSQL» и не шина «как Kafka»;
- Redis Enterprise / облачный Memorystore;
- Cluster и не Sentinel на этом стенде — обычный TCP на один адрес **не** переживает смену primary.

Между дата-центрами не один набор ключей: либо свой Valkey в каждом Kubernetes, либо клиенты ходят в домашний зал.

## Ссылки на материал

| Факт | URL / файл |
|---|---|
| Релиз 9.1.1, CVE-2026-56684 / CVE-2026-63639 | https://github.com/valkey-io/valkey/releases/tag/9.1.1 |
| Образы `valkey/valkey:9.1.1` (`-alpine`, `-trixie`) | https://valkey.io/download/ |
| Установка, порт 6379, нет auth с завода, firewall | https://valkey.io/topics/installation/ |
| Docker: protected mode выкл., `valkey-server` args, `/data` | https://hub.docker.com/r/valkey/valkey/ |
| `requirepass`, bind, port, `maxmemory` не задан | https://github.com/valkey-io/valkey/blob/9.1.1/valkey.conf |
| ACL, `ACL SETUSER` | https://valkey.io/topics/acl/ · https://valkey.io/commands/acl-setuser/ |
| `AUTH` (default vs username) | https://valkey.io/commands/auth/ |
| Нет цифры CPU/RAM; без `maxmemory` ест свободную память | https://valkey.io/topics/memory-optimization/ |
| Не класть RDB/AOF на NFS/NAS | https://valkey.io/topics/benchmark/ |
| Helm repo, чарт `valkey` **0.11.0**, `appVersion` 9.1.1 | https://valkey.io/valkey-helm/ · https://github.com/valkey-io/valkey-helm/blob/main/valkey/Chart.yaml |
| Чарт: не Cluster; auth + обязательный `default`; replica + PVC | https://github.com/valkey-io/valkey-helm/blob/main/valkey/README.md |
| Пустой primary без persistence уничтожает replica | https://valkey.io/topics/replication/#safety-of-replication-when-primary-has-persistence-turned-off |
| Оператор *not ready for production* | https://github.com/valkey-io/valkey-operator/blob/main/README.md |
| Клиенты (в т.ч. Redis 7.2+) | https://valkey.io/clients/ |
| Windows не поддерживается, WSL | https://valkey.io/topics/installation/ |
| Зачем продукт, порты, роль в платформе | `Valkey.md` |
| Словарь | `Valkey.info.md` |
| Стыковка с платформой | `Valkey.shema.md` |
| Эта инструкция | `Valkey.install.md` |
