
Ресурсы:
+ Выделенная **Linux/UNIX**-машина: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
+ Для старта **2 CPU и 8 Gb RAM** на 1 000 метрик; **HDD** : для 3 000 проверок раз в 60 секунд и хранения 30 дней — около **10,9 Gb истории**. К этому объёму нужны место для ОС, PostgreSQL и свободный резерв
+ Свободные порты: **10050/TCP** (агент), **10051/TCP** (server/proxy/trapper), **80/443 TCP** (web-интерфейс); опционально **10052/TCP** для Java gateway и **10053/TCP** для web service

Необходимое ПО:
+ Zabbix **7.0.30 LTS**, официальные образы с тегом `alpine-7.0.30` или `ubuntu-7.0.30`, не `latest`: 
    + https://www.zabbix.com/download_sources 
    + https://hub.docker.com/r/zabbix/zabbix-server-pgsql/tags?name=7.0.30
+ Для теста: Docker Engine + Docker Compose **v2 ≥ 2.24**, официальный `zabbix-docker` ветки **7.0**, файл `compose_pgsql.yaml`:    
    + https://www.zabbix.com/documentation/7.0/en/manual/installation/containers
    + https://github.com/zabbix/zabbix-docker/tree/7.0
+ База — PostgreSQL **13–18**. 


Установка:
+ Официальный запуск в контейнерах: https://www.zabbix.com/documentation/7.0/en/manual/installation/containers
+ Официальный Quickstart: https://www.zabbix.com/documentation/7.0/en/manual/quickstart
+ Первый вход: `Admin` / `zabbix`; пароль сразу сменить, web-интерфейс и порт **10051**

Подключение:
+ 
