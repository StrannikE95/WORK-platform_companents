
Ресурсы:
+ Spring Cloud Gateway **5.0.3** — библиотека в приложении Spring Boot **4.0.8**, поезд Spring Cloud **2025.1.3**; готового сервера или официального Docker-образа нет: https://spring.io/projects/spring-cloud · https://github.com/spring-cloud/spring-cloud-gateway/releases/tag/v5.0.3
+ Официальных минимумов **CPU, RAM и HDD** проект не устанавливает. Практический старт для закрытого учебного стенда: **2 vCPU, 2 ГБ RAM, 5 ГБ HDD** под JDK, приложение и кэш Maven; это не норматив для production, ресурсы подбираются нагрузочным тестом: https://docs.spring.io/spring-boot/4.0.8/system-requirements.html · https://docs.spring.io/spring-cloud-gateway/reference/
+ Официального списка поддерживаемых ОС нет. Для quickstart подойдёт **Windows, Linux или macOS** с **JDK 21** (допустимо Java **17–26**) и **Maven ≥ 3.6.3**; также поддерживаются Gradle **8.14+ / 9.x**: https://docs.spring.io/spring-boot/4.0.8/system-requirements.html
+ Нужен стартер `spring-cloud-starter-gateway-server-webflux`; маршруты 5.x задаются в `spring.cloud.gateway.server.webflux.routes`: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html
+ Порты: вход приложения — **8080/TCP** по умолчанию (оставить на localhost для стенда); Actuator использует тот же порт, если отдельный management-порт не задан; исходящий доступ — к портам HTTP(S) целевых сервисов и **443/TCP** к Spring Initializr/Maven-репозиторию при сборке: https://docs.spring.io/spring-boot/4.0.8/appendix/application-properties/index.html · https://docs.spring.io/spring-boot/4.0.8/reference/actuator/endpoints.html

Установка:
+ Официальный quickstart — сгенерировать Maven-проект через **Spring Initializr**, зафиксировать Spring Boot **4.0.8**, BOM Spring Cloud **2025.1.3** и добавить Gateway Server WebFlux; не использовать `latest`: https://spring.io/projects/spring-cloud · https://start.spring.io/
+ Запуск приложения: `mvn spring-boot:run`: https://docs.spring.io/spring-boot/4.0.8/maven-plugin/run.html

Подключение:
+ Клиенты обращаются по HTTP к `http://127.0.0.1:8080`; Gateway проксирует запросы на URI маршрутов. В production порт шлюза открывают только для Ingress/HAProxy/WAF, а Actuator — только для мониторинга и операторов: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/how-it-works.html · https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html
