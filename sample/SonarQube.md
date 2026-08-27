
Ресурсы:
+ Выделенная Linux-машина (x86_64 или ARM64) в закрытой сети.
+ Docker Engine ≥ 20.10.
+ Для учебного контура: **2 CPU**, **4 ГБ RAM**, **30 ГБ** на локальном диске; всегда оставлять ≥ 10% свободного места.
+ Свободен порт **9000**. В интернет его не публиковать.
+ Для встроенного Elasticsearch: `vm.max_map_count ≥ 524288`, `fs.file-max ≥ 131072`, для пользователя `nofile ≥ 131072`, `nproc ≥ 8192`; включён `seccomp`, каталог `/tmp` доступен на запись.
+ Данные Elasticsearch хранить на локальном диске, не на NFS/SMB/NAS. Отдельный Elasticsearch/OpenSearch и JDK для Docker-установки не нужны.
+ Требования к Linux-хосту: https://docs.sonarsource.com/sonarqube-community-build/server-installation/pre-installation/linux.md
+ Требования к ресурсам и диску: https://docs.sonarsource.com/sonarqube-community-build/server-installation/server-host-requirements.md
+ Учебный образ: `sonarqube:26.8.0.126808-community`, не `latest`. Это один контейнер, не отказоустойчивый кластер.
+ Встроенная H2 допустима только для теста; для длительного стенда нужен PostgreSQL 14–18 UTF-8: https://docs.sonarsource.com/sonarqube-server/2026.1/server-installation/installing-the-database.md

Установка:
+ https://docs.sonarsource.com/sonarqube-community-build/try-out-sonarqube.md
+ https://docs.sonarsource.com/sonarqube-community-build/server-installation/from-docker-image/installation-overview.md
+ Образ и постоянные тома: https://hub.docker.com/_/sonarqube/
+ Успешный запуск: контейнер имеет статус `Up`, в журнале появилась строка `SonarQube is operational`, URL `http://127.0.0.1:9000` отвечает.

Первый вход:
+ URL: `http://127.0.0.1:9000`; логин и пароль: `admin` / `admin`.
+ После входа пароль необходимо сразу сменить и не хранить в git. Порт **9000** оставлять доступным только из закрытой сети.
+ https://docs.sonarsource.com/sonarqube-community-build/try-out-sonarqube.md

Подключение:
+ Нужен GitLab Runner с Docker executor. Сканер передаёт результаты через `SONAR_HOST_URL` и секретный `SONAR_TOKEN`; токен хранить в masked CI/CD variable, не в git.
+ Для полной истории анализа установить `GIT_DEPTH: "0"`. Community Build анализирует только основную ветку; анализ веток и merge request требует коммерческой редакции.
+ https://docs.sonarsource.com/sonarqube-community-build/devops-platform-integration/gitlab-integration/adding-analysis-to-gitlab-ci-cd.md
