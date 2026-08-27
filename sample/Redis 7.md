
Ресурсы:
+ Один изолированный Linux-хост x86_64/ARM64; Windows/macOS — только через Docker Desktop с Linux-контейнерами. Официальный образ: `redis:7.4.11`, не `latest`: https://hub.docker.com/_/redis
+ Нужен Docker Engine или Docker Desktop; минимальную версию Docker документация Redis Community 7.4 не устанавливает, поэтому использовать поддерживаемый актуальный выпуск: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/
+ Официального минимума CPU/RAM/HDD для Redis Community 7.4 нет. Практический ориентир для небольшого учебного стенда: **1 vCPU, 512 МБ RAM, 1 ГБ локального SSD**; данные находятся в RAM, а `maxmemory` по умолчанию не задан: https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
+ Размер RAM считать по рабочему набору с запасом на репликацию и снимки; размер диска — по RDB/AOF и сроку хранения. Для persistence использовать локальный SSD, не NFS/NAS: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/ и https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/
+ Свободные TCP-порты: **6379** — клиенты и репликация; **26379** — Sentinel; **16379** — Cluster bus при клиентском порте 6379. Для одиночного стенда нужен только 6379, публиковать его в интернет нельзя: https://github.com/redis/redis/blob/7.4/redis.conf и https://redis.io/docs/latest/operate/oss_and_stack/management/security/

Установка:
+ Официальный Docker quickstart: https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/

Подключение:
+ Клиенты подключаются по RESP к `127.0.0.1:6379`; заводской пользователь `default` не имеет пароля, поэтому доступ оставлять на loopback либо настраивать ACL и TLS: https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/ и https://redis.io/docs/latest/operate/oss_and_stack/management/security/
+ Официальные клиентские библиотеки для Java, Python, .NET и других языков: https://redis.io/docs/latest/develop/clients/
