
Ресурсы:
+ Один Unix-хост в закрытой сети: **Linux**, macOS либо Windows через **WSL**; нативный Windows quickstart не заявлен ([официальный Getting Started](https://backstage.io/docs/getting-started/)).
+ Для **Backstage 1.54.0** нужны Node.js **22 или 24**, Yarn **4.4.1**, `git`, `curl`/`wget` и GNU build tools; Docker для основного запуска `yarn start` не обязателен ([Getting Started](https://backstage.io/docs/getting-started/), [релиз 1.54.0](https://github.com/backstage/backstage/releases/tag/v1.54.0)).
+ Официальный минимум для quickstart: **RAM ≥ 6 ГБ**, свободный диск **≥ 20 ГБ**; минимум CPU вендор **не задаёт**. Ориентир для небольшого учебного стенда: **2 vCPU, 6 ГБ RAM, 20 ГБ HDD/SSD**; CPU — наша стартовая оценка, не гарантия производительности ([Getting Started](https://backstage.io/docs/getting-started/)).
+ Свободны **3000/TCP** (frontend) и **7007/TCP** (backend); наружу их не публиковать — Backstage рассчитан на защищённый контур ([Getting Started](https://backstage.io/docs/getting-started/), [Threat Model](https://backstage.io/docs/overview/threat-model)).
+ Quickstart использует SQLite в памяти, Guest-вход и один процесс — это не боевая схема; для контейнерного/боевого запуска нужны свой образ приложения, PostgreSQL и не-Guest-аутентификация ([SQLite → PostgreSQL](https://backstage.io/docs/tutorials/switching-sqlite-postgres/), [Docker deployment](https://backstage.io/docs/deployment/docker)).

Установка:
+ Официальный quickstart (`npx @backstage/create-app@latest`, затем `yarn start`): https://backstage.io/docs/getting-started/
+ После создания приложение следует зафиксировать на **1.54.0**, а не оставлять произвольный `latest`: https://backstage.io/docs/getting-started/keeping-backstage-updated

Первый запуск:
+ UI: `http://localhost:3000`; backend: `http://localhost:7007`. Заводского логина/пароля нет: quickstart использует Guest без пароля ([Getting Started](https://backstage.io/docs/getting-started/), [Guest provider](https://backstage.io/docs/auth/guest/provider)).
