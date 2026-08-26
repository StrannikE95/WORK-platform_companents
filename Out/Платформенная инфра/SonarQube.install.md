# SonarQube Server 2026.1.5 LTA — установка и конфигурирование

Связанный документ (глоссарий, редакции, DCE, почему так): `SonarQube.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch application/search-нод на несколько ЦОДов **не делаем**: порты 9001–9003 и JDBC требуют низкой задержки; вендор для search пишет *same region*, миллисекунд нет.

Версии: **SonarQube Server 2026.1.5** (LTA). Latest на ту же дату — **2026.4.1**; официальный Helm `sonarsource/sonarqube` / `sonarqube-dce` **2026.4.1 ставит Latest**, не LTA — образ/values **пинить на 2026.1.5**. Community Build — **другая линейка** (на загрузках 26.8.x), не «бесплатный DCE».  
Документация LTA: https://docs.sonarsource.com/sonarqube-server/2026.1/

HA самого приложения даёт только **Data Center Edition**. Без ключа DCE — один процесс + HA Postgres + холодный DR, это не кластер SQ.

---

## Допущения этой инструкции

1. **Stretch запрещён.** Все application nodes, все search nodes и writer PostgreSQL **одного** инстанса — **внутри одного ЦОДа**. Между ЦОДами — **Active-Cold Standby** (официальный DR): второй кластер холодный, БД реплицируется, трафик переключают вручную, затем **forced ES reindex**. Read-only replica БД для живого SQ вендор **не поддерживает**.
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`). Инсталлятор DCE — Helm `sonarqube-dce` с тегом образа **2026.1.5**.
3. Dev — изолированная сеть; Community Build или Developer на одном поде допустимы.
4. Нагрузки и LOC нет — нет цифры «N app nodes». Есть очередь CE и рычаги (workers EE+, ноды DCE).
5. Лицензия **на инстанс / LOC**. Второй живой инстанс в ЦОД-2 = вторая лицензия, если его включили. Cold DR — сверять с договором.
6. Для 2 ЦОДов: живой SQ в ЦОД-1, cold в ЦОД-2. Для 3 ЦОДов: то же + ЦОД-3 только бэкапы или второй cold. Третий ЦОД **не** добавляет третий writer и не кладёт search в чужой ЦОД.
7. PostgreSQL 14–18 UTF-8, отдельный кластер (с 2026.1 Helm **не** кладёт Postgres в чарт). H2 в проде запрещена вендором.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** проект, отчёт сканера, Quality Gate. **Не** цель: отказ ЦОДа и пачка параллельных PR.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 9000 свободен на localhost.
- Linux: `vm.max_map_count ≥ 524288` (дока 2026.1; не путать со старым 262144).
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

Community Build — чтобы увидеть UI без ключа DCE. На странице загрузок соседнего файла: Community **26.8.0.126808** (это **не** Server 2026.1.5 и не DCE). Тег образа берите со страницы загрузок / Docker Hub линейки `sonarqube` **той** сборки, не `latest` и **не** DCE-образ без лицензии.

```bash
docker run -d --name sq-dev \
  -p 127.0.0.1:9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

`sonarqube:community` без пина патча — только если сразу посмотрите digest и зафиксируете его; лучше тег с номером 26.8.x со страницы загрузок. Для препрода, который должен совпасть с боем, — образ **Server 2026.1.5** (Developer, если ключ уже есть), не Community «потому что на тесте хватало».

Привязка к `127.0.0.1` обязательна. UI: `http://127.0.0.1:9000`, вход `admin`/`admin`, сразу сменить пароль.

Для стенда дольше недели — **сразу внешний PostgreSQL**, не H2 (H2 не умеет нормальный бэкап/HA — формулировка вендора «not recommended in production»). JDBC: `jdbcOverwrite` / `SONAR_JDBC_*` на стендовый Postgres.

Проверка: Administration → System / about: линейка **не** Latest 2026.4, если цель — как в проде LTA. Сканер в CI (`sonar.token`), не пароль в Git. Background Tasks = SUCCESS.

### Установка (Kubernetes Dev)

Чарт **`sonarqube`** (не `sonarqube-dce`): `community.enabled=true` **или** Developer + ключ; `jdbcOverwrite.*` на свой Postgres; `ingress-nginx` чарта не тащить в привычку прода. `initSysctl: true` на тесте часто оставляют; в прод sysctl делает администратор нод.

Режим «3 search DCE на ноутбуке» **не нужен**. DCE без ключа как кластер не взлетит.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Кластер DCE | нет | Нет требования пережить узел |
| БД | Postgres, если стенд не однодневный | H2 врёт про прод |
| Force authentication | вкл | Дефолт; выключение открывает куски API |
| IdP | можно локальных | Сначала контур сканера |
| HPA / CE workers | дефолт | Нет нагрузки |
| Пароль Helm `AdminAdmin_12$` | **не** использовать | Публичный values |

Чего **не** упрощать: сканер живёт в **CI**; Force authentication, если стенд маршрутизируется; не путать Community с Server LTA.

### Проверка Dev

1. Логин, смена `admin`/`admin`.
2. Один репозиторий: задача SUCCESS, issues видны.
3. Понимание: зелёный gate на игрушке **не** доказывает очередь CE и DCE.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, Try-out / Helm non-DCE | Нет HA приложения, нет LB на два app |
| Дешёво показывает шум правил на вашем коде | Community без PR-анализа легко уехать в прод «потому что хватало» |
| | Дефолтные пароли приучают открыть :9000 |

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **одного app или одного search внутри ЦОДа** (DCE 2+3: вендор — *one application node and one search node can be lost*). Отказ **целого ЦОДа** = SQ лежит, пока cold promote + **reindex ES**. Падение SQ **не должно** ронять Kafka; политика CI (ждать / падать / обход gate) — ваша, её нет в продукте.

### Почему не stretch

Search — Elasticsearch: кворум из 3 нод переживает **одну** зону, не две. Hazelcast 9003 и ES 9002 чувствительны к RTT. JDBC — low latency до writer. Stretch «по одному search на ЦОД» без замера — лотерея cluster state. Официальный DR — **два кластера**, не один ES на три площадки.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один DCE:

- `searchNodes.replicaCount: 3`, `applicationNodes.replicaCount: ≥ 2`, все в **этом** ЦОДе, anti-affinity по ноде (не по чужому ЦОДу);
- SSD PVC на search, **не NFS**; не дефолт **5G** как план ёмкости;
- `vm.max_map_count=524288` на нодах **без** privileged `initSysctl` в проде;
- ES authentication + TLS (`searchAuthentication`, `nodeEncryption`);
- свой Ingress/Gateway + health check, **без sticky** (JWT); не `ingress-nginx` из чарта (testing only);
- PostgreSQL writer HA **в ЦОД-1**; приложение **не** ходит в read-only replica;
- образ **2026.1.5**, не 2026.4.1 из дефолта Helm.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: живой DCE. ЦОД-2: **cold** DCE + реплика Postgres (streaming/PITR). CI в штате ходит на VIP ЦОД-1 | Запись/UI стоят, пока оператор не переключил трафик и не сделал **forced reindex**. RPO > 0 при догоне архивом |
| **3 ЦОДа** | ЦОД-3: второй cold **или** только бэкапы БД | То же; третий живой SQ = ещё одна лицензия и вторая «правда» Quality Gate |

Не ставить по DCE «в каждый кластер как Falco». Три полноценных инстанса — тройной LOC и тройные профили, обычно хуже.

Без лицензии DCE прод-HA приложения **нет**: остаётся EE/DE + тот же cold DR (один процесс в ЦОД-1).

### Предпосылки прода

- Ключ DCE (или честный отказ от HA приложения).
- HA Postgres UTF-8, failover writer прогнан **без** SQ.
- Sysctl на нодах; StorageClass SSD зональный **этого** ЦОДа.
- CI-агенты доверяют CA прокси. Токены `sonar.token`, не пароль.
- Секреты: JDBC, JWT (`applicationNodes.jwtSecret`), ES password, PKCS#12 — Vault. Нет `admin`/`admin`, нет `AdminAdmin_12$`.

### Установка (DCE, ЦОД-1)

```bash
helm upgrade --install sonarqube sonarqube/sonarqube-dce \
  --version <чарт,-совместимый-с-LTA> \
  --set elasticsearch.javaOpts=-Xmx2g \
  --set applicationNodes.jwtSecret="<свой>" \
  --set jdbcOverwrite.jdbcUrl=jdbc:postgresql://pg-sonarqube-rw:5432/sonar \
  --set postgresql.enabled=false
```

Точный флаг пина образа **2026.1.5** — в values чарта вашей поставки (часто `image.tag` / `applicationNodes.image.tag`); дефолт чарта 2026.4.1 **не** оставлять. JDBC-пароль только из Secret, не plaintext deprecated-поля.

Плагины ставить на **все** app nodes; выкат = простой **всего** кластера (цитата cluster-доки). ArgoCD вендор помечает как *not currently fully supported*.

### Конфигурирование

1. Сменить админа, **переименовать** `sonar-administrators` до синхронизации групп с IdP.
2. Force authentication оставить. IdP (SAML/LDAP). Токены CI с правом Execute Analysis, ротация.
3. Quality Gate в PR (Developer+). Политика: красный gate = нельзя в `main`.
4. NetworkPolicy: 9000 только через HTTPS; 9001–9003 не с мира; JDBC только с app.
5. `ingress-nginx.enabled=false`, `initSysctl.enabled=false`, memory request = limit (вендор: memory non-compressible).
6. PDB: не снимать сразу 2 из 3 search.
7. Мониторинг очереди CE (`pending_count` / `pending_time`), heap, диск search (≥10% свободно; watermark ES ~90–95%).

Cold в ЦОД-2: отдельный Helm, bootstrap от реплики БД, **не** включён в LB, пока не DR. После включения — reindex (UI может встать, Postgres жив).

### Масштабирование (когда появятся цифры)

1. Очередь CE → workers (EE/DCE, глобальные: число × app nodes) **после** запаса heap; HPA app (min 2) только если нет bottleneck на БД; на апгрейд HPA выкл.
2. Индекс issues → RAM/диск search, не «ещё Kafka broker».
3. Тяжёлый скан → агент CI. Search HPA **не** масштабирует.

### Проверка прода (пока это не пройдено — это не прод)

1. О about: **2026.1.5**, не 2026.4.1.
2. Убить 1 app: LB жив. Убить 1 search: кластер зелёный/стабильный по доке 2+3.
3. Сканер токеном, Background Task SUCCESS. Без токена — отказ.
4. Failover writer Postgres **с** живым SQ (учение).
5. На препроде: включить cold ЦОД-2, переключить VIP, **reindex**, замерить RTO. Ошибка promote = две правды, не «потом смержим».

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + cold DR)

| Сильное | Слабое |
|---|---|
| ES и JDBC не ходят между ЦОДами | Падение ЦОД-1 = нет SQ, пока DR |
| Совпадает с документом Active-Cold | RTO включает reindex; минут в цитате вендора нет |
| Одна лицензия, один Quality Gate | Без DCE нет HA приложения |
| | Лицензия на инстанс/LOC — второй живой SQ считать отдельно |

**Не готов к проду**, если: Community/DE/EE «в трёх ЦОДах для HA»; H2; Helm Latest 2026.4.1 вместо LTA; `admin`/`admin` или `AdminAdmin_12$`; search на NFS; read-replica как второй живой SQ; stretch 9002 на 2–3 ЦОДа; CI обходит gate; ждут, что SQ заменит Falco/Wazuh.

---

## Источники

- Загрузки LTA 2026.1.5: https://www.sonarsource.com/products/sonarqube/downloads/
- DCE topology 2+3: https://docs.sonarsource.com/sonarqube-server/2026.1/server-installation/data-center-edition/dce-topology.md
- DR Active-Cold, нет read-only DB, reindex: https://docs.sonarsource.com/sonarqube-server/2026.1/server-installation/data-center-edition/on-kubernetes-or-openshift/setting-up-disaster-recovery.md
- БД, PostgreSQL 14–18, H2 не прод: https://docs.sonarsource.com/sonarqube-server/2026.1/server-installation/installing-the-database.md
- Helm sonarqube / sonarqube-dce: https://artifacthub.io/packages/helm/sonarqube/sonarqube
- Правила: `SonarQube.md`
