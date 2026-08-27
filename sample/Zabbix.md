
Ресурсы:
+ Выделенная **Linux/UNIX**-машина в закрытой сети; Windows поддерживается для агента, но не заявлена как платформа Zabbix server: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
+ Для малого старта вендор приводит ориентир **2 CPU и 8 ГиБ RAM** на 1 000 метрик; это пример, а не универсальный минимум — каждая установка рассчитывается по нагрузке: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
+ Фиксированной нормы **HDD** нет. Официальный ориентир: около **90 байт на числовое значение**; пример вендора для 3 000 проверок раз в 60 секунд и хранения 30 дней — около **10,9 ГиБ истории**. К этому объёму нужны место для ОС, PostgreSQL и свободный резерв: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
+ Свободные порты: **10050/TCP** (агент), **10051/TCP** (server/proxy/trapper), **80/443 TCP** (web-интерфейс); опционально **10052/TCP** для Java gateway и **10053/TCP** для web service: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements

Необходимое ПО:
+ Zabbix **7.0.30 LTS**, официальные образы с тегом `alpine-7.0.30` или `ubuntu-7.0.30`, не `latest`: https://www.zabbix.com/download_sources · https://hub.docker.com/r/zabbix/zabbix-server-pgsql/tags?name=7.0.30
+ Для короткого стенда: Docker Engine и Docker Compose **v2 ≥ 2.24**, официальный `zabbix-docker` ветки **7.0**, файл `compose_pgsql.yaml`: https://www.zabbix.com/documentation/7.0/en/manual/installation/containers · https://github.com/zabbix/zabbix-docker/tree/7.0
+ База — PostgreSQL **13–18**. При пакетной установке frontend требует PHP **8.0–8.5** и Nginx **1.20+** либо Apache **2.4+**; в официальном Compose эти зависимости поставляются контейнерами: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements

Установка:
+ Официальный запуск в контейнерах: https://www.zabbix.com/documentation/7.0/en/manual/installation/containers
+ Официальный Quickstart: https://www.zabbix.com/documentation/7.0/en/manual/quickstart
+ Первый вход: `Admin` / `zabbix`; пароль сразу сменить, web-интерфейс и порт **10051** не публиковать в интернет: https://www.zabbix.com/documentation/7.0/en/manual/quickstart/basic_config/login · https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements

Подключение:
+ Linux-хосты подключаются через Zabbix Agent 2: passive-проверки используют **10050/TCP**, active-проверки отправляются на **10051/TCP**: https://www.zabbix.com/documentation/7.0/en/manual/quickstart/linux · https://www.zabbix.com/documentation/7.0/en/manual/concepts/agent
