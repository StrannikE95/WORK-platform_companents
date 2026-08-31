Ресурсы:
+ 1 **Linux-VM** в закрытой сети, рядом с уже живым OpenSearch **3.8.0**. ОС вендор для Docker-пути не фиксирует.
	+ **Docker Engine**. Отдельный Node.js не ставить: в дистрибутиве 3.5+ уже **Node.js 22**.
	+ Версия UI **3.8.0** (релиз 4 августа 2026) = кластеру. Образ `opensearchproject/opensearch-dashboards:3.8.0`, не `latest`.
	+ CPU/RAM/HDD контейнера в Docker-гайде **нет**. Ориентир стенда: **2 vCPU, 4 Gb RAM, 10 Gb** локального диска (сохранённые объекты в OpenSearch, большой том UI не нужен).
	+ Цифры вендора есть только у Helm: на ноде рекомендуется **8 GiB** available, минимум **4 GiB**; дефолт чарта **100m / 512M** — не смета боя.
	+ Порты: **5601/TCP** — браузер (только localhost / закрытая сеть); исходящий **9200/TCP** к OpenSearch. UDP нет.

Установка:
+ Официальный quickstart — Docker рядом с живым OpenSearch.
+ Первый вход: `http://127.0.0.1:5601`, логин `admin`, пароль из `OPENSEARCH_INITIAL_ADMIN_PASSWORD`. Учебные `kibanaserver` / `verificationMode: none` в прод не копировать.

Подключение:
+ Люди — браузер на **5601**. Приложения ходят в OpenSearch на **9200** напрямую, не через Dashboards.

Ссылки:
+ https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/ — Docker-quickstart, порты 5601/9200, образ
+ https://docs.opensearch.org/latest/install-and-configure/install-dashboards/ — установка, Node.js 22 в 3.5+, браузеры
+ https://docs.opensearch.org/latest/version-history/ — релиз 3.8.0 (4 августа 2026)
+ https://docs.opensearch.org/latest/install-and-configure/install-dashboards/plugins/ — версия UI = версия OpenSearch, не `latest`
+ https://docs.opensearch.org/latest/security/multi-tenancy/multi-tenancy-config/ — объекты UI в индексах `.kibana*`, не на диске пода
+ https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/README.md — 8 GiB available / минимум 4 GiB на ноде
+ https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/values.yaml — дефолт чарта 100m / 512M
+ https://docs.opensearch.org/latest/security/getting-started/ — вход `admin`, пароль `OPENSEARCH_INITIAL_ADMIN_PASSWORD`; demo `kibanaserver`
