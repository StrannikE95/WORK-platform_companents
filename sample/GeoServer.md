Ресурсы:

- Для официального quickstart: одна Linux-машина либо Docker Desktop; 
  - **Docker Engine**, отдельные Java и Tomcat не нужны — они входят в образ `docker.osgeo.org/geoserver:2.24.4`: [https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html](https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html)
  - Для WAR-варианта нужны **Java 11** и **Tomcat 8.5 или 9**; 
    - [https://github.com/geoserver/geoserver/wiki/Release-Schedule](https://github.com/geoserver/geoserver/wiki/Release-Schedule)
    - [https://docs-archive.geoserver.org/2.24.x/en/user/production/java.html](https://docs-archive.geoserver.org/2.24.x/en/user/production/java.html)
- Официальных минимумов **CPU/RAM/HDD нет**. Ориентир: **2 vCPU, 4 Gb RAM, 10 Gb локального SSD**; 
- Свободен **8080/TCP** только на localhost/в закрытой сети; снаружи **443/TCP** через reverse proxy. **8443/TCP** нужен только при HTTPS внутри контейнера
- Версия **2.24.4** завершила поддержку в августе 2024. [https://github.com/geoserver/geoserver/wiki/Release-Schedule](https://github.com/geoserver/geoserver/wiki/Release-Schedule)

Установка:

- Официальный quickstart — Docker с образом `docker.osgeo.org/geoserver:2.24.4`, внешним томом `/opt/geoserver_data` и публикацией **8080** на `127.0.0.1`: [https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html](https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html)
- Проверка: `http://127.0.0.1:8080/geoserver`; заводская учётка `admin` / `geoserver`, пароль сразу сменить: [https://docs-archive.geoserver.org/2.24.x/en/user/gettingstarted/web-admin-quickstart/index.html](https://docs-archive.geoserver.org/2.24.x/en/user/gettingstarted/web-admin-quickstart/index.html)

Подключение:

- PostGIS подключается как отдельное хранилище данных по **5432/TCP**: [https://docs-archive.geoserver.org/2.24.x/en/user/data/database/postgis.html](https://docs-archive.geoserver.org/2.24.x/en/user/data/database/postgis.html)

