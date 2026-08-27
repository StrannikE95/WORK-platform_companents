
Ресурсы:
+ Одна Linux-VM в закрытой сети для официального стенда SAIO; это все процессы Swift на одной машине, не отказоустойчивый кластер: https://docs.openstack.org/swift/2026.1/development_saio.html
+ Версия: **OpenStack Swift 2.37.3**, серия OpenStack **2026.1 (Gazpacho)**: https://docs.openstack.org/releasenotes/swift/2026.1.html
+ ОС из SAIO: **Ubuntu 24.04 LTS**, **CentOS Stream 9**, Fedora или openSUSE; Windows-сервер в quickstart не заявлен: https://docs.openstack.org/swift/2026.1/development_saio.html
+ Требуемое ПО: Python **≥ 3.7** для пакета Swift 2.37.3; Git, pip, rsync, Memcached, SQLite, XFS tools и компилятор. Минимальные версии последних компонентов документация не фиксирует: https://pypi.org/project/swift/2.37.3/ и https://docs.openstack.org/swift/2026.1/development_saio.html
+ CPU: официального минимума нет; практический ориентир для небольшого стенда — **2 vCPU**. RAM и диск по SAIO: **≥ 2 ГиБ RAM**, **≥ 40 ГиБ HDD/SSD**; данные — на локальном **XFS**: https://docs.openstack.org/swift/2026.1/development_saio.html
+ Для боя официальной универсальной сметы CPU/RAM/HDD нет: её считают по нагрузке; Swift рекомендует **3 реплики**, поэтому полезный объём требует примерно тройной сырой ёмкости плюс свободное место для перемещения данных: https://docs.openstack.org/swift/2026.1/deployment_guide.html
+ Порты: клиентский proxy **8080/TCP**; внутренние **6200/TCP** (object), **6201/TCP** (container), **6202/TCP** (account), **873/TCP** (rsync), **11211/TCP** (Memcached). Клиентам открывать только proxy, в бою обычно через TLS/LB на **443**: https://docs.openstack.org/swift/2026.1/deployment_guide.html

Установка:
+ Официальный quickstart при наличии вариантов — **SAIO на Ubuntu 24.04 LTS**; на той же странице есть варианты пакетов для CentOS Stream 9, Fedora и openSUSE: https://docs.openstack.org/swift/2026.1/development_saio.html
+ SAIO использует `tempauth`, loopback XFS и localhost; эту схему нельзя переносить в бой, где нужны несколько proxy, локальные диски storage-нод и Keystone: https://docs.openstack.org/swift/2026.1/development_saio.html и https://docs.openstack.org/swift/2026.1/overview_auth.html

Подключение:
+ Приложения работают через Swift REST или S3 API (`s3api`) на том же proxy; для S3 `s3api` ставится в pipeline перед auth: https://docs.openstack.org/swift/2026.1/middleware.html
+ Swift не полностью повторяет AWS S3: bucket lifecycle, bucket policy и object tagging не поддерживаются; совместимость клиента проверить до внедрения: https://docs.openstack.org/swift/2026.1/s3_compat.html
