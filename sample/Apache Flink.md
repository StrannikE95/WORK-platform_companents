
Ресурсы:
+ Одна **Linux**-машина x86_64/ARM64 с Docker; официальный образ — `flink:2.2.1-java17`
+ Официальных минимумов **CPU/RAM/HDD** для Docker-стенда нет. Ориентир **2 vCPU, 4 Gb RAM, 30 Gb SSD**;
+ **RocksDB**
+ Нужное ПО: Docker Engine; при сборке вне контейнера — Java **17**. 
+ Порты TCP: **8081** — REST API/Web UI, **6123** — внутренний RPC JobManager; 
    + https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/config/

Установка:
+ Официальный quickstart для Docker: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/docker/
+ Официальный quickstart для kubernetes: https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/deployment/resource-providers/standalone/kubernetes/

Подключение:
+ 


