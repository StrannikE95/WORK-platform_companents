
Ресурсы:
+ ОС: одна **Linux**-машина в закрытой сети; для Docker-варианта Grafana не задаёт конкретный дистрибутив и его версию — нужен хост с поддерживаемым Docker Engine: https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/
+ ПО: Docker Engine (минимальную версию Docker документация Grafana не устанавливает), современный браузер с JavaScript и образ **`grafana/grafana:13.2.0`**, не `latest`: https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/ и https://grafana.com/grafana/download/13.2.0?platform=docker&edition=oss
+ Официальный минимум Grafana: **1 CPU**, **512 МиБ RAM**; для небольшого стенда официальный ориентир Small — **2 CPU**, **2–4 ГиБ RAM**, **10–20 ГБ SSD** под БД Grafana. Это стартовый ориентир, а не расчёт под неизвестную нагрузку: https://grafana.com/docs/grafana/latest/setup-grafana/installation/
+ Свободен **3000/TCP** для UI/API; публиковать его только на localhost или во внутреннюю сеть. **9094/TCP+UDP** нужен лишь между репликами при HA alerting и для этого одноконтейнерного quickstart не требуется: https://grafana.com/docs/grafana/latest/setup-grafana/sign-in-to-grafana/ и https://grafana.com/docs/grafana/latest/alerting/set-up/configure-high-availability/

Установка:
+ Официальный quickstart — один Docker-контейнер OSS **13.2.0** с постоянным volume `/var/lib/grafana`; встроенная SQLite подходит для такого стенда, но не рекомендуется для production или нескольких реплик: https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/ и https://grafana.com/docs/grafana/latest/setup-grafana/installation/

Первый вход:
+ URL: `http://127.0.0.1:3000`; начальные логин/пароль **`admin` / `admin`**, пароль сразу сменить: https://grafana.com/docs/grafana/latest/setup-grafana/sign-in-to-grafana/

Подключение:
+ Для первой проверки подключить Prometheus через **Connections → Add new connection**; Grafana визуализирует данные источника и сама ряды метрик не хранит: https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/ и https://grafana.com/docs/grafana/latest/fundamentals/
