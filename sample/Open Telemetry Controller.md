
Ресурсы:
+ Выделенная **Linux**-машина в закрытой сети. Официальное имя — **OpenTelemetry Collector**, не Controller: https://opentelemetry.io/docs/collector/
	+ Официального списка ОС нет; для quickstart — Docker Engine (версию вендор не фиксирует) и образ **`otel/opentelemetry-collector:0.159.0`**, не `latest`. Состав модулей зависит от distribution: https://opentelemetry.io/docs/collector/quick-start/ · https://github.com/open-telemetry/opentelemetry-collector-releases/releases/tag/v0.159.0
	+ Официального минимума **CPU/RAM/HDD нет** (размер считается по нагрузке). Ориентир учебного контейнера: **1 vCPU, 1 ГБ RAM, 5 ГБ локального SSD**; это не хранилище, диск — под конфиг и журналы: https://opentelemetry.io/docs/collector/scaling/
	+ Свободны **4317/TCP** (OTLP/gRPC) и **4318/TCP** (OTLP/HTTP). В quickstart также **55679/TCP** (zPages). Публиковать только на `127.0.0.1`, в интернет не открывать: https://opentelemetry.io/docs/collector/quick-start/ · https://opentelemetry.io/docs/specs/otlp/

Установка:
+ https://opentelemetry.io/docs/collector/quick-start/

Подключение:
+ SDK/агенты шлют OTLP на **4317/4318** этого Collector; дальше Collector экспортирует в Tempo/Prometheus/журнальный бэкенд: https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/
