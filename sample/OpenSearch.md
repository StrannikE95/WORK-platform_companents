Ресурсы:

- Одна **Linux-VM**. ОС: Rocky Linux 8, Alma Linux 8, Amazon Linux 2/2023, Ubuntu 24.04; также Windows Server 2019. 
  - [https://docs.opensearch.org/latest/install-and-configure/os-comp/](https://docs.opensearch.org/latest/install-and-configure/os-comp/)
- ПО: Docker Engine; образ `opensearchproject/opensearch:3.8.0`, не `latest`. 
  - [https://docs.opensearch.org/latest/version-history/](https://docs.opensearch.org/latest/version-history/)
- Официальных минимумов **CPU и размера диска нет**. Вендор: Docker Desktop **≥ 4 ГБ RAM**; Диск ноды — локальный SSD, не NFS. Ориентир: **2 vCPU, 4 ГБ RAM, 30 ГБ локального SSD**. 
  - [https://docs.opensearch.org/latest/install-and-configure/install-opensearch/index/](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/index/)
- Порты: **9200/TCP** REST, **9300/TCP** transport (узел↔узел), **9600/TCP** Performance Analyzer. **5601** — OpenSearch Dashboards, другое ПО.

Установка:

- Официальный quickstart — Docker, один контейнер `discovery.type=single-node`, тег **3.8.0**. 
  - [https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/)
  - [https://docs.opensearch.org/latest/security/configuration/demo-configuration/](https://docs.opensearch.org/latest/security/configuration/demo-configuration/)
- Проверка: `https://127.0.0.1:9200`, пользователь `admin`, пароль из переменной

Подключение:

+ 