# Apache Kafka 4.3.1 — установка (учебный контур)

Словарь: `Apache Kafka.info.md`. Зачем продукт и порты: `Apache Kafka.md`. Стыковка: `Apache Kafka.shema.md`.

**Допущение:** закрытый учебный стенд на одной машине. Это не бой и не Confluent Platform. Настройки и пароли отсюда в бой не копировать.

Ставите **Apache Kafka 4.3.1**, только **KRaft** (метаданные внутри Kafka; **ZooKeeper нет**). Образ: `apache/kafka:4.3.1`. Один процесс сразу брокер и контроллер (**combined mode**) Apache разрешает для development и **не рекомендует** для critical.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна машина в закрытой сети. Основной путь — **Docker** (программа, которая запускает **образ** — упакованную программу с зависимостями — как контейнер). Брокер слушает **только** `127.0.0.1:9092`. Живой кластер на 2–3 дата-центра **не** собираем: порога RTT в документации Apache **нет**; `acks=all` не быстрее реплики в чужой площадке.

Если уже есть однонодовый Kubernetes — можно Strimzi **1.2.0** и Kafka **4.3.1** с **одной** replica. Этот YAML в бой не копировать.

Без Docker: бинарный дистрибутив `kafka_2.13-4.3.1` и **Java ≥ 17** (официально 17 / 21 / 25). Нативная установка на Windows у Apache «not currently a well supported platform».

**Сколько железа.** В мануале Apache **нет** минимума CPU/RAM «чтобы Docker-стенд поднялся» и **нет** сметы «хватит для терабайтов». Цифры с официальных страниц — про другое:

| Цифра | Что это на самом деле | Откуда |
|---|---|---|
| `-Xmx6g -Xms6g` | Пример **нагруженного** кластера LinkedIn (60 брокеров, 300 МБ/с inbound) | [Java Version](https://kafka.apache.org/43/operations/java-version/) |
| ~5 ГиБ RAM и ~5 ГиБ диск контроллера | Оценка Apache для **типичного** кластера, не combined-стенд | [KRaft](https://kafka.apache.org/43/operations/kraft/) |
| Dual Xeon / 24 ГиБ | Машины, на которых писали Hardware and OS, не требование учёбы | [Hardware and OS](https://kafka.apache.org/43/operations/hardware-and-os/) |

Для учёбы хватает одной машины с Docker **≥ 20.10.4** и свободным портом 9092. Кучу 6 ГиБ на ноутбук не копировать. Диск лога (`log.dirs`) — **локальный** POSIX (вендор описывает EXT4/XFS и page cache). Том контейнера — на локальном диске хоста.

**Сильная сторона:** официальный combined на одном процессе, минуты.  
**Слабая сторона:** падение этой машины = простой; RF=1 не копирует данные.

**Критично:**

- Порт **9092** не публиковать в интернет (`-p 9092:9092` без `127.0.0.1` часто слушает все интерфейсы).
- Тег **`4.3.1`**, не `latest`.
- NFS как единственный `log.dirs` на этой платформе **не** ставят. В [Hardware and OS](https://kafka.apache.org/43/operations/hardware-and-os/) NFS **не назван**; там локальные диски и page cache.
- Один combined-процесс — не кластер и не кворум из трёх контроллеров.
- Не растягивать брокер/контроллер на несколько дата-центров.

---

## Установка для новичка

Официальные шаги: [Docker](https://kafka.apache.org/43/getting-started/docker/) и [Quick Start](https://kafka.apache.org/43/getting-started/quickstart/). Гайд образа: [Kafka Docker Image Usage Guide](https://github.com/apache/kafka/blob/trunk/docker/examples/README.md) (Docker **≥ 20.10.4**). Без своих переменных образ поднимает **default KRaft combined**. Если задать хотя бы одну `KAFKA_*` — нужно задать **все** обязательные свойства, иначе дефолты образа не подхватятся.

### Что должно быть до установки

Есть:

- Docker ≥ 20.10.4 (на Windows — Docker Desktop, внутри контейнера Linux).
- Свободен порт **9092** на этой машине.
- Закрытая сеть; вход с jump/VPN.

Нет (и не нужно):

- ZooKeeper, Java на хосте (она внутри образа), веб-UI, SASL.
- Публикации 9092 наружу.
- Тега `latest`, переменных `KAFKA_ZOOKEEPER_CONNECT`.

### Этап 1. Проверяем Docker

**Что делаем:** убеждаемся, что Docker установлен и демон отвечает.

```bash
docker version
```

Успех: Client и Server есть, версия движка **≥ 20.10.4**. Если «Cannot connect to the Docker daemon» — демон не запущен.

### Этап 2. Скачиваем закреплённый образ

**Что делаем:** забираем именно **4.3.1**, не `latest`.

```bash
docker pull apache/kafka:4.3.1
```

Успех: в выводе тег `4.3.1`. Страница: https://kafka.apache.org/43/getting-started/docker/

### Этап 3. Запускаем combined-узел

**Что делаем:** один контейнер: брокер (принимает и хранит сообщения) + контроллер (метаданные, выборы лидера) в одном процессе. Официальная команда без адреса слушает все интерфейсы; на этой платформе привязываем к loopback.

```bash
docker run -d --name kafka-dev \
  -p 127.0.0.1:9092:9092 \
  apache/kafka:4.3.1
```

Успех: `docker ps` показывает `kafka-dev`, статус Up. В логах нет ошибки формата диска:

```bash
docker logs kafka-dev
```

Клиент с хоста должен видеть то имя/порт, который брокер **рекламирует** (`advertised.listeners`). Дефолт образа рассчитан на `localhost:9092` с хоста. Если клиент в другой Docker-сети — дефолта мало, смотрите Usage Guide (отдельный listener). Не копируйте устаревшие `KAFKA_ZOOKEEPER_CONNECT`.

### Этап 4. Топик, запись, чтение

**Что делаем:** создаём топик (именованный лог) с **RF=1** (одна копия: второй брокер некому быть), пишем и читаем. Утилиты лежат в контейнере, `/opt/kafka/bin/` (Docker Hub).

```bash
docker exec kafka-dev /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic quickstart-events \
  --replication-factor 1 --partitions 1

docker exec kafka-dev /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe --topic quickstart-events
```

Успех: `PartitionCount: 1`, `ReplicationFactor: 1`, есть Leader / Replicas / Isr.

```bash
echo "hello-stand" | docker exec -i kafka-dev /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 --topic quickstart-events

docker exec kafka-dev /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 --topic quickstart-events \
  --from-beginning --timeout-ms 5000
```

Успех: в выводе строка `hello-stand`.

В **клиентском коде** даже здесь: `acks=all`, явное имя топика, ключ партиции. На RF=1 это «лидер записал», не три копии.

### Этап 5. (Необязательно) Kubernetes на учёбе

Только если Docker-стенда мало и Kubernetes уже есть. Пакет **1.2.0** с [downloads](https://strimzi.io/downloads/), файл `strimzi-cluster-operator-1.2.0.yaml`, не `latest`. Kafka CR: версия **4.3.1**, 1 брокер. Combined / одна replica — только стенд.

```bash
kubectl create namespace kafka
kubectl apply -n kafka -f strimzi-cluster-operator-1.2.0.yaml
```

Имя файла — из релиза 1.2.0, не выдумывать. YAML одной replica в бой не копировать.

### Чего этот стенд не доказывает

- Выборы лидера контроллеров при отказе зала / дата-центра.
- ISR, RF=3, `min.insync.replicas=2`, нагрузку (MB/s), CPU TLS.
- Что клиент достучался «не до того» брокера за балансировщиком.
- Готовность боя. Успешный produce на одном процессе ≠ кластер.

---

## Первый запуск — URL, порт, учётка, смена пароля

Встроенного **веб-UI в дистрибутиве Apache Kafka нет**. AKHQ / Kafdrop / Conduktor — другое ПО, в этот стенд не входят. Смотреть топики — CLI выше или свой клиент.

| Что | Значение на стенде |
|---|---|
| Протокол | Kafka protocol (бинарный), не HTTP |
| Bootstrap / CLIENT | `127.0.0.1:9092` (дефолт `listeners`: `PLAINTEXT://:9092`) |
| Контроллер | В combined внутри контейнера (в примерах KRaft часто **9093**). На хост **не** публикуем |
| Аутентификация | **Нет.** PLAINTEXT, без логина. Заводской учётки нет — **менять нечего** |
| UI | Нет |

PLAINTEXT допустим **только** в закрытой сети стенда. Открытый 9092 = открытый лог.

Учебных секретов в этом запуске нет. Если позже включите SASL, пароли из примеров вендора (`admin` / `admin-secret` на странице SASL) — **только закрытый стенд**, не в git и не в бой. В бою: SASL/SCRAM-SHA-512 **поверх TLS** (SCRAM без TLS вендор считает опасным), учётки в Vault / Secret, не в манифесте.

`auto.create.topics.enable` на стенде можно оставить (удобство). Топики всё равно лучше создавать явно, как в этапе 4.

---

## Подключение к своей системе

**Протокол:** Kafka protocol на bootstrap **`host:9092`**. Клиент сначала спрашивает состав кластера у bootstrap, затем ходит **напрямую к лидеру партиции**. HAProxy «как HTTP» единственным `:9092` перед всеми брокерами ломает модель (см. `HAProxy.install.md`).

**Кто клиент на платформе:** продюсеры и консьюмеры микросервисов, воркеры **Camunda**, интеграционное API, Flink, Kafka Connect. Camunda / озеро / PostgreSQL / интеграционное API — **клиенты**, не часть Kafka.

**Что класть куда:**

| Куда | Что |
|---|---|
| Код / git | Имена топиков, `acks=all`, ключ партиции, `bootstrap.servers` стенда |
| Не в git | Пароли SCRAM, keystore (на этом стенде их нет) |
| Конфиг клиента | `bootstrap.servers=127.0.0.1:9092` (внутри Docker-сети — имя контейнера и тот listener, который брокер рекламирует) |

**Kafka не является:** базой и эталоном карточки (из лога неудобно «дай клиента по ИНН»); Camunda (процессы); RabbitMQ (очереди задач AMQP); GeoData Kafka 3.4 (чужое базовое ПО). Реестра схем и UI в дистрибутиве нет.

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Линейка 4.3, версия 4.3.1 | https://kafka.apache.org/43/ |
| Релиз 4.3.1 (25 июня 2026) | https://kafka.apache.org/blog/2026/06/25/apache-kafka-4.3.1-release-announcement/ |
| Docker: `docker pull` / `docker run`, порт 9092 | https://kafka.apache.org/43/getting-started/docker/ |
| Quick Start: format `--standalone`, топик, produce/consume | https://kafka.apache.org/43/getting-started/quickstart/ |
| Образ, `/opt/kafka/bin/`, default combined | https://hub.docker.com/r/apache/kafka |
| Usage Guide: Docker ≥ 20.10.4; env = задать все свойства | https://github.com/apache/kafka/blob/trunk/docker/examples/README.md |
| KRaft: combined vs isolated, кворум 2N+1, ~5 ГиБ контроллера | https://kafka.apache.org/43/operations/kraft/ |
| Java 17/21/25, пример кучи 6 ГиБ | https://kafka.apache.org/43/operations/java-version/ |
| 4.3 только KRaft, ZooKeeper удалён | https://kafka.apache.org/43/getting-started/upgrade/ |
| Дефолт `listeners` = `PLAINTEXT://:9092` | https://kafka.apache.org/43/configuration/broker-configs/ |
| Listeners / advertised | https://kafka.apache.org/43/security/listener-configuration/ |
| Диск, EXT4/XFS, page cache; Windows не well supported | https://kafka.apache.org/43/operations/hardware-and-os/ |
| SASL/SCRAM, TLS для SCRAM, пример `admin-secret` | https://kafka.apache.org/43/security/authentication-using-sasl/ |
| Strimzi 1.2.0, образ Kafka 4.3.1 | https://strimzi.io/downloads/ |
| Strimzi 1.2.0: версия Kafka 4.3.1 по умолчанию | https://strimzi.io/docs/operators/1.2.0/configuring |
| Правила, порты, роль в платформе | `Apache Kafka.md` |
| Термины | `Apache Kafka.info.md` |
| Схема стыковки | `Apache Kafka.shema.md` |

Порога RTT для stretch, минимума CPU/RAM учебного Docker-стенда и явного запрета NFS в Hardware and OS у Apache **нет**.
