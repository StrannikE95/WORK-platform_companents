
Ресурсы:
+ Одна машина **Windows, macOS или Linux** в закрытой сети; для Ubuntu — **22.04+**. Camunda 8 Run предназначен только для разработки и прототипа, не для production: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/
+ Официального минимума **CPU и объёма HDD/SSD** для C8 Run нет; практический стартовый ориентир — **2 vCPU, 8 ГБ RAM, 30 ГБ локального SSD**. Из них вендор рекомендует только **8 ГБ RAM**, а sizing требует проверять нагрузкой: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/ и https://docs.camunda.io/docs/reference/supported-environments/
+ ПО: **Camunda 8 Run 8.9.17**; при необходимости внешней Java — **OpenJDK 21–25**; для моделирования — Desktop Modeler **5.46+**. Docker и Kubernetes этому quickstart не нужны: https://github.com/camunda/camunda/releases/tag/8.9.17 и https://docs.camunda.io/docs/reference/supported-environments/
+ Свободные TCP-порты: **8080** (UI/REST), **26500** (gRPC), **8086** (Connectors API), **9600** (метрики). Не публиковать их в интернет: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/

Установка:
+ Официальный quickstart — **Camunda 8 Run**; скачать архив 8.9.17 под ОС и запустить launcher: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/install-start/ и https://github.com/camunda/camunda/releases/tag/8.9.17

Подключение:
+ Operate: `http://localhost:8080/operate`; REST: `http://localhost:8080/v2`; gRPC для job workers: `localhost:26500`. Стендовая учётка `demo`/`demo` допустима только в закрытой сети: https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/
