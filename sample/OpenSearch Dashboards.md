
Ресурсы:
+ Linux-VM в закрытой сети; официально доступны также Docker, Kubernetes/Helm, RPM, Debian, tarball и Windows: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/
+ Версия **OpenSearch Dashboards 3.8.0** должна совпадать с OpenSearch **3.8.0**; образ `opensearchproject/opensearch-dashboards:3.8.0`, не `latest`: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/plugins/
+ Необходимое ПО: уже работающий OpenSearch **3.8.0** и Docker Engine с Compose; Node.js отдельно не нужен — в дистрибутиве 3.5+ включён **Node.js 22**: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/
+ Официальных минимумов **CPU/RAM/HDD** для Docker нет. Стартовый ориентир платформы для небольшого стенда (не требование вендора): **2 vCPU, 4 ГиБ RAM, 10 ГБ локального диска**. Данные и сохранённые дашборды находятся в OpenSearch, поэтому отдельный большой HDD/PVC для UI не нужен: https://docs.opensearch.org/latest/security/multi-tenancy/multi-tenancy-config/
+ Для Helm README рекомендует **8 ГиБ доступной памяти на ноде**, минимум **4 ГиБ**; это не лимит контейнера и не боевая смета: https://github.com/opensearch-project/helm-charts/tree/main/charts/opensearch-dashboards
+ Порты: **5601/TCP** — вход браузера в Dashboards; **9200/TCP** — исходящее подключение Dashboards к OpenSearch. UDP-портов нет; 5601 не публиковать в интернет: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/

Установка:
+ Официальный quickstart — Docker/Docker Compose рядом с уже запущенным OpenSearch: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/

Первый вход:
+ `http://127.0.0.1:5601`; пользователь `admin`, пароль задан при запуске OpenSearch через `OPENSEARCH_INITIAL_ADMIN_PASSWORD`. Учебные `kibanaserver` и отключение проверки TLS не переносить в прод: https://docs.opensearch.org/latest/security/getting-started/

Подключение:
+ Люди работают через браузер на **5601**; Dashboards подключается к OpenSearch по **9200**, а приложения обращаются к OpenSearch напрямую, не через Dashboards: https://docs.opensearch.org/latest/install-and-configure/install-dashboards/docker/
