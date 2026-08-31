
Ресурсы:
+ Отдельной машины нет: OpenTelemetry — API/SDK/агент **внутри процесса приложения**, не хранилище: https://opentelemetry.io/docs/what-is-opentelemetry/
	+ ОС — как у приложения. Для Java-агента: **Java 8+** (Linux/Windows/macOS). Агент **`opentelemetry-javaagent` 2.31.1** (SDK **1.65.0**), не `latest`: https://opentelemetry.io/docs/zero-code/java/agent/ · https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/tag/v2.31.1
	+ Официального минимума **CPU/RAM/HDD нет** (зависит от приложения). Ориентир накладных расходов агента: **+0.5 vCPU, +256–512 МБ RAM**, отдельный диск не нужен (очередь в памяти процесса): https://opentelemetry.io/docs/zero-code/java/agent/performance/
	+ Исходящие порты к Collector: **4317/TCP** (OTLP/gRPC), **4318/TCP** (OTLP/HTTP). SDK сам не слушает. В интернет не публиковать: https://opentelemetry.io/docs/specs/otlp/

Установка:
+ https://opentelemetry.io/docs/zero-code/java/agent/getting-started/

Подключение:
+ Приложение шлёт OTLP на ближайший Collector (**4317/4318**), не напрямую во все бэкенды: https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/
