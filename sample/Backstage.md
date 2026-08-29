
Ресурсы:
+ Один Unix-хост в закрытой сети: **Linux**, macOS либо Windows через **WSL**; 
    + https://backstage.io/docs/getting-started/
    + Для **Backstage 1.54.0** нужны Node.js **22 или 24**, Yarn **4.4.1**, `git`, `curl`/`wget` и GNU build tools; Docker для основного запуска `yarn start` не обязателен 
    + https://github.com/backstage/backstage/releases/tag/v1.54.0
    + Официальный минимум: **RAM ≥ 6 ГБ**, свободный диск **≥ 20 ГБ**; CPU вендор не задаёт, наш ориентир ~ **2 vCPU**; 
+ Свободны **3000/TCP** (frontend) и **7007/TCP** (backend); наружу их не публиковать
    + https://backstage.io/docs/overview/threat-model
+ Для теста Quickstart использует SQLite в памяти, Guest-вход и один процесс; для контейнерного/боевого запуска нужны свой образ приложения, PostgreSQL и не-Guest-аутентификация 
    + https://backstage.io/docs/tutorials/switching-sqlite-postgres/
    + https://backstage.io/docs/deployment/docker

Установка:
+ Официальный quickstart 
    + https://backstage.io/docs/getting-started/
+ После создания приложение следует зафиксировать на **1.54.0**, а не оставлять произвольный `latest
    + https://backstage.io/docs/getting-started/keeping-backstage-updated

Первый запуск:
+ UI: `http://localhost:3000`; backend: `http://localhost:7007`. Заводского логина/пароля нет: quickstart использует Guest без пароля 
    + https://backstage.io/docs/getting-started/ 
    + https://backstage.io/docs/auth/guest/provider
