Ресурсы:

- 1 **Linux-VM** рядом с OpenSearch **3.8.0**. ОС вендор для Docker-пути не фиксирует.
  - **Docker Engine**. Отдельный Node.js не ставить: в дистрибутиве 3.5+ уже **Node.js 22**.
  - Версия UI **3.8.0** Образ `opensearchproject/opensearch-dashboards:3.8.0`, не `latest`.
  - CPU/RAM/HDD контейнера в Docker-гайде **нет**. Ориентир стенда: **2 vCPU, 4 Gb RAM, 10 Gb** локального диска
  - Цифры вендора есть только у Helm: на ноде рекомендуется **8 GiB** available, минимум **4 GiB**;
  - Порты: **5601/TCP** — браузер (только localhost / закрытая сеть); исходящий **9200/TCP** к OpenSearch.

Установка:

- Официальный quickstart — Docker рядом с живым OpenSearch.
- Первый вход: `http://127.0.0.1:5601`, логин `admin`, пароль из `OPENSEARCH_INITIAL_ADMIN_PASSWORD`.

Подключение:

- Люди — браузер на **5601**. Приложения ходят в OpenSearch на **9200** напрямую, не через Dashboards.

Ссылки:

- [https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/](https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/) — Docker-quickstart, порты 5601/9200, образ
- [https://docs.opensearch.org/latest/version-history/](https://docs.opensearch.org/latest/version-history/) — релиз 3.8.0 (4 августа 2026)
- [https://docs.opensearch.org/latest/install-and-configure/install-dashboards/plugins/](https://docs.opensearch.org/latest/install-and-configure/install-dashboards/plugins/) — версия UI = версия OpenSearch, не `latest`

