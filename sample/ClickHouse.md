
Ресурсы:
+ Одна Linux-машина в закрытой сети; CPU x86-64-v3 или ARMv8.2-A. Docker **≥ 20.10.10**, образ `clickhouse/clickhouse-server:26.7.5.10`, не `latest`: https://clickhouse.com/docs/get-started/setup/self-managed/docker
+ Официального минимума CPU и размера диска для одного контейнера нет. Практический стартовый ориентир учебного стенда: **2 vCPU, 8 ГиБ RAM, 30 ГБ локального SSD**; это не расчёт для production. Вендор рекомендует не менее **8 ГиБ RAM**, SSD и подбор ресурсов по нагрузке: https://clickhouse.com/docs/guides/oss/best-practices/sizing-and-hardware-recommendations
+ Постоянный том для `/var/lib/clickhouse`; лимит открытых файлов `nofile=262144`. Требования Docker и тома: https://clickhouse.com/docs/get-started/setup/self-managed/docker
+ Свободны **8123/TCP** (HTTP) и **9000/TCP** (native), публиковать только в закрытой сети. Для TLS используются **8443** и **9440**; кластерные **9009**, **9181/9281**, **9234** снаружи не открывать: https://clickhouse.com/docs/concepts/features/security/network-ports

Установка:
+ Официальный quickstart: https://clickhouse.com/docs/getting-started/quick-start/oss
+ Официальная установка Docker: https://clickhouse.com/docs/get-started/setup/self-managed/docker
