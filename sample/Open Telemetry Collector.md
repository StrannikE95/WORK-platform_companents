Ресурсы:

- Выделенная **Linux**-машина
  - Официального списка ОС нет; 
  - Docker Engine (версию вендор не фиксирует) и образ `otel/opentelemetry-collector:0.159.0`, не `latest`. 
  - Официального **CPU/RAM/HDD - нет** . 
    - Ориентир для VM: **1 vCPU, 1 ГБ RAM, 5 ГБ локального SSD**; это не хранилище, диск — под конфиг и журналы
    - Ориентир для VM: ???
  - Свободны **4317/TCP** (OTLP/gRPC) и **4318/TCP** (OTLP/HTTP). В quickstart также **55679/TCP** (zPages). П

Установка:

- [https://opentelemetry.io/docs/collector/quick-start/](https://opentelemetry.io/docs/collector/quick-start/)
- Если приложения в Kubernetes — Kubernetes + Helm
- Если приложения на Linux VM - на этой же VM в Docker

Подключение:

+ 