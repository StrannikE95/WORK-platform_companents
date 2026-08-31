
Ресурсы:
+ Выделенная **Linux**-машина в закрытой сети.
	+ Официального списка ОС нет; для Docker нужен хост с Docker Engine (минимальную версию вендор не фиксирует): https://github.com/prometheus/alertmanager/blob/v0.34.0/README.md
	+ Образ **`quay.io/prometheus/alertmanager:v0.34.0`**, не `latest`: https://github.com/prometheus/alertmanager/releases/tag/v0.34.0
	+ Официального минимума **CPU/RAM/HDD нет**. Ориентир учебного контейнера: **1 vCPU, 1 ГБ RAM, 5 ГБ локального SSD**: https://prometheus.io/docs/alerting/latest/alertmanager/
	+ Свободен **9093/TCP** (UI/API). **9094/TCP+UDP** — только gossip HA; для одного процесса не слушать. В интернет не публиковать: https://prometheus.io/docs/alerting/latest/https/

Установка:
+ https://github.com/prometheus/alertmanager/blob/v0.34.0/README.md

Подключение:
+ Prometheus отправляет алерты на **9093** этого процесса: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alertmanager_config
