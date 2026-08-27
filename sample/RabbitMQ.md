
Ресурсы ([официальный production checklist](https://www.rabbitmq.com/docs/production-checklist)):
+ Одна 64-битная Linux-машина в закрытой сети; официальный Docker quickstart также допускает Docker Desktop на Windows/macOS ([загрузка RabbitMQ](https://www.rabbitmq.com/docs/download), [официальный образ](https://hub.docker.com/_/rabbitmq)).
+ Для учебного контейнера официальных минимумов CPU/RAM/HDD нет. Стартовый ориентир платформы: **2 vCPU, 4 ГиБ RAM, 20 ГБ локального SSD**; это не оценка ёмкости. Для production вендор указывает минимум **4 CPU и 4 ГиБ RAM на ноду**, а объём диска требует определять нагрузочным тестом ([production checklist](https://www.rabbitmq.com/docs/production-checklist)).
+ Нужен Docker Engine; минимальную версию Docker вендор не фиксирует. Использовать образ **`rabbitmq:4.3.5-management`**, не `latest`; он уже содержит совместимый Erlang **27.x** ([Docker quickstart](https://www.rabbitmq.com/docs/download), [образ и теги](https://hub.docker.com/_/rabbitmq), [матрица Erlang](https://www.rabbitmq.com/docs/which-erlang)).
+ Для выбранного стенда свободны **5672/TCP** (AMQP) и **15672/TCP** (Management UI/API), оба только на `127.0.0.1`; кластерные **4369/TCP** и **25672/TCP** наружу не публиковать ([сетевые порты](https://www.rabbitmq.com/docs/networking)).

Установка ([официальный Docker quickstart](https://www.rabbitmq.com/docs/download)):
+ Выбран официальный быстрый запуск одного контейнера `rabbitmq:4.3.5-management`; это учебная нода, не отказоустойчивый кластер и не шаблон production ([Docker quickstart](https://www.rabbitmq.com/docs/download), [production checklist](https://www.rabbitmq.com/docs/production-checklist)).
+ Образ, переменные начального пользователя, постоянный том и публикация портов описаны на странице официального образа ([Docker Hub](https://hub.docker.com/_/rabbitmq)).
+ Успешный старт проверяется `rabbitmq-diagnostics ping`, версия брокера должна быть **4.3.5** ([diagnostics](https://www.rabbitmq.com/docs/man/rabbitmq-diagnostics.8), [релиз 4.3.5](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.3.5)).

Первый вход ([Management Plugin](https://www.rabbitmq.com/docs/management)):
+ UI: `http://127.0.0.1:15672`; заводской `guest` разрешён только с loopback, поэтому для стенда создать отдельного пользователя и не хранить пароль в git ([Management Plugin](https://www.rabbitmq.com/docs/management), [контроль доступа](https://www.rabbitmq.com/docs/access-control)).

Подключение ([сетевой справочник](https://www.rabbitmq.com/docs/networking)):
+ Приложения подключаются по AMQP к `127.0.0.1:5672`; в production использовать TLS на **5671/TCP**, отдельную учётку на приложение, publisher confirms и manual consumer acknowledgements ([TLS](https://www.rabbitmq.com/docs/ssl), [подтверждения](https://www.rabbitmq.com/docs/confirms)).
