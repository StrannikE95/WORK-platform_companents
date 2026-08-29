
Ресурсы:
+ Одна VM **Linux** в закрытой сети; для Ubuntu — **22.04+**. Для тест: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/
+ Официального минимума **CPU и объёма HDD/SSD** нет; Ориентир — **2 vCPU, 8 ГБ RAM, 30 ГБ локального SSD**. 
    + https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/ 
    + https://docs.camunda.io/docs/reference/supported-environments/
+ ПО: 
    + **Camunda 8 Run 8.9.17**; при необходимости внешней Java — **OpenJDK 21–25**; 
    + Desktop Modeler **5.46+**. 
+ Свободные TCP-порты: **8080** (UI/REST), **26500** (gRPC), **8086** (Connectors API), **9600** (метрики). 
+ https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/

Установка:
+ Официальный quickstart — **Camunda 8 Run**; скачать архив 8.9.17 под ОС и запустить launcher: 
    + https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/install-start/ 
    + https://github.com/camunda/camunda/releases/tag/8.9.17

Подключение:
+ Operate: `http://localhost:8080/operate`; REST: `http://localhost:8080/v2`; gRPC для job workers: `localhost:26500`. Стендовая учётка `demo`/`demo` допустима только в закрытой сети: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/
