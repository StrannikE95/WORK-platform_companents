
Ресурсы:
+ Одна Linux-машина в закрытой сети.
+ Docker Engine; локальный POSIX-диск под TSDB. NFS, включая AWS EFS, официально не поддерживается.
+ Свободен порт **9090**. В интернет его не публиковать.
+ Образ: `quay.io/prometheus/prometheus:v3.13.2`, не `latest`. Это один процесс с локальной TSDB, а не отказоустойчивый кластер.
+ Минимальные CPU и RAM вендор не устанавливает. Наша стартовая рекомендация для небольшого тестового стенда: **2 vCPU, 4 ГБ RAM и 30 ГБ локального SSD**.
+ Начать со стандартного срока хранения **15 дней**. Диск рассчитывается по нагрузке: срок хранения × число сэмплов в секунду × 1–2 байта; оставлять свободными 15–20% тома.
+ Если памяти не хватает, запросы заметно замедляются или диск заполняется выше 80%, сначала увеличить ресурсы VM/тома либо сократить retention. Переустановка Prometheus для увеличения CPU и RAM не нужна; данные должны оставаться в отдельном Docker volume.
+ Версия и загрузки: https://prometheus.io/download/
+ Требования к хранилищу: https://prometheus.io/docs/prometheus/latest/storage/

Установка:
+ Docker-образ, подключение `prometheus.yml` и постоянного тома `/prometheus`: https://prometheus.io/docs/prometheus/3.13/installation/
+ Минимальная конфигурация со съёмом собственных метрик: https://prometheus.io/docs/introduction/first_steps/
+ Успешный запуск: контейнер имеет статус `Up`, `http://127.0.0.1:9090/-/ready` отвечает **200**, запрос `up{job="prometheus"}` возвращает **1**.

Первый вход:
+ URL: `http://127.0.0.1:9090`. Заводского логина и пароля у Prometheus нет.
+ Доступ к порту оставлять только из закрытой сети. Для внешнего доступа нужны TLS и аутентификация: https://prometheus.io/docs/prometheus/latest/configuration/https/

Подключение:
+ Prometheus сам опрашивает HTTP-эндпоинты **`/metrics`** приложений и exporter-ов; цели добавляются в `scrape_configs`.
+ Приложение должно быть доступно из контейнера Prometheus. `localhost` внутри контейнера указывает на сам контейнер, а не на хост.
+ Клиентские библиотеки: https://prometheus.io/docs/instrumenting/clientlibs/
+ Подключение Linux-хостов через node_exporter: https://prometheus.io/docs/guides/node-exporter/
