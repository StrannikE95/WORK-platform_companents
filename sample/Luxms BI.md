
Ресурсы:
+ Одна Linux-VM в закрытой сети; для quickstart подходят Astra Linux SE 1.7/1.8, РЕД ОС 7.3/8.x, Альт СП 10, Rocky Linux 9 или MosOS Arbat 15.5: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning
+ Для теста/демонстрации при объёме около 1 млн строк: **4 CPU** (от 2 ГГц), **6 ГБ RAM**, **200 ГБ HDD/SSD** полезного места со скоростью чтения от 150 МБ/с. Это оценка стенда, не расчёт production-нагрузки: https://luxmsbi.ru/docs/12.1.0/overviews/general-overview/tech-info
+ Данные PostgreSQL размещать на локальном диске; quickstart использует `/data/pgdata`, отдельный диск при необходимости монтируется в `/data`: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup

ПО:
+ Luxms BI — коммерческая линейка **12**; точный patch-релиз определяется договором и доступным клиентским репозиторием, а не выбирается произвольно: https://luxmsbi.ru/docs/ · https://luxmsbi.ru/products/relizy/
+ Для quickstart нужны подключённые репозитории **BI** и **BI 3rd-party packages**, пакет `bi-setup` и запускаемый им Ansible-сценарий: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup
+ Ядро работает на **PostgreSQL 15 или 17**; обязательны расширения `http 1.7`, `plv8 3.2`, `redis_pubsub 1.0`, `redis_fdw 1.0`: https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/postgresql
+ KeyDB, NATS, OpenJDK и остальные зависимости устанавливаются пакетами quickstart; их точные patch-версии для конкретной поставки страница quickstart не фиксирует, поэтому использовать версии из клиентского репозитория этой сборки, не подставлять свои: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup · https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-rocky

Порты:
+ **80/TCP** — веб-интерфейс `http://<IP|FQDN>`; доступ только из закрытой сети/VPN, не из интернета: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup
+ **5432/TCP** — PostgreSQL, **6379/TCP** — KeyDB; для одноузлового стенда оставлять внутренними: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-check · https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-rocky
+ **4222/TCP** — клиенты NATS, **8888/TCP** — WebSocket NATS; **8222/TCP** нужен только при включении мониторинга NATS: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/appendix/nats

Установка:
+ Использовать официальный одноузловой **quickstart** на Ansible; он предназначен для разработки и небольших ненагруженных решений и не обеспечивает отказоустойчивость: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup
+ Нужны лицензия и аутентифицированный доступ к репозиторию поставщика: https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning
