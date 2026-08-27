
Ресурсы:
+ Для официального quickstart: одна Linux-машина либо Docker Desktop в закрытой сети; **Docker Engine** (минимальная версия в руководстве 2.24 не указана), отдельные Java и Tomcat не нужны — они входят в образ `docker.osgeo.org/geoserver:2.24.4`: https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html
+ Для WAR-варианта нужны **Java 11** и **Tomcat 8.5 или 9**; Java 17 для линии 2.24 экспериментальна, Tomcat 10+ не подходит: https://github.com/geoserver/geoserver/wiki/Release-Schedule · https://docs-archive.geoserver.org/2.24.x/en/user/production/java.html
+ Официальных минимумов **CPU/RAM/HDD нет**. Ориентир для небольшого учебного стенда: **2 vCPU, 4 ГБ RAM, 10 ГБ локального SSD**; это стартовая рекомендация, не норма вендора и не расчёт для production. Каталог данных следует вынести во внешний том; объём тайлов и данных рассчитывать по своей нагрузке: https://docs-archive.geoserver.org/2.24.x/en/user/production/container.html · https://docs-archive.geoserver.org/2.24.x/en/user/datadirectory/setting.html
+ Свободен **8080/TCP** только на localhost/в закрытой сети; снаружи production обычно использует **443/TCP** через reverse proxy. **8009/AJP** наружу не открывать; **8443/TCP** нужен только при HTTPS внутри контейнера: https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html · https://docs-archive.geoserver.org/2.24.x/en/user/production/misc.html
+ Версия **2.24.4** завершила поддержку в августе 2024; без отдельного решения ИБ её нельзя считать готовой для production. Не использовать `latest` или nightly-тег `2.24.x`: https://github.com/geoserver/geoserver/wiki/Release-Schedule · https://geoserver.org/release/2.24.4/ · https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html

Установка:
+ Официальный quickstart — Docker с образом `docker.osgeo.org/geoserver:2.24.4`, внешним томом `/opt/geoserver_data` и публикацией **8080** только на `127.0.0.1`: https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html
+ Проверка: `http://127.0.0.1:8080/geoserver`; заводская учётка `admin` / `geoserver`, пароль сразу сменить: https://docs-archive.geoserver.org/2.24.x/en/user/gettingstarted/web-admin-quickstart/index.html

Подключение:
+ Клиенты используют WMS/WFS/WCS и WMTS по HTTP(S); PostGIS подключается как отдельное хранилище данных, обычно по **5432/TCP**: https://docs-archive.geoserver.org/2.24.x/en/user/services/wms/reference.html · https://docs-archive.geoserver.org/2.24.x/en/user/services/wfs/reference.html · https://docs-archive.geoserver.org/2.24.x/en/user/data/database/postgis.html
