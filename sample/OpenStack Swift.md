Ресурсы:

- Одна **Linux-VM** в закрытой сети (официальный SAIO: все процессы на одной машине, не кластер).
  - ОС (вендор): **Ubuntu 24.04 LTS**, CentOS Stream 9, Fedora, openSUSE. Windows-сервер не заявлен: [https://docs.openstack.org/swift/2026.1/development_saio.html](https://docs.openstack.org/swift/2026.1/development_saio.html)
  - Версия: **OpenStack Swift 2.37.3**, серия **2026.1**. Не `latest`, не 2.38.x, не GeoData 2.29.2: [https://docs.openstack.org/releasenotes/swift/2026.1.html](https://docs.openstack.org/releasenotes/swift/2026.1.html)
  - ПО: Python **≥ 3.7**; gcc, Git, pip, Memcached, rsync, **sqlite3**, xfsprogs.
  - RAM/диск SAIO: **≥ 2 Gb RAM**, **≥ Gb ГиБ** места. Числа **CPU нет**; ориентир VM: **2 vCPU**. Данные — локальный **XFS**
  - Порты: клиентам только **8080/TCP** (стенд — localhost; бой — обычно **443** на LB). Внутри: **6200** object, **6201** container, **6202** account (на SAIO 6210–6242), **873** rsync, **11211**: [https://docs.openstack.org/swift/2026.1/deployment_guide.html](https://docs.openstack.org/swift/2026.1/deployment_guide.html)

Установка:

+ 

Подключение:

+ 