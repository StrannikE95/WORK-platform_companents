# RabbitMQ 4.3.5 — установка (учебный контур)

Связанные документы: зачем и из чего состоит — `RabbitMQ.md`; словарь — `RabbitMQ.info.md`; стыковка с платформой — `RabbitMQ.shema.md`.

Это **как поставить учебный стенд**. Команды и пароли отсюда в бой не копируйте.

**Допущение:** закрытая сеть, одна машина с Docker, одна нода, AMQP без TLS. Community RabbitMQ **4.3.5** (MPL-2.0), не Tanzu. Шина событий платформы — **Kafka**; этот брокер — очереди задач / AMQP. Если все потоки уже на Kafka и AMQP-клиентов нет — стенд можно не поднимать.

Версия: **4.3.5**. Образ: `rabbitmq:4.3.5-management` (внутри уже Erlang **27.x**; свой Erlang младше 27.0 нода не стартует). Теги `4.3`, `4`, `latest`, `management` на дату Docker Hub указывают на этот патч — в командах пините **полный** тег. Релиз: https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.3.5

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна рабочая машина или Linux-VM **рядом** с учебным Kubernetes, не как кластер внутри него. На ней — Docker: программа, которая запускает **образ** (упакованная программа с зависимостями) как **контейнер** (запущенная копия). Порты AMQP **5672** и management UI **15672** публикуйте только на `127.0.0.1`. Порты **4369** (epmd — демон, по которому ноды и CLI находят друг друга) и **25672** (Erlang distribution — канал нода↔нода и CLI) **не** публиковать.

Одна нода на учёбе допустима. Живой кластер на 2–3 дата-центра здесь **не** собираем: при p99 RTT **> 100 мс** или потерях пакетов проект кластер **не рекомендует** (https://www.rabbitmq.com/docs/clustering). Kubernetes и Cluster Operator **v2.22.5** — не этот стенд; дефолт оператора на дату README тега — образ **4.3.4**, если пойдёте туда позже — пинить `rabbitmq:4.3.5-management` в `spec.image` (https://github.com/rabbitmq/cluster-operator).

**Сколько железа.** Цифр «хватит N ядер учебному контейнеру» у вендора **нет**. Не путать с боевым минимумом чеклиста.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Боевой минимум **на ноду** (не этот стенд) | 4 ядра, без соседства с I/O-тяжёлыми СУБД | 4 ГиБ | постоянный том | https://www.rabbitmq.com/docs/production-checklist |
| Учебный один контейнер | вендор: для QA/dev допустима более слабая машина | то же | эфемерный том на учёбе терпим | тот же чеклист: «lower-spec … QA and development» |
| `disk_free_limit` | — | — | завод **50 МБ** — для туториалов | тот же чеклист |

Дескрипторов файлов чеклист просит **не меньше 50 тысяч** и на разработке. Docker Desktop на ноутбуке часто даёт меньше — знакомство с протоколом это не ломает, готовность боя не доказывает.

**Сильная сторона:** совпадает с официальным Docker-путём для эксперимента (https://www.rabbitmq.com/docs/download, https://hub.docker.com/_/rabbitmq).  
**Слабая сторона:** падение этой машины = нет брокера; одна classic-очередь не реплицируется.

**Критично:** 5672/15672 не в интернет; учебный пароль не в бой; один контейнер ≠ кластер; не `latest`; Erlang **28** на живой 4.3.5 не ставить (только новые кластеры).

---

## Установка для новичка

Официальные страницы шагов: https://www.rabbitmq.com/docs/download · https://hub.docker.com/_/rabbitmq

Команды — в терминале той машины, где установлен Docker (Docker Desktop на Windows подходит).

### Что должно быть до установки

Есть:

- Docker (демон запущен).
- Свободны на localhost порты **5672** и **15672**.
- Закрытая сеть; вход с этой машины или VPN, не публикация портов в интернет.

Нет (и не должно появиться на этом стенде):

- Публикация **4369** / **25672**.
- Тег `latest` / `4-management` без патча.
- `loopback_users.guest = false` («пустить guest снаружи»).
- Копирование этого `docker run` в Kubernetes как «кластер».

### Этап 1. Проверяем Docker

**Что делаем:** убеждаемся, что Docker установлен и демон отвечает.

```bash
docker version
```

Успех: печатается версия клиента и сервера, без `Cannot connect to the Docker daemon`.

### Этап 2. Скачиваем закреплённый образ

**Что делаем:** забираем образ `rabbitmq:4.3.5-management` — брокер **и** плагин management (UI и HTTP API). Без суффикса `-management` UI на 15672 не будет.

```bash
docker pull rabbitmq:4.3.5-management
```

Успех: в выводе тег **4.3.5-management**, не `latest`.

### Этап 3. Запускаем контейнер

**Что делаем:** стартуем одну ноду. `--hostname` задаём явно: данные на диске привязаны к **имени ноды** (иначе случайный hostname — после рестарта брокер может не узнать свой каталог). `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` задают учётку **только при первом** создании базы; позже смена этих переменных пароль не меняет. Пароль ниже — **только закрытый стенд**.

```bash
docker run -d --hostname rmq-dev --name rmq-dev \
  -e RABBITMQ_DEFAULT_USER=app \
  -e RABBITMQ_DEFAULT_PASS=dev \
  -p 127.0.0.1:5672:5672 \
  -p 127.0.0.1:15672:15672 \
  rabbitmq:4.3.5-management
```

Привязка к `127.0.0.1` обязательна: `-p 5672:5672` без адреса часто слушает все интерфейсы.

Успех: `docker ps` показывает контейнер `rmq-dev`, порты `127.0.0.1:5672` и `127.0.0.1:15672`.

### Этап 4. Ждём готовности и проверяем версию

**Что делаем:** нода поднимается не мгновенно. `rabbitmq-diagnostics ping` — процесс отвечает; затем смотрим версию.

```bash
docker exec rmq-dev rabbitmq-diagnostics ping
docker exec rmq-dev rabbitmqctl version
docker exec rmq-dev rabbitmq-diagnostics status
```

Успех: `ping` — `Ping succeeded`; `version` — **4.3.5**; в `status` — Erlang **27.x**. Если `not enough arguments` / connection refused — подождать 10–20 с и повторить, не пересоздавать контейнер сразу.

Проверка учёток:

```bash
docker exec rmq-dev rabbitmqctl list_users
```

Успех: есть пользователь `app` с тегом администратора. Заводской `guest` при смене `default_user` обычно не создаётся; если вдруг есть — с сети он всё равно не войдёт (см. следующий раздел), на стенде его лучше удалить: `docker exec rmq-dev rabbitmqctl delete_user guest`.

### Этап 5. Стенд живой — сообщение туда и обратно

**Что делаем:** из UI (раздел ниже) или клиента публикуем в очередь на vhost `/` и забираем с **ручным ack**. Publisher confirm в клиенте лучше включить сразу: без него клиент не знает, дошло ли.

Успех: сообщение видно в очереди, потребитель подтвердил, очередь пуста.

Чего этот стенд **ещё не доказывает:** отказ зала, кворум Khepri, выборы лидера quorum queue, confirm через majority, TLS и его CPU, мост Shovel/Federation, нагрузку, «три ноды». Classic-очередь на одной ноде **не копируется**. Успешный publish ≠ готовность боя.

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL и порт.** Management UI (браузерная админка плагина management): `http://127.0.0.1:15672/`  
Порт **15672** — HTTP UI и HTTP API. Это **не** AMQP, **не** MQTT и **не** STOMP: клиенты сообщений сюда не ходят (https://www.rabbitmq.com/docs/management).

Открывайте с той же машины (или SSH-туннель). 15672 в интернет не выставлять.

**Учётка стенда.** Логин **`app`**, пароль **`dev`** — только закрытый стенд. В UI нужен тег `administrator` / `management` / `monitoring`; без тега вход в UI отвергается, даже если AMQP пускает.

Заводские `guest` / `guest` (если учётка есть): ходят **только с loopback** брокера, на любом протоколе. С хоста через `-p 127.0.0.1:15672:15672` пакет внутри контейнера часто приходит **не** с `127.0.0.1` контейнера — `guest` тогда **откажут**. Это штатно, не «сломанный Docker». Пускать `guest` снаружи (`loopback_users = none`) вендор считает опасной практикой: учётку удаляют, заводят новую (https://www.rabbitmq.com/docs/access-control).

**Смена пароля.** Сразу после входа смените учебный `dev`. CLI (пароль подставьте свой; в бою — секрет из Vault, не эта строка):

```bash
docker exec rmq-dev rabbitmqctl change_password app 'НОВЫЙ_ПАРОЛЬ_СТЕНДА'
```

Документация команды: https://www.rabbitmq.com/docs/man/rabbitmqctl.8  
В UI: пользователь с тегом администратора может менять пользователей (Admin → Users). HTTP API: `PUT /api/users/{name}` (тот же плагин, порт 15672).

После смены пароля старые AMQP-сессии с прежним паролем могут ещё жить — при утечке: `rabbitmqctl close_all_user_connections app`. Переменная `RABBITMQ_DEFAULT_PASS` на уже созданной базе пароль **не** обновит.

---

## Подключение к своей системе

Приложения открывают **протокол сообщений**, не браузер. Имена exchange / очередей / vhost кладите в git; пароли — нет.

### Протокол и порт

| Что | Порт | Зачем | На этом стенде |
|---|---|---|---|
| **AMQP 0-9-1** (и AMQP 1.0 на том же порту) | **5672** без TLS; в бою **5671** + TLS | Очередь задач: publish в **exchange**, ждать в **очереди**, competing consumers | основной путь |
| **MQTT** 3.1 / 3.1.1 / 5.0 | **1883** (TLS **8883**) | IoT-подписки; плагин `rabbitmq_mqtt`, в образе management **выключен**, пока не включите | не нужен, пока нет MQTT-клиентов |
| Management HTTP / UI | **15672** (HTTPS **15671**) | Люди, Topology Operator, `rabbitmqadmin`; не шина сообщений | только localhost |

MQTT: https://www.rabbitmq.com/docs/mqtt · порты: https://www.rabbitmq.com/docs/networking

Клиент AMQP: `localhost:5672`, vhost **`/`**, пользователь `app`. Publisher confirms + manual ack в **коде**. Auto-ack теряет сообщение, если воркер умер на полуслове.

### Кто клиент

- Микросервисы и воркеры интеграционного API — AMQP-библиотека на **5672**.
- Java AMQP 0-9-1: `com.rabbitmq:amqp-client` **5.33.0** (на дату страницы https://www.rabbitmq.com/client-libraries/java-client).
- .NET: официальный клиент линейки **7.0** (https://www.rabbitmq.com/client-libraries/dotnet-api-guide).
- Python: у команды RabbitMQ есть AMQP **1.0**-клиент; типичный 0-9-1 — сообщество (Pika и др.), версию 0-9-1 вендор здесь не пинит.
- Camunda **8** / Zeebe в этот брокер **не** ходит (свой протокол). GeoData — легаси-мост AMQP, не замена Kafka.
- Kafka, озеро, эталон карточек в RabbitMQ **не** пишут.

### Что в секрет, что в git

| Секрет | Где живёт на стенде | Куда не класть |
|---|---|---|
| Пароль `app` | переменная первого запуска / Vault | git, образ, чат |
| Erlang cookie | файл в томе контейнера (`/var/lib/rabbitmq/.erlang.cookie`) | git; это пропуск **между нодами и CLI**, не пароль приложения |
| TLS-ключи (когда появятся) | PKI / Secret | публичный репозиторий |

В git: имена vhost, exchange, очередей, binding. Не учебный `dev`.

### Чем продукт не является

| Сосед / ожидание | Чем отличается |
|---|---|
| **Kafka** | Шина событий, долгий лог, consumer group на партиции. Rabbit — очередь работ: после ack сообщения в брокере нет |
| NATS в Luxms | Внутренняя очередь **другого** продукта |
| Озеро / PostgreSQL | Эталон карточки. Тело в очереди — буфер задачи, не SoT |
| Camunda 8 | Исполнение BPMN; транспорт — Zeebe, не этот брокер |
| MQTT-брокер «для всего IoT» | Плагин опционален; платформенная роль — AMQP |
| HTTP API 15672 | Админка, не протокол публикации бизнес-событий |
| Stretch-кластер на 2–3 ЦОДа | На учёбе одной ноды нет; в схеме платформы — отдельные кластеры + Shovel/Federation, не Raft через город |

На стенде TLS нет — так и задумано для routing key. Клиентский код лучше сразу писать с confirm/ack, иначе тест врёт про бой.

---

## Ссылки на материал

Факты в этом файле взяты со **страниц** ниже, не «из документации вообще».

| Факт | URL |
|---|---|
| Релиз **4.3.5** (17 августа 2026), минимум Erlang **27.0** | https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.3.5 |
| Матрица Erlang: 4.3.5 → **27.x**; Erlang 28 — только новые кластеры | https://www.rabbitmq.com/docs/which-erlang |
| Docker-эксперимент, «latest 4.x» как `rabbitmq:4-management` (мы пиним патч) | https://www.rabbitmq.com/docs/download |
| Образ: теги **4.3.5-management**; `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS`; UI `http://localhost:15672`; `--hostname` | https://hub.docker.com/_/rabbitmq |
| `guest` только с localhost; не `loopback_users = none`; `change_password` / `delete_user` | https://www.rabbitmq.com/docs/access-control |
| `rabbitmqctl change_password` | https://www.rabbitmq.com/docs/man/rabbitmqctl.8 |
| UI/HTTP API порт **15672**; не AMQP/MQTT; теги пользователей | https://www.rabbitmq.com/docs/management |
| Порты 5672/5671, 15672/15671, 1883/8883, 4369, 25672 | https://www.rabbitmq.com/docs/networking |
| MQTT-плагин, порт 1883, по умолчанию выключен | https://www.rabbitmq.com/docs/mqtt |
| Таблица RTT; > 100 мс / потери → не кластер, а Shovel/Federation | https://www.rabbitmq.com/docs/clustering |
| Боевой минимум 4 CPU / 4 GiB; QA/dev слабее можно; `disk_free_limit` 50 МБ для туториалов; ≥ 50k дескрипторов; удалить `guest` | https://www.rabbitmq.com/docs/production-checklist |
| Cluster Operator **v2.22.5** (24 августа 2026), дефолт образа **4.3.4** | https://github.com/rabbitmq/cluster-operator |
| Обзор операторов Kubernetes | https://www.rabbitmq.com/kubernetes/operator/operator-overview |
| Java-клиент AMQP 0-9-1 **5.33.0** | https://www.rabbitmq.com/client-libraries/java-client |
| .NET-клиент, гайд 7.0 | https://www.rabbitmq.com/client-libraries/dotnet-api-guide |
| Publisher confirms / consumer ack | https://www.rabbitmq.com/docs/confirms |
| Quorum vs classic; mirroring удалён | https://www.rabbitmq.com/docs/quorum-queues |
| Правила, порты, роль в платформе | `RabbitMQ.md` |
| Словарь | `RabbitMQ.info.md` |
| Схемы стыковки (без stretch) | `RabbitMQ.shema.md` |

Порога «хватит N ядер учебному контейнеру» и вашей нагрузки в документации **нет** — в этом файле их нет. Stretch-кластер не предлагается.
