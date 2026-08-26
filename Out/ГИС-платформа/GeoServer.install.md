# GeoServer 2.24.4 — установка и конфигурирование

Связанный документ (глоссарий, data dir, GWC, почему так): `GeoServer.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного каталога (общий data dir / NFS) на несколько ЦОДов **не делаем**: file-lock и RTT дают битый XML, не HA. Это **не** платформа GeoData (ООО «АйТи Гео»).

Версия: **GeoServer 2.24.4** (18 июня 2024, GeoTools 30.4, GeoWebCache 1.24.4). Образ: `docker.osgeo.org/geoserver` — пин **`2.24.4`**, не тег **`2.24.x`** (в User Manual 2.24 это **nightly/SNAPSHOT**, *not suitable for production*). Если тега 2.24.4 в реестре OSGeo нет — свой образ из **WAR 2.24.4** на Tomcat 9.  
Документация линии: https://docs-archive.geoserver.org/2.24.x/en/user/

---

## Предупреждение ИБ (прочитать до установки)

Линия **2.24.x**: stable сентябрь 2023, maintenance апрель 2024, **EOL — август 2024**. На дату этого файла линейка **два года без официальных патчей**. 2.24.4 закрывал в том числе **CVE-2024-36401 (RCE, Critical)**. Всё, что нашли **после** июня 2024, проект в 2.24.x **не чинит**. Считать 2.24.4 «готовым к проду с гособменом» без отдельного решения ИБ **нельзя**. Документ описывает **запрошенную** версию; поддерживаемая линия на август 2026 — другой документ (Stable/Maintenance на geoserver.org).

---

## Допущения этой инструкции

1. **Stretch запрещён.** Активный GeoServer (JVM + data dir + доступ к PostGIS) — **в одном ЦОДе**. Другие ЦОДы — клиенты на 443 и **DR** (копия data dir + promote PostGIS). Общий RWX/NFS data dir на несколько ЦОДов — риск split-brain / порчи каталога.
2. Классический WAR/Docker OSGeo, **не** GeoServer Cloud (другой продукт, свой Helm) и не GeoServer Enterprise (GeoCat).
3. Официального оператора Kubernetes под WAR 2.24 **нет**.
4. Dev — изолированная сеть. Java 11 — опорный runtime линии 2.24; Java 17 в доке 2.24 — *experimental*.
5. Нагрузки (GetMap/s) нет — нет цифры heap. Store — PostGIS; WFS-T в проде выкл.
6. Для 2 ЦОДов: актив в ЦОД-1, DR в ЦОД-2. Для 3 ЦОДов: то же + ЦОД-3 бэкап/второй DR. Третий ЦОД **не** третий писатель каталога.
7. Community JMS/JDBCConfig в проде **не** считаем официальным HA (experimental, нет патчей после EOL).
8. Лицензия GeoServer — GPL (не Apache 2.0).

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** слой из PostGIS, SLD, GetMap/GetFeature. **Не** цель: отказ ЦОДа и ёмкость тайлов.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 8080 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

Проверить наличие тега в реестре, затем:

```bash
docker pull docker.osgeo.org/geoserver:2.24.4
docker run -d --name geoserver-dev \
  -p 127.0.0.1:8080:8080 \
  -v geoserver-data:/opt/geoserver_data \
  docker.osgeo.org/geoserver:2.24.4
```

Привязка к `127.0.0.1` обязательна. UI: `http://127.0.0.1:8080/geoserver`. Дефолт **`admin` / `geoserver`** — сменить сразу. Пустой data dir образ может заполнить **sample**; в прод sample уносят.

Глава Docker User Manual 2.24 показывает `2.24.x` — **не** копировать в прод и лучше не на препрод.

Расширения — zip **той же** 2.24.4. На стенде имеет смысл `control-flow` (stable), не community JAR с nightly.

На Kubernetes Dev: Deployment **1** реплика, PVC **RWO**, Ingress в изолированном namespace. **Не** `replicas: 3` с этого YAML в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Нод | 1 | Нет требования пережить выкат |
| Каталог | один volume | Не учим NFS locking |
| GWC | локальный | Нет 10k клиентов |
| Пароль | не `geoserver`, сеть закрыта | Дефолт в доке открытым текстом |
| Community cluster | не ставить | Иначе отладка JMS вместо GetMap |
| WFS-T | выкл, если не цель стенда | Привычка записи через GeoServer |

Чего **не** упрощать: слой живёт в **store**, не «в GeoServer»; `PROXY_BASE_URL`, если уже есть Ingress; плагин = версия сервера; nightly ≠ 2.24.4.

### Проверка Dev

1. GetCapabilities / GetMap. About/статус — **2.24.4**.
2. Удалите PostGIS — карта пустая при живом поде.
3. 8080 не с корпоративной сети без фильтра: линейка EOL, RCE в прошлом уже были.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Быстрый путь Docker | Нет модели отказа, нет общего кэша |
| Дешёво показывает SLD на вашей геометрии | Успех на одном поде ≠ RWX через город |
| | EOL: даже стенд не светить наружу |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода внутри ЦОДа** (вторая JVM за LB, если каталог решён). Отказ **ЦОДа с активным GeoServer** = нет карты, пока DR (data dir + PostGIS). HA PostGIS — `PostgreSQL.install.md`, не этот файл.

### Почему не stretch

В ядре **нет** кворума. «Кластер» в production-доке — несколько узлов за proxy и **одна** правда каталога. Общий data dir на RWX/NFS между ЦОДами: латентность lock XML, два писателя = порча. JMS не возит **данные слоёв**. Три реплики на одном RWO не стартуют.

### Топология

**Внутри активного ЦОДа (ЦОД-1):**

- ≥ 2 JVM (Deployment) за Ingress, anti-affinity по ноде **этого** ЦОДа;
- схема каталога **без** шаринга через город:
  - **предпочтительно:** иммутабельный data dir (выкат пайплайном) + ноды только читают; админка/REST записи — с bastion на **один** писатель;
  - либо 2 пода + **один** RWO не бывает: либо 1 писатель, либо RWX **только внутри ЦОДа** после замера lock/IOPS;
- PostGIS read-only учётка; WFS **Basic** (без транзакций);
- GWC: общее хранилище **в ЦОДе** или **gwc-s3** (stable); свой каталог на каждом поде = «мигание» тайлов;
- `GEOSERVER_DATA_DIR` снаружи приложения (`/opt/geoserver_data` в Docker 2.24);
- `PROXY_BASE_URL` = публичный HTTPS;
- **control-flow** на каждой ноде (лимиты **на инстанс**, не на кластер);
- образ/WAR **2.24.4**;
- админка: часто `GEOSERVER_CONSOLE_DISABLED=true` на читателях; `/web` не в интернет.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: активные ноды + data dir + (локальный) доступ к PostGIS. ЦОД-2: клиенты ходят на 443 ЦОД-1; холодная копия data dir + реплика PostGIS | Нет WMS/WFS, пока restore/promote. RTO прогнать |
| **3 ЦОДа** | ЦОД-3 аналогично ЦОД-2 | То же; не три писателя каталога |

GeoServer Cloud / Helm camptocamp — **другой** дистрибутив, не этот инстанс 2.24.4.

### Предпосылки прода

- Письменное решение ИБ: остаёмся на EOL 2.24.4.
- Замер RTT: под → PostGIS, под → диск каталога — **внутри ЦОД-1**.
- Бэкап: data dir + security xml + PostGIS. Тайлы GWC можно пересчитать; каталог — нет.
- PKI на Ingress. Cookie сессии: флаг **Secure**.
- Расширения только stable той же 2.24.4. Community JAR с `2.24.x` nightly на release WAR часто **не встанет**.

### Установка (ЦОД-1)

1. Собрать/пролить **2.24.4** + `control-flow` (+ `gwc-s3`, если тайлы в объектку).
2. Вынести data dir. Сменить `admin` и master password (Docker README: проверить на теге 2.24.4, не копировать ENV линейки 2.28 вслепую).
3. Снять demo `topp:states` / sample.
4. Ingress TLS, `PROXY_BASE_URL`, `GEOSERVER_CSRF_WHITELIST` (не `GEOSERVER_CSRF_DISABLED=true`).
5. XXE: `-DENTITY_RESOLUTION_ALLOWLIST=...`; unrestricted entity resolution **выкл**.
6. CORS — явный origin UI, не `*` с примеров Tomcat.
7. Status: не светить env (`GEOSERVER_MODULE_SYSTEM_ENVIRONMENT_STATUS_ENABLED` false) — в 2.24.4 чинили утечку env (CVE-2024-34696).
8. Лог профиль **PRODUCTION**.
9. Readiness: не только «Tomcat принял 8080», а GetMap известного слоя **200**.

Второй под: понять, **когда** он видит смену стиля (рестарт / reload / никогда). Пока не поняли — это не кластер, это две копии.

### Конфигурирование прода

| Слой | Правило |
|---|---|
| Сеть | 8080 не в интернет; admin/REST из внутренней сети |
| JDBC | пароли не в Git и не в Capabilities |
| HSTS | на Ingress (в GeoServer HSTS дефолт выкл) |
| JVM | `-Xms`=`-Xmx` явно; limit K8s **выше** heap (JAI, native); G1 при большой куче |
| `serviceStrategy` | в примерах доки PARTIAL-BUFFER2 |
| Шрифты кириллицы | проверить `/opt/additional_fonts` на **конкретном** 2.24.4 |
| PDB | не снимать все поды зоны сразу |

Не ставить два механизма синка каталога сразу (JMS + JDBCConfig). JMS multicast в Kubernetes обычно не живёт.

### Масштабирование (когда появятся цифры)

1. GetMap → ноды **и** control-flow (ориентир модуля: ~2× ядер параллельных GetMap **на инстанс** — старт, не смета).
2. Подложка → GWC seed, не 64G heap.
3. WFS → лимиты GetFeature и индексы PostGIS; вектору лишняя RAM почти не помогает (дока).
4. Растр → heap + пирамиды, не 200 мелких GeoTIFF.

### Проверка прода (пока это не пройдено — это не прод)

1. About: **2.24.4**, не nightly `2.24.x`. Пароль не `geoserver`. Demo-слоёв нет.
2. GetCapabilities: URL внешние, не ClusterIP.
3. Убить под: GetMap жив. WFS-T отказ. JDBC write в эталон не проходит.
4. Restore data dir + PostGIS на стенде.
5. Учение «ЦОД-1 выключен»: карта нет, пока DR. Учение «PostGIS мёртв»: пустая карта при живых подах — ожидаемо.
6. ИБ зафиксировала EOL-риск.

### Сильные / слабые стороны прод-схемы (актив в одном ЦОДе + DR)

| Сильное | Слабое |
|---|---|
| Каталог не шарится через город | Падение ЦОД-1 = нет WMS у всех |
| Меньше community-модулей | RTO = restore data dir + store, минуты надо замерить |
| Чтение масштабируется подами внутри ЦОДа | EOL: дыры после 2.24.4 апстрим не закроет |
| gwc-s3 в stable | control-flow считает ноду; кривой HPA = OOM |

**Не готов к проду**, если: тег `2.24.x` nightly; пароль `geoserver`; demo-слои; один под как «прод»; три реплики на RWO; общий data dir через город; WFS-T «на всякий случай»; `CORS=*`; CSRF выкл; unrestricted XXE; JMS multicast в K8s; community nightly JAR; карта назначена SoT; линейка 2.24 в КИИ **без** фиксации EOL; путаница с GeoData.

---

## Источники

- Релиз 2.24.4, CVE-2024-36401: https://geoserver.org/release/2.24.4/
- EOL / Java 11 / Tomcat 8.5–9: https://github.com/geoserver/geoserver/wiki/Release-Schedule
- Docker 2.24 (nightly `2.24.x`): https://docs-archive.geoserver.org/2.24.x/en/user/installation/docker.html
- Production config, WFS-T, XXE, console: https://docs-archive.geoserver.org/2.24.x/en/user/production/config.html
- Кластер = ноды за proxy: https://docs-archive.geoserver.org/2.24.x/en/user/production/misc.html
- Control-flow: https://docs-archive.geoserver.org/2.24.x/en/user/extensions/controlflow/index.html
- Дефолт `admin`/`geoserver`: https://docs.geoserver.org/2.24.x/en/user/gettingstarted/web-admin-quickstart/index.html
- Правила: `GeoServer.md`
