
Ресурсы:
+ Одна **Linux**-VM; + Docker Engine, Docker Compose **v2**, Git и OpenSSL; 
+ Версия приложения — **Apache Superset 6.1.0**, образ `apache/superset:6.1.0`, не `latest`
    + https://github.com/apache/superset/releases/tag/6.1.0
+ Официальных минимумов **CPU, RAM и HDD нет**. Ориентир: **2 vCPU, 4 Gb RAM, 30 Gb локального SSD**
+ Docker Compose 6.1.0 поднимает 
    + **PostgreSQL 17** для метаданных 
    + **Redis 7** для кэша/очереди; 
        + https://github.com/apache/superset/blob/6.1.0/docker-compose-image-tag.yml
+ Для Kubernetes: Kubernetes-кластер и Helm **3**; официальный чарт `superset/superset` **0.21.1** (`appVersion: 6.1.0`): 
    + https://superset.apache.org/docs/installation/kubernetes

Установка:
+ Официальный quickstart — Docker Compose: https://superset.apache.org/docs/installation/docker-compose
+ Официальный способ для Kubernetes — Helm; до запуска обязательно задать собственный `SECRET_KEY`: https://superset.apache.org/docs/installation/kubernetes

Первый вход:
+ URL: `http://127.0.0.1:8088`; пример Compose использует `admin` / `admin`, пароль нужно сразу сменить

Подключение:
+ 

