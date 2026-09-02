
Ресурсы:
+ Одна Linux-VM; OS: Astra Linux SE 1.7/1.8, РЕД ОС 7.3/8.x, Альт СП 10, Rocky Linux 9 или MosOS Arbat 15.5 
    + https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning
+ Для теста при объёме около 1 млн строк: **4 CPU** (от 2 ГГц), **6 ГБ RAM**, **200 ГБ HDD/SSD**
    + https://luxmsbi.ru/docs/12.1.0/overviews/general-overview/tech-info
+ Данные PostgreSQL размещать на локальном диске; quickstart использует `/data/pgdata`, отдельный диск при необходимости монтируется в `/data`
    + https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup

ПО:
+ Подключённые репозитории **BI** и **BI 3rd-party packages**
    + https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup
+ **PostgreSQL 15 или 17** + обязательны расширения `http 1.7`, `plv8 3.2`, `redis_pubsub 1.0`, `redis_fdw 1.0` 
    + https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/postgresql
+ **KeyDB** (fork Redis), 
+ **NATS** (легковесная система обмена сообщениями с открытым исходным кодом) 
    + https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/appendix/nats
+ OpenJDK и остальные зависимости устанавливаются пакетами quickstart; 

Порты:
+ **80/TCP** — веб-интерфейс `http://<IP|FQDN>`
+ **5432/TCP** — PostgreSQL, **6379/TCP** — KeyDB;
+ **4222/TCP** — клиенты NATS, **8888/TCP** — WebSocket NATS

Установка:
+ Официальный одноузловой **quickstart**: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup
+ Нужны лицензия и аутентифицированный доступ к репозиторию поставщика: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning
