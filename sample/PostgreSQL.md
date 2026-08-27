
Ресурсы:
+ Закрытая **Linux**-машина с Docker Engine; PostgreSQL и официальный образ не задают минимальную версию ОС или Docker, поэтому использовать поддерживаемые стабильные версии дистрибутива и Docker. Образ `postgres:18.6`, не `latest`: https://hub.docker.com/_/postgres
+ Вендор не устанавливает минимум CPU/RAM/HDD для готового сервера: его requirements относятся к сборке из исходников. Практический старт для небольшого учебного стенда: **2 vCPU, 2 ГБ RAM, 20 ГБ локального SSD**; затем подбирать по нагрузке: https://www.postgresql.org/docs/18/install-requirements.html
+ Постоянный Docker volume монтировать в `/var/lib/postgresql`; в PostgreSQL 18 данные образа лежат в `/var/lib/postgresql/18/docker`: https://hub.docker.com/_/postgres
+ Открывать только **5432/TCP**, для локального стенда привязать к `127.0.0.1`; это порт клиентов и репликации, TLS использует тот же порт: https://www.postgresql.org/docs/18/runtime-config-connection.html

Установка:
+ Официальный quickstart — раздел **How to use this image** для `postgres:18.6`; обязательны собственный `POSTGRES_PASSWORD` и постоянный том, `POSTGRES_HOST_AUTH_METHOD=trust` не использовать: https://hub.docker.com/_/postgres
+ Успешный запуск: контейнер работает, а `SELECT version()` возвращает **PostgreSQL 18.6**: https://www.postgresql.org/docs/18/functions-info.html

Первое подключение:
+ Хост `127.0.0.1`, порт **5432**, стартовая роль `postgres`, пароль задаётся при запуске и заводского пароля нет; приложению создать отдельную роль без прав суперпользователя: https://hub.docker.com/_/postgres
+ Строка подключения: `postgresql://USER:PASSWORD@127.0.0.1:5432/DB`; формат URI описан здесь: https://www.postgresql.org/docs/18/libpq-connect.html#LIBPQ-CONNSTRING-URIS

Подключение:
+ Приложения подключаются драйвером PostgreSQL по **TCP 5432**; порт не публиковать в интернет, а в бою включить TLS и SCRAM-SHA-256, поскольку TLS по умолчанию выключен, а MD5 в PostgreSQL 18 устарел: https://www.postgresql.org/docs/18/runtime-config-connection.html#GUC-SSL · https://www.postgresql.org/docs/18/auth-password.html
