Ресурсы:

- Отдельного инстанса нет: OpenTelemetry — API/SDK/агент **внутри процесса приложения** не `latest`: [https://opentelemetry.io/docs/what-is-opentelemetry/](https://opentelemetry.io/docs/what-is-opentelemetry/) 
  - Java: [https://opentelemetry.io/docs/zero-code/java/agent/getting-started/](https://opentelemetry.io/docs/zero-code/java/agent/getting-started/)
  - .Net: [https://opentelemetry.io/docs/zero-code/dotnet/getting-started/](https://opentelemetry.io/docs/zero-code/dotnet/getting-started/)
  - Исходящие порты к Collector: **4317/TCP** (OTLP/gRPC), **4318/TCP** (OTLP/HTTP). SDK сам не слушает.

Установка:

- Вариант 1: **SDK instrumentation** — добавляем библиотеки в приложение и конфигурируем exporter.
  - .Net: 
    - [https://opentelemetry.netlify.app/docs/languages/dotnet/exporters/](https://opentelemetry.netlify.app/docs/languages/dotnet/exporters/)
    - [https://www.nuget.org/packages/opentelemetry.exporter.opentelemetryprotocol/](https://www.nuget.org/packages/opentelemetry.exporter.opentelemetryprotocol/)
  - Java: 
    - [https://opentelemetry.io/docs/languages/java/configuration/](https://opentelemetry.io/docs/languages/java/configuration/)
- Вариант 2: **Zero-code / automatic instrumentation** — подключаем агент/автоинструментацию без изменений исходного кода.
  - .Net: 
    - [https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/config.md](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/config.md)
  - Java: 
    - [https://opentelemetry.io/uk/docs/zero-code/java/agent/getting-started/](https://opentelemetry.io/uk/docs/zero-code/java/agent/getting-started/)
    - [https://github.com/open-telemetry/opentelemetry-java-instrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation)

Подключение:

- Приложение шлёт OTLP на ближайший Collector (**4317/4318**), не напрямую во все бэкенды: [https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/](https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/)

