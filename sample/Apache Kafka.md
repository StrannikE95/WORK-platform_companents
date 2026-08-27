
Ресурсы:
+ Закрытый учебный стенд: одна **Linux**-машина; нативная Windows не считается хорошо поддерживаемой платформой. На Windows использовать Docker Desktop с Linux-контейнером: https://kafka.apache.org/43/operations/hardware-and-os/
+ ПО: Docker Engine **≥ 20.10.4**, официальный образ `apache/kafka:4.3.1` (KRaft, без ZooKeeper). Java на хосте для контейнера не нужна; для запуска бинарного дистрибутива нужны Java **17, 21 или 25**: https://github.com/apache/kafka/blob/trunk/docker/examples/README.md, https://kafka.apache.org/43/operations/java-version/
+ Официального минимума **CPU/RAM/HDD** для учебного стенда Apache не устанавливает. Наш стартовый ориентир, не требование вендора: **2 vCPU, 8 ГБ RAM, 30 ГБ локального SSD/HDD**; для Kafka предпочтительнее быстрый локальный диск. Apache приводит 6 ГиБ heap лишь как пример нагруженного кластера, а остальную RAM рекомендует оставлять файловому кэшу: https://kafka.apache.org/43/operations/java-version/, https://kafka.apache.org/43/operations/hardware-and-os/
+ Объём диска подбирать по входящему потоку, сроку хранения и числу реплик; универсального официального размера нет. Данные размещать на локальной POSIX-ФС, ориентиры вендора — XFS или ext4: https://kafka.apache.org/43/operations/hardware-and-os/
+ Свободен TCP-порт **9092** для клиентов, только в закрытой сети. Порты межброкерного и KRaft-контроллерного listener-ов задаются конфигурацией и наружу не публикуются: https://kafka.apache.org/43/getting-started/docker/, https://kafka.apache.org/43/security/listener-configuration/

Установка:
+ Официальный Quick Start для Kafka **4.3.1**: загрузка бинарного дистрибутива, запуск локального KRaft-сервера, создание топика, запись и чтение тестового события: https://kafka.apache.org/43/getting-started/quickstart/
+ Более короткий стендовый способ — официальный Docker-образ `apache/kafka:4.3.1` в combined mode (брокер и контроллер одним процессом); это вариант для разработки, не боевая отказоустойчивая схема: https://kafka.apache.org/43/getting-started/docker/, https://kafka.apache.org/43/operations/kraft/
+ Порт публиковать только на loopback (`127.0.0.1:9092`) и не использовать тег `latest`; PLAINTEXT допустим лишь в изолированном стенде: https://kafka.apache.org/43/getting-started/docker/, https://kafka.apache.org/43/security/listener-configuration/

Подключение:
+ Приложения подключаются Kafka-клиентом к `127.0.0.1:9092`; встроенного HTTP API или веб-интерфейса у Kafka нет. После bootstrap клиент обращается к адресам из `advertised.listeners`, поэтому они должны быть доступны клиенту: https://kafka.apache.org/43/getting-started/quickstart/, https://kafka.apache.org/43/security/listener-configuration/
+ Для Kubernetes использовать Strimzi **1.2.0**, поддерживающий Kafka **4.3.1**; одноузловую конфигурацию не переносить в production: https://strimzi.io/downloads/, https://strimzi.io/docs/operators/1.2.0/configuring
