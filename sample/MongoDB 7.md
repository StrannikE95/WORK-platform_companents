
Ресурсы:
+ Закрытый учебный стенд на Linux x86_64/ARM64; на Windows/macOS — через Docker Desktop. Официальный образ Linux поддерживает `amd64` и `arm64`, а MongoDB 5.0+ требует CPU с AVX: https://hub.docker.com/r/mongodb/mongodb-community-server/tags?name=7.0.40 и https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/
+ Полной официальной минимальной конфигурации CPU/RAM/HDD нет. Вендор указывает минимум два реальных ядра или один многоядерный CPU и нижнюю границу кэша WiredTiger 256 МБ; наш ориентир для небольшого стенда — **2 vCPU, 2 ГБ RAM, 10 ГБ локального SSD**, не расчёт для production: https://www.mongodb.com/docs/v7.0/administration/production-notes/ и https://www.mongodb.com/docs/v7.0/core/wiredtiger/
+ Данные хранить в постоянном томе `/data/db` на локальном диске; для production рекомендован XFS, NFS обычно медленнее: https://www.mongodb.com/docs/v7.0/reference/program/mongod/ и https://www.mongodb.com/docs/v7.0/administration/production-notes/
+ Нужны Docker Engine/Docker Desktop и отдельно `mongosh`; минимальные версии этих программ MongoDB не фиксирует. Образ — `mongodb/mongodb-community-server:7.0.40-ubi9-slim`, не `latest`: https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/ и https://www.mongodb.com/docs/mongodb-shell/install/
+ Свободен **27017/TCP** — клиентский порт; публиковать только на `127.0.0.1`. Порты **27018/TCP** и **27019/TCP** нужны только для ролей shard/config server и в этом quickstart не используются: https://www.mongodb.com/docs/v7.0/reference/default-mongodb-port/ и https://www.mongodb.com/docs/v7.0/core/security-mongodb-configuration/

Установка:
+ Официальный quickstart — Docker; использовать зафиксированный тег `7.0.40-ubi9-slim`, постоянный том и размер кэша меньше лимита памяти контейнера: https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/ · https://hub.docker.com/r/mongodb/mongodb-community-server/tags?name=7.0.40 · https://www.mongodb.com/docs/v7.0/reference/program/mongod/ · https://www.mongodb.com/docs/v7.0/core/wiredtiger/
+ Это одиночный стенд, не отказоустойчивый production-кластер. Лицензия Community Server — SSPL и должна быть принята юристами: https://www.mongodb.com/licensing/server-side-public-license и https://www.mongodb.com/docs/v7.0/core/replica-set-architectures/

Первое подключение:
+ Подключение выполняется `mongosh` к `127.0.0.1:27017`; готовность проверяется командой `hello`: https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-community-with-docker/
+ Проверка прав по умолчанию выключена; доступ нельзя открывать в сеть без включённой аутентификации и отдельного пользователя приложения: https://www.mongodb.com/docs/v7.0/administration/security-checklist/ и https://www.mongodb.com/docs/v7.0/tutorial/configure-scram-client-authentication/
