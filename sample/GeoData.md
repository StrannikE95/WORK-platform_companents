
Ресурсы:
+ On-premise GeoData разворачивается в закрытой сети на одной площадке: 
    + базовое ПО — на **Ubuntu 22.04 LTS**, 
    + прикладные модули **4.0.1** (`workflow` **4.0.2**) — в Kubernetes **1.24.1**. Совместимость с более новым Kubernetes не заявлена: https://docs.datageo.ru/ 
+ Норм CPU/RAM/HDD для всей системы и прикладных подов нет. Ориентир — **минимумы** компонентов из руководства: 
    + Cassandra — от **8 ГиБ RAM / 100 ГиБ HDD** на узел (от 3 узлов); 
    + Kafka — **4 CPU / 8 ГиБ / 100 ГиБ** на узел (3 узла); 
    + Elasticsearch — **8 ГиБ / 100 ГиБ** на узел (рекомендуется 3); 
    + PostgreSQL/PostGIS — от **4 ГиБ / 2 ГиБ**; 
    + Redis — **4 CPU / 4 ГиБ / 20 ГиБ**; 
    + Keycloak — **2 CPU / 4 ГиБ / 250 ГиБ**. 
+ Нужное ПО фиксированных версий: 
    + Cassandra **4.1.0**
    + Kafka **3.4.0** + ZooKeeper 
    + Elasticsearch **8.6.2**
    + PostgreSQL **15.2** + PostGIS **3.3**
    + Redis **7.0.8**
    + OpenStack Swift **2.29.2**
    + Nexus **3.49.0**
    + HAProxy **2.4**
    + Keepalived **2.2.4**;
    + В главе установки Keycloak — **21.1.2** и OpenJDK **11** 
+ Для клиента нужен браузер с WebGL 1.0: Chrome ≥ 81, Яндекс.Браузер ≥ 76, Opera ≥ 12 или Safari ≥ 13: https://docs.datageo.ru/
+ Основные порты: **443/TCP** (UI), **9042** (Cassandra), **9092** (Kafka; дополнительно служебные порты ZooKeeper), **9200** (Elasticsearch), **5432** (PostgreSQL), **6379** (Redis), а также согласованные HTTPS-порты Keycloak и Swift API. 

Установка:
+ Руководство администратора: https://docs.datageo.ru/

Подключение:
+ Пользователи входят по HTTPS через `adminui`/`clientui` и Keycloak; источники и интеграции настраиваются внутри GeoData. 
