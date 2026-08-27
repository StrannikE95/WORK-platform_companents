
Ресурсы:
+ 1 Linux-VM в закрытой сети; Docker должен иметь не менее **4 ГБ RAM**.
  + Docker Engine и локальный постоянный том для `/usr/share/opensearch/data`. NFS не использовать как диск ноды.
  + На Linux-хосте, в том числе внутри VM Docker Desktop: `vm.max_map_count ≥ 262144`.
+ Образ: `opensearchproject/opensearch:3.8.0`, не `latest`. 
+ JVM heap задавать с одинаковыми `Xms` и `Xmx`, ориентир вендора — около половины RAM; Ресурсов вендор не устанавливает. **2 vCPU, 4 ГБ RAM и 30 ГБ локального SSD**.
+ Свободные порты:
  + 443	Панели мониторинга OpenSearch в сервисе AWS OpenSearch с шифрованием при передаче данных (TLS)
  + 5601	Панели мониторинга OpenSearch
  + 9200	REST API OpenSearch
  + 9300	Связь и транспорт между узлами (внутренний), межкластерный поиск
  + 9600	Анализатор производительности
+ Требования к хосту: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/

Установка:
+ Docker: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/
  + Перед запуском установить `vm.max_map_count=262144`. Начиная с OpenSearch 2.12, для demo-конфигурации обязателен собственный пароль `OPENSEARCH_INITIAL_ADMIN_PASSWORD`; без него контейнер не стартует.
+ Helm: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/helm/

Первый вход:
+ URL API: `https://127.0.0.1:9200`; логин `admin`, пароль задан при запуске. Заводского пароля для 3.8.0 нет.
+ https://docs.opensearch.org/latest/security/configuration/demo-configuration/

Подключение:
+ Приложения подключаются по REST/HTTPS к порту **9200**. Для каждого сервиса - отдельного пользователь с минимальными правами;
+ Поток из Kafka пишет в OpenSearch через отдельный consumer, Kafka Connect или Data Prepper;
+ Для Prod: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/operator/
