# Apache Flink 2.2.1 — установка (учебный контур)

Apache Flink — потоковый движок: читает события, считает окна и соединения, пишет проекции. Это **не** шина Kafka, **не** база и **не** Camunda. Ставите **свою** копию, не облако вендора.

**Допущение:** закрытая сеть, одна машина с Docker, версия **2.2.1**, образ **`flink:2.2.1-java17`** (Java **17**). Живой кластер на несколько дата-центров **не** собираем: порога RTT в документации нет. Этот запуск в прод не копировать.

**Docker** — программа, которая запускает **образ** (упакованная программа с зависимостями) как **контейнер**. Официальные шаги: [Docker Setup, линейка 2.2](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/). Тег `latest` на Docker Hub — **2.3.0**; оператор **1.15.0** линейку 2.3 **не** перечисляет — `latest` не брать.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна машина с Docker (Linux или Docker Desktop). **JobManager** (процесс-планировщик: план, снимки, REST/UI) и **TaskManager** (процесс-рабочий: чтение, окна, запись) — **два контейнера** в одной Docker-сети. Люди и клиенты ходят на **8081** JobManager, не на внутренний RPC рабочих.

```mermaid
flowchart LR
  U["Браузер / CLI"] -->|"8081 REST/UI"| JM["Контейнер JobManager"]
  JM --> TM["Контейнер TaskManager"]
  TM -->|"учебный job"| KF["Kafka стенда\nне обязательна для старта"]
```

Порт **8081** на хосте публикуйте только как `127.0.0.1:8081`. Внутри образа REST слушает все интерфейсы контейнера (`0.0.0.0`) — иначе проброс порта не работает; ограничение «только эта машина» делается **публикацией Docker**, не выставлением 8081 в интернет.

Оператор Kubernetes **1.15.0** в этот учебный контур **не** входит. Java **17** на хосте нужна, только если собираете JAR задания **без** Docker (Java 21 в 2.2 — экспериментально).

**Сколько.** Числа ядер «хватит для учёбы» в Docker Setup **нет**. Заводской `config.yaml` дистрибутива **2.2.1** задаёт память процессов:

| Процесс | Заводской размер | Откуда |
|---|---|---|
| JobManager | `jobmanager.memory.process.size`: **1600m** | [config.yaml 2.2.1](https://github.com/apache/flink/blob/release-2.2.1/flink-dist/src/main/resources/config.yaml) |
| TaskManager | `taskmanager.memory.process.size`: **1728m** | тот же файл |

Хост должен вместить **оба** процесса плюс Docker и ОС. Это «чтобы процессы с заводским конфигом стартовали», не смета боя и не «хватит для терабайтов». Нагрузки в контексте нет — ядер и гигабайт «на поток» не подставляем.

**Сильная сторона:** совпадает с Getting Started Docker, UI за минуты. **Слабая:** один JobManager без запаса лидера; снимок по умолчанию в куче менеджера; падение машины = нет кластера.

**Критично:** 8081 в интернет не публиковать (REST клиента **не** спрашивает). Не `flink:latest`. Не один процесс «как кластер»: без TaskManager задание считать негде. NFS как рабочий диск TaskManager не описываем и не берём.

---

## Установка для новичка

Команды — **bash**, как на [Docker Setup](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/). На Windows — WSL или Git Bash, не «голый» PowerShell с теми же кавычками.

Образ: **`flink:2.2.1-java17`**. На Docker Hub это тот же набор, что `2.2.1-scala_2.12-java17`; страница Docker Setup пишет `flink:2.2.1-scala_2.12` (алиас Java 17). Не `2.3.0`.

### Что должно быть до установки

**Есть:**

- Docker (демон запущен: `docker version` отвечает)
- свободен порт **8081** на этой машине
- закрытая сеть; вход с jump-хоста / VPN

**Нет** (и не должно появиться):

- публикация `0.0.0.0:8081` в интернет
- тег `latest` / образ 2.3.0
- Java 21 как рантайм этого стенда
- учебная Kafka — **не** обязательна, чтобы поднять JM+TM и UI; нужна только чтобы job прочитал топик

### Этап 1. Сеть Docker и адрес менеджера

**Что делаем:** контейнеры должны видеть друг друга по имени. Свойство `jobmanager.rpc.address` (куда рабочий стучится за планом; RPC JobManager, порт **6123**) ставим в имя контейнера менеджера.

```bash
docker network create flink-network
export FLINK_PROPERTIES="jobmanager.rpc.address: jobmanager"
```

Успех: `docker network ls` показывает `flink-network`. Без этой сети имя `jobmanager` для рабочего не резолвится.

### Этап 2. JobManager

**Что делаем:** запускаем планировщик. Официальный пример без `-d`; здесь `-d`, чтобы стенд жил в фоне. Имя контейнера **`jobmanager`** — как в `FLINK_PROPERTIES`.

```bash
docker run -d \
  --name=jobmanager \
  --network flink-network \
  -p 127.0.0.1:8081:8081 \
  --env FLINK_PROPERTIES="${FLINK_PROPERTIES}" \
  flink:2.2.1-java17 jobmanager
```

Успех: `docker ps` — контейнер `jobmanager`, статус Up. Проброс только на loopback хоста.

### Этап 3. TaskManager

**Что делаем:** запускаем рабочего в **той же** сети. Адрес менеджера — имя контейнера, не чужой хост из старых compose.

```bash
docker run -d \
  --name=taskmanager \
  --network flink-network \
  --env FLINK_PROPERTIES="${FLINK_PROPERTIES}" \
  flink:2.2.1-java17 taskmanager
```

Успех: `docker ps` — оба контейнера Up. В UI (следующий раздел) виден зарегистрированный TaskManager, слоты не нулевые.

### Этап 4. Стенд живой

**Что делаем:** проверяем REST JobManager и сдаём комплектный пример из образа (Kafka не нужна).

```bash
curl -s http://127.0.0.1:8081/overview
docker exec jobmanager ./bin/flink run examples/streaming/TopSpeedWindowing.jar
```

Успех: в JSON поле `"flink-version"` = **`2.2.1`**; команда `flink run` завершается без ошибки; в UI задание RUNNING или FINISHED. Пример — с [той же страницы Docker](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/).

Чего этот стенд **ещё не доказывает:** отказ зала, выборы лидера JobManager, выгрузку снимка в объектное хранилище, ёмкость RocksDB, нагрузку, «ровно один раз» до Kafka (заводской приёмник — **без гарантии**). Рестарт менеджера без внешнего каталога снимков — состояние потеряно: это ожидаемо.

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL и порт.** Web UI и REST — один сервер JobManager, порт **`8081`** (`rest.port` / `rest.bind-port`).

- В браузере: `http://127.0.0.1:8081`
- Проверка: `curl -s http://127.0.0.1:8081/overview` — в ответе `"flink-version": "2.2.1"`

Документация порта: [REST API 2.2](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/rest_api/), [Configuration](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/config/). Docker Setup: UI на `localhost:8081`.

**Учётка.** Заводской Web UI **логина не спрашивает**. Пароля «из коробки» нет — менять нечего. REST «принимает соединения от любого клиента» ([SSL Setup 2.2](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/security/security-ssl/)).

**Закрытый стенд.** 8081 только с этой машины (`127.0.0.1`) или с VPN/jump на loopback. Не публиковать в интернет: это панель управления кластером (остановить job, снять сейвпоинт, сдать архив).

Если к UI должны ходить люди не с localhost — не «открыть 8081 миру». Рекомендация вендора: REST на loopback (или интерфейс пода) + **прокси** (Envoy / NGINX) с нормальной аутентификацией. Встроенный логин/пароль UI в документации 2.2 как заводской режим **не** описан. mTLS REST (`security.ssl.rest.authentication-enabled`) есть, вендор всё равно советует прокси.

TLS REST на этом стенде не включаем (сеть закрыта). Учебной учётки в git класть нечего.

---

## Подключение к своей системе

Задание Flink **читает и пишет Kafka**. Шина остаётся Kafka; Flink не хранит лог «навсегда» и не является эталоном карточек.

### Протокол

| Что | Как |
|---|---|
| Люди и автоматизация к кластеру | **HTTP REST** JobManager, порт **8081** (UI — тот же порт, другие URL) |
| Сдача job | CLI `flink run` на REST **или** кнопка Submit в UI (`web.submit.enable` по умолчанию включён) |
| Вход/выход данных | **KafkaSource / KafkaSink**, коннектор **5.0.0** (Maven: `flink-connector-kafka` **`5.0.0-2.2`**, совместим с Flink **2.2.x**) |
| Внутренний RPC JM | **6123** — клиентам платформы **не** открывать |
| Обмен рабочих | свои порты (часто эфемерные); не HTTP |

Коннектор **не** входит в бинарный дистрибутив — кладётся в JAR задания (или в образ) **до** сдачи. Старые `FlinkKafkaConsumer` / `FlinkKafkaProducer` не использовать.

Учебная Kafka — отдельный стенд (`../Бэкенд/Apache Kafka.install.md`). В job: `setBootstrapServers(...)` на брокер **этого** зала, не через город.

### Кто клиент

- Человек — браузер на `http://127.0.0.1:8081`
- CI — собирает JAR (Java **17**) и сдаёт на REST
- Микросервисы / Camunda / интеграционное API — пишут и читают **Kafka**, не «ходят в Flink как в БД»

### Что в секрет, что в git

| Секрет | Где | Не класть |
|---|---|---|
| Логин/пароль / SASL / TLS к Kafka | секрет стенда / Vault; в job — через env/секрет, не хардкод | git, манифест в чате |
| Префикс транзакций Kafka (`transactionalIdPrefix`) | уникален среди job на **одном** кластере брокеров | два живых job с одним префиксом |
| Ключи объектного хранилища снимков | на этом стенде **нет** (куча JM) | — |

В git: код job, стабильные **uid** операторов с состоянием, имена топиков. Не git: пароли брокера.

Заводской `KafkaSink` — `DeliveryGuarantee.NONE` (могут быть потери и дубли). «Поставили Flink, значит ровно один раз» — неверно. Режим транзакций на стенде не обязателен; **в коде** лучше сразу тот режим, что будет дальше. Два job с одним `transactional.id` на одну Kafka — конфликт.

### Чем продукт не является

| Сосед / ожидание | Чем отличается |
|---|---|
| Kafka | Шина (лог). Flink — обработка |
| ClickHouse / озеро | Склад аналитики и эталон карточек. Чекпоинт Flink — не копия озера |
| Camunda | Долгие процессы с людьми, не stream job |
| PostgreSQL | Точечные правки «как в СУБД» здесь плохо живут: после сбоя job **переигрывает** вход со снимка |
| Интеграционное API к ведомствам | Ходить в 30 ведомств «вместо API» из job — сломать контур лагов |

На стенде прямой `flink run` игрушечного JAR — норма. В проде артефакт — из CI, не «с ноутбука в открытый 8081».

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз **2.2.1** (15 мая 2026) | https://flink.apache.org/2026/05/15/apache-flink-2.2.1-release-announcement/ |
| Образ, теги **2.2.1-java17** / не путать с `latest`=2.3.0 | https://hub.docker.com/_/flink |
| Docker: сеть, JM+TM, порт **8081**, пример `TopSpeedWindowing.jar` | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/ |
| Java **17** рекомендуется; Java 21 в 2.2 экспериментально | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/java_compatibility/ |
| REST/UI порт **8081**, `GET /overview` → `flink-version` | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/rest_api/ |
| `rest.port` / `rest.bind-port` = 8081 | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/config/ |
| REST без аутентификации клиента; рекомендация — прокси | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/security/security-ssl/ |
| Заводская память JM **1600m**, TM **1728m** | https://github.com/apache/flink/blob/release-2.2.1/flink-dist/src/main/resources/config.yaml |
| Снимки: куча JM — разработка / крошечное состояние | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/state/checkpoints/ |
| Kafka Connector **5.0.0**, совместим с **2.2.x**; артефакт `5.0.0-2.2` | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/connectors/datastream/kafka/ |
| Загрузки коннектора 5.0.0 (2.1.x и 2.2.x) | https://flink.apache.org/downloads/ |
| Оператор **1.15.0**, матрица **2.2.x** (не 2.3.0) | https://flink.apache.org/2026/05/26/apache-flink-kubernetes-operator-1.15.0-release-announcement/ |
| Helm-репозиторий оператора 1.15.0 | https://archive.apache.org/dist/flink/flink-kubernetes-operator-1.15.0/ |
| Запас лидера JobManager | https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/ha/overview/ |
| Документация линейки 2.2 | https://nightlies.apache.org/flink/flink-docs-release-2.2/ |
| Зачем продукт, порты, состав | `Apache Flink.md` |
| Словарь | `Apache Flink.info.md` |
| Схема стыковки с платформой | `Apache Flink.shema.md` |
| Роль консультанта | `Apache Flink.consultant.md` |
| Учебная шина | `../Бэкенд/Apache Kafka.install.md` |

**В доке вендора нет (и здесь не выдумано):** число ядер CPU для Docker-стенда; смета RAM «хватит для вашей нагрузки»; порог RTT между дата-центрами; заводской логин/пароль Web UI; stretch-кластер JM/TM на 2–3 зала.
