
Ресурсы:
+ Закрытый Kubernetes-кластер; поддерживаемые версии Kubernetes — **1.32–1.36**. Для production HashiCorp рекомендует **Linux**, но официальной матрицы поддерживаемых дистрибутивов и их версий на этих страницах нет: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm · https://developer.hashicorp.com/vault/docs/concepts/production-hardening
+ Нужны **Helm ≥ 3.6** и настроенный доступ к Kubernetes; использовать Helm-чарт `hashicorp/vault` **0.34.1** с Vault Community **2.0.4**, не `latest`: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm · https://github.com/hashicorp/vault/releases/tag/v2.0.4
+ Официального абсолютного минимума CPU/RAM/HDD нет. Стартовый ориентир HashiCorp для каждого узла small-кластера: **2–4 CPU**, **8–16 ГБ RAM**, **100+ ГБ SSD** (не магнитный HDD), от 3000 IOPS и 75 МБ/с; размер уточнять нагрузочным тестом: https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture
+ Свободные порты: **8200/TCP** — API Vault и присоединение узлов, **8201/TCP** — Raft и обмен между узлами; при внешнем балансировщике также **443/TCP**. 8201 открывать только между узлами, весь трафик шифровать TLS: https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture

Установка:
+ Предпочтительный официальный quickstart — Helm в режиме **HA + Integrated Storage (Raft)**; после установки кластер отдельно инициализируют, распечатывают и присоединяют остальные поды. Дефолтный standalone-режим чарта не подходит для production: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft · https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm

Подключение:
+ Клиенты обращаются к HTTP(S) API на **8200**, а балансировщик проверяет `/v1/sys/health`; порт **8201** клиентам не нужен: https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture · https://developer.hashicorp.com/vault/api-docs/system/health
