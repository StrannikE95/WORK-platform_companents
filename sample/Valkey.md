
Ресурсы:
+ Linux-машина (x86_64 или ARM64) в закрытой сети; Windows нативно не поддерживается, для разработки — WSL. [Официальная установка](https://valkey.io/topics/installation/) · [образы](https://valkey.io/download/)
+ ПО: Docker Engine (минимальную версию Valkey не фиксирует), образ **`valkey/valkey:9.1.1`**, не `latest`. [Docker quickstart](https://valkey.io/topics/installation/) · [релиз 9.1.1](https://github.com/valkey-io/valkey/releases/tag/9.1.1)
+ Официальных минимумов **CPU, RAM и HDD нет**. Практический старт для малого тестового стенда: **1 vCPU, 1 ГБ RAM, 10 ГБ локального SSD**; это ориентир, не требование вендора. RAM рассчитывать по рабочему набору с запасом, диск — по RDB/AOF; сетевые NFS/NAS не использовать. [Память](https://valkey.io/topics/memory-optimization/) · [persistence](https://valkey.io/topics/persistence/) · [диск и benchmark](https://valkey.io/topics/benchmark/)
+ Порты: **6379/TCP** — клиенты; **16379/TCP** — Cluster bus при стандартном клиентском порте; **26379/TCP** — Sentinel. Для одиночного Docker-стенда нужен только 6379, публиковать на `127.0.0.1`, не в интернет. [Установка](https://valkey.io/topics/installation/) · [Cluster](https://valkey.io/topics/cluster-tutorial/) · [Sentinel](https://valkey.io/topics/sentinel/)

Установка:
+ Официальный quickstart для выбранного способа — Docker, один контейнер Valkey 9.1.1; это стенд без HA. https://valkey.io/topics/installation/

Подключение:
+ Приложения подключаются по TCP/RESP к **6379**; подходят перечисленные Valkey-клиенты и совместимые Redis OSS 7.2+ клиенты. https://valkey.io/clients/
