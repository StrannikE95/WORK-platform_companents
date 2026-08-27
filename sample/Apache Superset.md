
Ресурсы:
+ Учебный стенд: одна **Linux**-машина в закрытой сети; Windows официально не поддерживается. Нужны Docker Engine, Docker Compose **v2**, Git и OpenSSL; минимальные версии этих утилит вендор не фиксирует: https://superset.apache.org/docs/installation/docker-compose
+ Версия приложения — **Apache Superset 6.1.0**, образ `apache/superset:6.1.0`, не `latest`: https://github.com/apache/superset/releases/tag/6.1.0
+ Официальных минимумов **CPU, RAM и HDD нет**. Неофициальный стартовый ориентир для небольшого учебного стенда: **2 vCPU, 4 ГБ RAM, 30 ГБ локального SSD**; это не требование вендора и не расчёт для production. Упомянутые в документации **16 ГБ RAM** относятся к интерактивной сборке frontend, а не к запуску готового release-образа: https://superset.apache.org/docs/installation/docker-compose
+ Docker Compose 6.1.0 поднимает PostgreSQL **17** для метаданных и Redis **7** для кэша/очереди; для production нужны внешние PostgreSQL/Redis, а SQLite не рекомендуется: https://github.com/apache/superset/blob/6.1.0/docker-compose-image-tag.yml · https://superset.apache.org/docs/configuration/configuring-superset
+ Свободен **8088/TCP** для UI/API; публиковать его в интернет нельзя. PostgreSQL **5432/TCP** и Redis **6379/TCP** наружу не открывать: https://github.com/apache/superset/blob/6.1.0/docker-compose-image-tag.yml · https://superset.apache.org/docs/installation/docker-compose
+ Для Kubernetes: Kubernetes-кластер и Helm **3**; официальный чарт `superset/superset` **0.21.1** (`appVersion: 6.1.0`): https://superset.apache.org/docs/installation/kubernetes · https://artifacthub.io/packages/helm/superset/superset/0.21.1

Установка:
+ Официальный quickstart для учебного стенда — Docker Compose; этот способ не поддерживается как production-развёртывание: https://superset.apache.org/docs/installation/docker-compose
+ Официальный способ для Kubernetes — Helm; до запуска обязательно задать собственный `SECRET_KEY`: https://superset.apache.org/docs/installation/kubernetes

Первый вход:
+ URL: `http://127.0.0.1:8088`; пример Compose использует `admin` / `admin`, пароль нужно сразу сменить, а порт оставить только в закрытой сети: https://superset.apache.org/docs/installation/docker-compose

Подключение:
+ SQL-источники подключаются через **Settings → Data: Database Connections**; нужный Python-драйвер базы должен быть добавлен в образ, а учётная запись источника — только для чтения: https://superset.apache.org/docs/databases/
