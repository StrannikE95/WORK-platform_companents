# GeoServer 2.24.4 — схемы устройства

Связанные документы: правила — `GeoServer.md`; установка — `GeoServer.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код SLD не рисуем.

Допущения: stretch data dir / NFS на 2–3 ЦОДа **нет**; классический WAR/Docker OSGeo, **не** GeoServer Cloud и **не** GeoData (ООО «АйТи Гео»); линия **EOL август 2024**; нагрузка GetMap не замерена.

---

## 1. Контекст

GeoServer — **OGC-фасад** (WMS/WFS/WMTS) над store. Не SoT геометрии, не Kafka, не платформа GeoData.

```mermaid
flowchart LR
  CLI["UI / ГИС-клиенты"]
  GS["GeoServer 2.24.4"]
  PG["PostGIS / озеро контуров"]
  KF["Kafka"]
  GD["GeoData IT Geo"]

  CLI -->|"WMS WFS WMTS"| GS
  GS -->|"JDBC / файлы"| PG
  GS -.->|"не протокол карты"| KF
  GS -.->|"другой продукт"| GD
```

Упал store — карта пустая при живом Tomcat. Запись геометрии в проде — ваши сервисы в БД, не интернет-**WFS-T**.

**Критично:** 2.24.4 (18 июня 2024) закрывал **CVE-2024-36401 (RCE)**. После июня 2024 линейка **не чинится**. Считать готовой к гособмену без решения ИБ **нельзя**. Тег Docker `2.24.x` в manual — **nightly**, не прод.

---

## 2. Контейнеры (актив в одном ЦОДе)

В ядре **нет** кворума. «Кластер» доки — несколько JVM за proxy и **одна** правда каталога.

```mermaid
flowchart TB
  ING["Ingress :443\nPROXY_BASE_URL"]
  subgraph dc["ЦОД-1 актив"]
    G1["JVM / WAR :8080"]
    G2["JVM"]
    DD["GEOSERVER_DATA_DIR\nкаталог XML"]
    GWC["GWC blob / gwc-s3"]
  end
  STORE["PostGIS"]

  ING --> G1
  ING --> G2
  G1 --> DD
  G2 --> DD
  G1 --> GWC
  G2 --> GWC
  G1 --> STORE
  G2 --> STORE
```

Порт приложения **8080**; снаружи 443. Между нодами **нет** своего бинарного протокола как у Kafka. Официального оператора K8s под WAR 2.24 **нет**.

Каталог: не 3 реплики на одном **RWO**. Общий RWX/NFS **внутри ЦОДа** — после замера lock; через город — порча XML. Предпочтительно иммутабельный data dir + один писатель.

**Сильное:** падение одного пода — GetMap с остальных, если store жив. **Слабое:** три реплики без одной правды каталога = три сервера.

---

## 3. Компоненты внутри JVM

```mermaid
flowchart TB
  subgraph jvm["Один процесс GeoServer"]
    OWS["WMS / WFS / WCS"]
    ADM["Admin / REST"]
    CF["control-flow\nлимиты на инстанс"]
    SEC["security/ в data dir"]
  end

  CAT["Каталог слоёв"]
  TILE["Тайлы GWC"]

  OWS --> CAT
  OWS --> TILE
  ADM --> CAT
  CF --> OWS
  SEC --> ADM
```

| Компонент | Для чего настраивать |
|---|---|
| Data directory | Внешний в проде (*highly recommended*). Потеря тома = потеря публикации |
| GWC | Свой каталог на каждом поде = «мигание» тайлов |
| control-flow | Stable. Ориентир модуля: ~2× ядер параллельных GetMap **на инстанс** — старт, не смета. Кластер не видит |
| JMS / JDBCConfig | Community, experimental, нет патчей после EOL. Данные слоёв по JMS **не** едут |

Java 11 — опорный runtime 2.24. Java 17 в доке — *experimental*. Tomcat 8.5/9, не Tomcat 10.

---

## 4. Поток: GetMap

```mermaid
sequenceDiagram
  participant C as Клиент
  participant P as Ingress
  participant G as GeoServer
  participant T as GWC
  participant S as PostGIS

  C->>P: GetMap / WMTS
  P->>G: :8080
  alt тайл есть
    G->>T: hit
    T-->>C: PNG
  else miss
    G->>S: вектор / растр
    S-->>G: геометрия
    G->>T: положить
    G-->>C: картинка
  end
```

Смена в PostGIS сама тайл не обновит — нужен сброс GWC. Capabilities без `PROXY_BASE_URL` отдают внутренний `http://pod:8080`.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    N2["≥2 JVM anti-affinity"]
    CAT["Одна правда каталога"]
    RO["JDBC read-only"]
    CF["control-flow на каждой"]
  end

  subgraph dr["Между ЦОДами"]
    COPY["Копия data dir + promote PostGIS"]
  end

  inside -->|"падение пода"| OK["карта жива"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если забыть |
|---|---|
| Pin **2.24.4** | Nightly `2.24.x` не прод |
| Не WFS-T | Клиент пишет в эталон |
| CSRF whitelist | Не `GEOSERVER_CSRF_DISABLED` |
| Бэкап data dir + PostGIS | Тайлы пересчитаешь; каталог — нет |

Community JMS multicast в Kubernetes обычно **не** живёт. Не два механизма синка каталога сразу.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Ось"]
  Q --> M["GetMap"]
  Q --> B["Подложка"]
  Q --> F["WFS объекты"]
  Q --> R["Растр"]

  M --> M1["Ещё JVM + control-flow"]
  B --> B1["GWC seed, не 64G heap"]
  F --> F1["Лимиты GetFeature + GIST\nвектору лишняя RAM почти не помогает"]
  R --> R1["Heap + пирамиды"]
```

Цифр «ядер под терабайты» нет. `-Xmx756M` в таблице доки — иллюстрация команд. Limit K8s **выше** heap (JAI, native).

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    LIVE["JVM + data dir + доступ к PostGIS"]
  end
  subgraph dc2["ЦОД-2"]
    CL["Клиенты → 443 ЦОД-1\nхолодная копия каталога"]
  end
  subgraph dc3["ЦОД-3"]
    B3["Бэкап / второй DR\nне третий писатель"]
  end
  LIVE -.->|"не NFS каталога"| CL
  LIVE --> B3
```

Третий ЦОД не делает третий writer XML. GeoServer Cloud / Helm camptocamp — **другой** дистрибутив.

**Сильное:** каталог не шарится через город. **Слабое:** падение ЦОД-1 = нет WMS у всех; EOL — дыры после 2.24.4 апстрим не закроет.

---

## 8. Безопасность (ручки на той же схеме)

Сменить `admin`/`geoserver` до любой маршрутизации. Demo `topp:states` снять. XXE: `ENTITY_RESOLUTION_ALLOWLIST`, unrestricted **выкл**. CORS — явный origin, не `*`. Status не светить env (CVE-2024-34696). Сессия **Secure**. Админку часто `GEOSERVER_CONSOLE_DISABLED` на читателях. Лицензия GeoServer — **GPL**.

Источники: `GeoServer.md`. Порога RTT для shared data dir у проекта **нет**.
