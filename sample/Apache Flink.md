
Ресурсы:
+ Учебный стенд: одна **Linux**-машина x86_64/ARM64 в закрытой сети с Docker; официальный образ — `flink:2.2.1-java17`, Java 17 рекомендуется: https://hub.docker.com/_/flink и https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/java_compatibility/
+ Официальных минимумов **CPU/RAM/HDD** для Docker-стенда нет. Практический стартовый ориентир, не требование вендора: **2 vCPU, 4 ГБ RAM, 30 ГБ локального SSD**; заводские размеры процессов — JobManager **1600m** и TaskManager **1728m**: https://github.com/apache/flink/blob/release-2.2.1/flink-dist/src/main/resources/config.yaml
+ Для боя ресурсы рассчитываются по параллелизму, слотам и размеру состояния; RocksDB требует быстрого локального диска, а чекпоинты — внешнего файлового/объектного хранилища: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/memory/mem_setup/ и https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/state/checkpoints/
+ Нужное ПО: Docker Engine (минимальную версию документация Flink не фиксирует); при сборке JAR вне контейнера — Java **17**. Для Kubernetes — Flink Kubernetes Operator **1.15.0**, совместимый с Flink **2.2.x**: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/ и https://flink.apache.org/2026/05/26/apache-flink-kubernetes-operator-1.15.0-release-announcement/
+ Порты TCP: **8081** — REST API/Web UI, **6123** — внутренний RPC JobManager; Blob Server, RPC и data-порты TaskManager могут быть динамическими. Порт 8081 не публиковать в интернет: штатный REST не аутентифицирует клиента: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/config/ и https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/security/security-ssl/

Установка:
+ Официальный quickstart для Docker (JobManager + TaskManager, UI и тестовое задание): https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/

Подключение:
+ UI/REST: `http://127.0.0.1:8081`; приложения читают и пишут Kafka через Kafka Connector **5.0.0** (`flink-connector-kafka:5.0.0-2.2`), который добавляется отдельно: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/ops/rest_api/ и https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/connectors/datastream/kafka/
