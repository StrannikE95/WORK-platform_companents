
Ресурсы:
+ Одна Linux-машина (x86_64 или ARM64) в закрытой сети; учебный Tempo запускается монолитом, а `backend: local`
    + https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/locally/linux/
+ Официальный стартовый ориентир для монолита: **4 CPU, 4–8 ГБ RAM**; при размещении рядом Grafana и Collector — **от 16 ГБ RAM**. Фиксированного требования к диску нет (зависит от ingest и retention), наш ориентир ~ **30 ГБ SSD** с отдельным постоянным Docker volume и контролировать заполнение: 
    + https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/plan/size/
+ Необходимое ПО: 
    + Docker Engine (версия не указана), образ **`grafana/tempo:3.0.3`**, не `latest`; 
        + https://github.com/grafana/tempo/releases/tag/v3.0.3
+ Свободные TCP-порты: **3200** — HTTP API, **4317** — OTLP gRPC, **4318** — OTLP HTTP. На учебном хосте публиковать только на `127.0.0.1`; встроенной аутентификации у Tempo нет: 
    + https://grafana.com/docs/tempo/latest/configuration/manifest/ 
    + https://grafana.com/docs/tempo/latest/operations/authentication/

Установка:
+ Официальный quickstart: https://grafana.com/docs/tempo/latest/getting-started/
+ Короткий вариант для стенда: один контейнер `grafana/tempo:3.0.3` с `-target=all`, конфигурацией OTLP и постоянным локальным volume; Kafka и object storage для монолитного теста не нужны. Это не боевая схема: https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/deploy/locally/linux/

Подключение:
+ Приложения передают трейсы по OTLP через OpenTelemetry Collector на **4317/4318**, Grafana подключается к Tempo по **3200**; 
    + Настройка Collector: https://grafana.com/docs/tempo/latest/set-up-for-tracing/instrument-send/set-up-collector/otel-collector/
+ Проверка приёма и поиск тестового трейса в Grafana
    + https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/test/test-monolithic-local/

