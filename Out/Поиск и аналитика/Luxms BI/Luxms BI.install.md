# Luxms BI 12.x — установка (учебный контур)

Luxms BI — платные дэшборды: без договора, лицензии и логина в репозиторий поставщика пакеты не скачать. Публичного образа на Docker Hub нет. Штатный путь — пакеты Linux на VM (`dnf` / `apt` / `zypper`). Самодельный docker-compose «как у Grafana» поставщик как прод не поддерживает.

**Версия.** Новая установка без требования государственного сертификата — линейка **12**. Точный номер пакета (на сайте заметок — **12.0.2**, портал документации — **12.1.0**) берётся из **договора и вашего** репозитория, не «самое свежее». Команды ниже — из руководства **12.1.0**. Если в договоре другая сборка 12 — имена пакетов сверять с **её** sysadm-guide.

**Сертификат.** Сборка **11.0.x** (ФСТЭК № 5055) — другой комплект. Её **нельзя** заменить обычной 12: сертификат 11 на 12 не действует.

**Допущение:** закрытая сеть, **одна** VM, лицензия DEVL / тест, если она есть в договоре. Этот стенд в бой не копировать.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна виртуальная машина. ОС только из матрицы поставщика: **Astra Linux SE 1.7 / 1.8**, **РЕД ОС 7.3 / 8.0**, **Альт СП 10**, **Rocky Linux 9**, **MosOS Arbat 15.5**. Windows-сервер не заявлен. «Любой Ubuntu» — нет. Kubernetes / Helm 12.x в открытой документации нет: на учёбе не ставим.

Каталог данных PostgreSQL — **локальный диск**, не NFS. NTP на этой VM обязателен.

```mermaid
flowchart LR
  U["Браузер с jump-хоста"] -->|"HTTP :80"| VM["Одна VM: Postgres 15/17\n+ bi-web + appserver\n+ KeyDB + NATS"]
  VM -->|"JDBC / FDW"| SRC["Ваша витрина\nPostgres / ClickHouse"]
```

**Сколько.** Цифры — таблица вендора для объёма фактов **~1 млн строк**; это не смета вашей нагрузки. «Минимум, чтобы процесс поднялся» и «минимум до 50 людей» — разные строки, путать нельзя.

| Зачем | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Тест / демонстрация | **4 ядра** ≥ 2 ГГц | **6 ГиБ** | **200 ГиБ**, чтение от 150 МБ/с | [tech-info 12.1.0](https://luxmsbi.ru/docs/12.1.0/overviews/general-overview/tech-info) |
| Минимум до **50** одновременных | **8 ядер** ≥ 2 ГГц | **24 ГиБ** | **500 ГиБ**, тот же порог диска | та же страница |
| Рекомендация на **50** одновременных | **16 ядер** ≥ 2 ГГц | **32 ГиБ** | **1 ТиБ** | та же страница |
| Один хост в виртуализации (гайд администратора) | от **8 vCPU** | от **32 ГиБ** | см. точки монтирования | [planning](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning) |

Для учебного контура достаточно строки **тест / демонстрация** (4 / 6 / 200). Её **нельзя** нести в бой как «официальный минимум до 50 людей».

Точки монтирования (старт, не смета): `/opt/bi` **2 ГиБ**, `/opt/nats` **10 ГиБ**, данные Postgres **10 ГиБ**, `/var/log` **8 ГиБ**. Терабайты фактов — в ClickHouse / озеро, не в единственный PostgreSQL ядра.

**Сильная сторона:** одна VM совпадает с официальной «быстрой» / dev-схемой. **Слабая:** снесли диск — снесли атласы и права. Patroni, Consul, кластер NATS на этом стенде нет.

**Критично:** порты **5432** (PostgreSQL) и веб-вход **не в интернет**. Не `latest`. Не подменять сертифицированную **11.0.x** линейкой 12.

---

## Установка для новичка

Команды — **на Linux-машине стенда**, не в PowerShell. Пример ниже — **Rocky Linux 9** + PostgreSQL **17**, как в [Rocky. Установка](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-rocky). Другая ОС — страница той ОС: [Astra](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-alse), [РЕД ОС](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-redos), [Альт СП](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-altsp), [MosOS](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-mosos). Имена пакетов 12.x: `bi-web`, `bi-appserver-mono`, `bi-pg-sql17` — не путать с именами сертифицированной 11 (`luxmsbi-*`).

Без учётки репозитория поставщика (`https://download.luxms.ru/repository/customers/12/…`) **установку завершить нельзя**: пакетов BI в публичных зеркалах нет.

### Что должно быть до установки

**Есть:**

- договор, лицензия (на тест — DEVL, если так в договоре), логин/пароль в закрытый репозиторий
- ОС из списка выше; NTP; вход с jump-хоста / VPN
- свободны **80** (веб `bi-web`), **5432** (Postgres), **6379** (KeyDB), **4222** (NATS)
- локальный диск под `$PGDATA`

**Нет** (и не должно появиться на этой VM):

- публикация 80 / 5432 / 6379 / 4222 в интернет
- самодельный Compose / Helm
- PostgreSQL **13** (пакеты вендор не выпускает с 01.12.2025)
- сертифицированная 11 вместо заказанной 12 (и наоборот)
- модуль «ИИ-аналитик»

### Этап 1. Репозиторий поставщика

**Что делаем:** подключаем репозитории BI и third-party, кладём GPG-ключи. Без `username` / `password` от поставщика `dnf` пакеты BI не отдаст.

Ключи (закрытый сегмент — скачать заранее и занести внутрь):

```bash
sudo curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-Luxms \
  https://download.luxms.ru/repository/thirdparty-signed/RPM-GPG-KEY-Luxms
sudo curl -o /etc/pki/rpm-gpg/GPG-KEY-LUXMS-10AF981E \
  https://download.luxms.ru/repository/thirdparty-signed/GPG-KEY-LUXMS-10AF981E
```

Файл `/etc/yum.repos.d/bi.repo` — как в гайде Rocky. Раскомментировать и заполнить `username=` / `password=` учётки репозитория. Не коммитить в git.

Успех: `dnf repolist` видит `bi12` и `bi-thirdparty`. Ошибка 401 / пустой список — нет доступа; дальше шаги бессмысленны, пока поставщик не откроет репозиторий.

Astra / Альт / MosOS — синтаксис репозитория на **их** страницах deploy, не копировать yum-файл на apt.

### Этап 2. Быстрый путь (официальный одноузловой)

**Что делаем:** ставим пакет `bi-setup` и запускаем Ansible-сценарий поставщика (Ansible — программа, которая по сценарию ставит пакеты и правит конфиги). Это [быстрая установка](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup) для одного узла без отказоустойчивости.

Сценарий кладёт файлы БД в `/data/pgdata`. Нужен отдельный диск — заранее смонтировать его в `/data`.

| ОС | Команда |
|---|---|
| Rocky Linux | `dnf -y install epel-release` затем `dnf -y install bi-setup` |
| РЕД ОС | `dnf -y install bi-setup` |
| Astra Linux | `apt -y install bi-setup` |
| Альт СП 10 | `apt-get -y install bi-setup` |
| MosOS 15 | `zypper -y install bi-setup` |

```bash
cd /opt/bi/install
ansible-playbook -i local-inv.yml book-deploy-bi.yml
```

Успех: playbook без ошибки; сервисы видны (этап 7). Если Ansible в контуре запрещён — этапы 3–6 вручную.

### Этап 3. PostgreSQL вручную (если не было этапа 2)

**Что делаем:** ставим СУБД **15 или 17** (ядро Luxms живёт **внутри** Postgres). Расширения (`plv8`, `pgsql-http`, pub/sub к KeyDB, FDW к KeyDB) — **из пакетов Luxms**, не с GitHub.

На Rocky 9 гайд показывает PostgreSQL 17 из PGDG + `epel-release` (репозиторий Cisco openh264 лучше выключить — с российских адресов его часто не достать).

```bash
sudo dnf -y install postgresql17 postgresql17-server postgresql17-contrib
echo -e 'PATH=/usr/pgsql-17/bin:$PATH\nexport PATH\n' \
  | sudo /usr/bin/tee -a /var/lib/pgsql/.bash_profile
sudo -iu postgres initdb
sudo systemctl enable --now postgresql-17
```

Нестандартный каталог данных: `-D /data/pgdata` и переменная `PGDATA` у пользователя `postgres`. Владелец каталога — `postgres`.

Проверка ([deploy-check](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-check)): `ss -nlpt` — процесс слушает **5432**; `sudo -iu postgres psql -V` печатает 15.x или 17.x.

**Пробел матрицы:** таблица совместимости ОС×СУБД на [postgresql](https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/postgresql) для Rocky 9 отмечает PostgreSQL **15**, ячейка **17** пустая, при этом страница установки Rocky даёт пакеты **17**. Что ставить — пакет из **вашего** репозитория и страница **вашей** ОС, не догадка.

### Этап 4. Ядро BI в Postgres

**Что делаем:** пакет создаёт базы `mi` (конфиг BI) и `bidata` (данные после импорта файлов) и роли, в том числе **`bi`**.

```bash
sudo dnf -y install bi-pg-sql17
```

Для Postgres Pro 17 на РЕД ОС в том же гайде — `bi-pg-pro17`. Уже существующие базы пакет не переписывает.

Сразу сменить пароль роли `bi` (компоненты им ходят в БД). Пароли по умолчанию вендор **запрещает** в продуктовых системах; на тесте без реальных данных — до первой настоящей витрины:

```bash
sudo -iu postgres psql -c '\password bi'
```

То же для `bireader`, `bidata`, `infosec_reader`. Новые пароли потом прописать в источниках UI.

### Этап 5. KeyDB и NATS (один процесс каждый)

**Что делаем:** KeyDB (порт **6379**, сессии и антибрут) и NATS (клиенты **4222**, файлы на `/opt/nats`) — по одному узлу. Блок `cluster` в конфиге NATS **не** включать.

```bash
sudo dnf -y install keydb
sudo systemctl enable keydb --now
sudo dnf -y install nats-server
```

В `/opt/nats/nats-server.conf`: `server_name` = FQDN этой VM. Пример `user: "x", pass: "y"`, `no_tls: true`, `no_auth_user: "be"` — учебный каркас; на стенде с реальными данными сменить. Затем:

```bash
sudo systemctl enable nats-server --now
```

Проверка: `nats-server -v`; `systemctl is-active keydb nats-server`.

На одном хосте **не** копировать из гайда `protected-mode no` и снятие `bind 127.0.0.1` — это для разноса KeyDB по машинам.

### Этап 6. Веб и Java

**Что делаем:** `bi-web` — отдельный Nginx + Lua (не системный nginx ОС), сервис `bi-web.service`. `bi-appserver-mono` — Java «всё в одном» (управление + источники). OpenJDK 17 ставится зависимостью.

Rocky 12: сначала модуль Nginx 1.26 (`dnf module enable nginx:1.26`) — 1.26 и 1.24 вместе не живут.

```bash
sudo dnf -y install bi-web
```

Файл `/opt/bi/conf/nginx/lua/bicfg.lua`: `dbhost`, `dbport=5432`, `dbname=mi`, `dbuser=bi`, **`dbpass` = пароль роли `bi` с этапа 4**, не пример `"bi"`.

```bash
sudo systemctl enable --now bi-web
sudo dnf -y install bi-appserver-mono
```

В `/opt/bi/conf/appserver/application.properties`:

```properties
bi.datasource.url=jdbc:postgresql://127.0.0.1:5432/mi
bi.datasource.username=bi
bi.datasource.password=<пароль роли bi>
bi.nats.servers=nats://localhost:4222
```

`JAVA_HOME=/usr/lib/jvm/jre-17-openjdk` в окружении unit, как в гайде.

```bash
sudo systemctl enable bi-appserver --now
sudo systemctl enable bi-headless-chrome --now
```

`bi-headless-chrome` (отрисовка PNG) ставится зависимостью mono; в интернет не открывать. Data Boring (`bi-etl`, порт **1880**) на первом стенде не обязателен: перед стартом нужен JWT из UI после входа Admin.

Учебный атлас, если репозиторий отдаёт пакет (Luxms BI **12.0.0+**):

```bash
sudo dnf install bi-pg-demo174
```

Astra: `sudo apt install bi-pg-demo174`. Имя в тексте новости иногда `bi-pg-ds-demo174` — ставить то, что реально есть в **вашем** `dnf search` / `apt-cache`.

### Этап 7. Стенд живой

```bash
systemctl list-units | grep 'nats\|bi-\|postgres'
ss -nlpt | grep -E '80|5432|6379|4222|8080'
```

Успех: `postgresql-*`, `bi-web`, `bi-appserver`, `keydb`, `nats-server` — `active`; браузер с jump-хоста открывает UI (следующий раздел); на дэше есть цифры (учебный атлас или ваша витрина), не пустая рамка.

**Чего этот стенд не доказывает:** отказ зала; выборы лидера Postgres (Patroni/Consul не ставили); кластер NATS; липкость сессий на нескольких веб-узлах; нагрузка «50 одновременных»; живой JDBC в боевое озеро; единый вход Keycloak; что сертифицированная 11 «уже стоит, потому что это 12».

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL.** После быстрой установки вендор пишет: открыть `http://[IP-адрес|FQDN]` — **без номера порта**, то есть HTTP на **80** (`bi-web` / Nginx). Отдельной «консоли на 9443» у продукта нет. Открывать **с VPN / jump-хоста**, не из интернета.

HTTPS на самом веб-узле вендор для высокой нагрузки **не рекомендует** (TLS — на балансировщике). На учебном одном хосте HTTP внутри закрытой сети — ожидаемо. Самоподписанный HTTPS — только если сами включите [приложение SSL](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/appendix/ssl).

**Учётка.** При установке из репозитория регистрируется администратор **`adm`**, пароль по умолчанию **`luxmsbi`**. Источник: [planning](https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning). Это **только закрытый стенд**.

**Смена пароля.** Сразу после входа: раздел «Администрирование» → пользователь `adm` → новый пароль. Вендор настоятельно рекомендует сменить. Учебный `luxmsbi` в бой не копировать. Новый — сейф / Vault, не git.

Парольная политика из коробки к паролям **не** применяется, пока её не назначат пользователю ([password-policy](https://luxmsbi.ru/docs/12.1.0/guides/administrator-guide/password-policy)). На стенде: сменить `adm`, не оставлять `bi` / `bi` в `bicfg.lua` и `application.properties`, если на витрине уже не учебный CSV.

Роли лицензии: **Viewer** смотрит, **Creator** подключает источники и ETL, **Admin** — всё, включая пользователей.

---

## Подключение к своей системе

Аналитики открывают **веб Luxms** (браузер → `bi-web`). Цифры — из озера / **PostgreSQL-витрины** / **ClickHouse**, которые наполнил интеграционный контур. Luxms **не ходит** в ведомства. Клиент к источнику — процесс **datagate** / appserver-mono по **JDBC** (или FDW: таблица в ядре Postgres, которая читает чужую БД по сети).

Ядро `mi` / `bidata` — не эталон карточек. Подключать **свою** витрину, не «данные живут только внутри Luxms».

В UI: раздел данных → вкладка **PostgreSQL** или **ClickHouse** → «Сервер» (хост:порт, БД, логин, пароль) или «URL» (JDBC). Кнопка «Проверить соединение». Источник сделать **readOnly**, если writeback не нужен.

Пример JDBC ClickHouse из гайда (хост, порт, TLS — **ваши**, не копировать 443 «на глаз»):

```text
jdbc:clickhouse://some.clickhouse.server:443/default?max_insert_block_size=100000&max_threads=8&ssl=true&socket_timeout=1800000
```

Порт вашего ClickHouse — как в `ClickHouse.install.md` (часто HTTP **8123**), не порт ядра Luxms **5432**. Список СУБД-источников: [dbms](https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/dbms). Тест 12.x для ClickHouse: **25.3.2.39**.

**Первый учебный источник:** не superuser озера; отдельная учётка SELECT по витрине. Успех: дэшборд не пустой. Упало озеро — UI жив, цифры пустые: так и должно быть.

### Секреты (не в git)

| Секрет | Где живёт | Куда не класть |
|---|---|---|
| Логин/пароль репозитория `bi.repo` | `/etc/yum.repos.d/bi.repo` (или apt/zypper) | git, чат |
| Пароль роли `bi` (и `bireader` / `bidata`) | Postgres + `bicfg.lua` + `application.properties` | git, образ |
| Пароль UI `adm` после смены | сейф / Vault | git |
| JDBC витрины / ClickHouse | UI источников + Vault | git |
| NATS SYS `x`/`y`, FE `fe` | `/opt/nats/nats-server.conf` | прод-репозиторий с учебными `"x"`/`"y"` |
| `ETL_MASTER_JWT` (если поставите Data Boring) | `/etc/sysconfig/bi-etl` | git |

В git — процедура и имена файлов без значений.

### Чем продукт не является

| Сосед | Чем отличается |
|---|---|
| Apache Superset | OSS, другой стек (Python, свой Postgres метаданных, Redis). Luxms — коммерческие пакеты, логика в PostgreSQL, внутренний NATS. Соседний UI, не подмена без решения «какой BI главный» |
| Grafana | Операционный мониторинг Nginx/Postgres/NATS, не бизнес-дэшборды |
| ClickHouse | Склад фактов. Luxms — UI и ядро на Postgres; ClickHouse подключается как источник |
| Kafka | Источник для витрин, не транспорт ядра Luxms |
| Camunda | Не BPMN |
| Интеграционное API к ведомствам | Datagate исходящие только на витрины, не в подсеть СМЭВ |

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Портал документации (редирект на latest через `versions.json`; сейчас **12.1.0**) | https://luxmsbi.ru/docs/ |
| Железо: тест 4/6/200; до 50 людей 8/24/500; рекомендация 16/32/1 ТиБ; лицензии DEVL | https://luxmsbi.ru/docs/12.1.0/overviews/general-overview/tech-info |
| PostgreSQL 15/17, конец пакетов 13 с 01.12.2025, расширения, матрица ОС | https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/postgresql |
| СУБД-источники, ClickHouse 25.3.2.39 в тестах 12 | https://luxmsbi.ru/docs/12.1.0/overviews/compatibility/dbms |
| ОС, NTP, `adm` / `luxmsbi`, 8 vCPU / 32 ГиБ на один хост, точки монтирования | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/planning |
| Быстрая установка: `bi-setup`, Ansible, вход `http://[IP\|FQDN]` | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/quick-setup |
| Варианты: образ Rocky 9 у облаков; Ansible боя — после согласования схемы | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-types |
| Репозиторий, Postgres, `bi-pg-sql17`, KeyDB, `bi-web`, NATS, `bi-appserver-mono`, порты 8080/8888 | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-rocky |
| Проверка 5432 и `systemctl` | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/deploy-check |
| NATS: пакет, порты 4222/6222/8222, `/opt/nats` 10 ГиБ, учебный конфиг без TLS | https://luxmsbi.ru/docs/12.1.0/guides/sysadm-guide/appendix/nats |
| JDBC ClickHouse/PostgreSQL в UI | https://luxmsbi.ru/docs/12.1.0/guides/advanced-user-guide/data |
| Смена пароля, политики | https://luxmsbi.ru/docs/12.1.0/guides/administrator-guide/password-policy |
| Релизы 12 (публичные notes **12.0.2**) | https://luxmsbi.ru/products/relizy/ |
| Демо-атлас 12: `bi-pg-demo174` | https://luxmsbi.ru/news/analitika/obnovlenie-demonstratsionnogo-atlasa/ |
| Сертифицированная сборка **11.0.x**, ФСТЭК № 5055 | https://luxmsbi.ru/products/fstek/ |
| Неверсионированные ярлыки (как в карточке консультанта) | https://luxmsbi.ru/docs/overviews/general-overview/tech-info · https://luxmsbi.ru/docs/overviews/compatibility/postgresql · https://luxmsbi.ru/docs/guides/sysadm-guide/appendix/nats/ |
| Зачем продукт, порты, железо | `Luxms BI.md` |
| Словарь | `Luxms BI.info.md` |
| Схема стыковки с платформой | `Luxms BI.shema.md` |
| Роль консультанта | `Luxms BI.consultant.md` |

**В доке вендора нет (и здесь не выдумано):** порог RTT между залами; какой именно патч 12.0.2 vs 12.1.0 лежит в **вашем** репозитории; публичный Docker Hub / Helm 12.x; однозначная ячейка «Rocky 9 + PostgreSQL 17» в матрице совместимости (гайд установки 17 даёт, матрица для Rocky 9 — 15); явная фраза «слушайте порт 80» (есть только URL без порта); готовый пароль UI кроме `adm` / `luxmsbi`.
