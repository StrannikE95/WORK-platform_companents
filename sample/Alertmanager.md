
Ресурсы:
+ Выделенная **Linux**-машина в закрытой сети.
	+ Официального списка ОС нет; для Docker нужен хост с Docker Engine (минимальную версию вендор не фиксирует): https://github.com/prometheus/alertmanager/blob/v0.34.0/README.md
	+ Образ **`quay.io/prometheus/alertmanager:v0.34.0`**, не `latest`
	+ Официального минимума **CPU/RAM/HDD нет**. Ориентир: **1 vCPU, 1 ГБ RAM, 5 ГБ SSD**
	+ Свободен **9093/TCP** (UI/API). **9094/TCP+UDP** — только gossip HA; для одного процесса не слушать. 

Установка:
+ https://github.com/prometheus/alertmanager/blob/v0.34.0/README.md

Подключение:
+ Prometheus отправляет алерты на **9093** этого процесса: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alertmanager_config
