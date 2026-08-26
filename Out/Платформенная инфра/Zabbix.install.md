# Zabbix 7.0.30 LTS — установка и конфигурирование

Связанный документ (глоссарий, native HA, proxy group, почему так): `Zabbix.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch связки server + PostgreSQL на несколько ЦОДов **не делаем**: heartbeat HA идёт **через БД**; sync-Postgres между площадками запрещён ping'ом (`PostgreSQL.install.md`). Сервер — **active-passive**: живой съём и триггеры только у **одной** active-ноды.

Версия: **Zabbix 7.0.30** (LTS, 25 августа 2026). Полная поддержка 7.0 — до 30 июня 2027. В прод **не** брать 7.4.14 (standard) и не ждать 8.0, которого на дату соседнего файла на странице исходников нет.  
Образы: `zabbix/zabbix-server-pgsql`, `zabbix-web-nginx-pgsql`, `zabbix-proxy-sqlite3` / `zabbix-proxy-pgsql`, `zabbix-agent2`, … Тег **`alpine-7.0.30` / `ubuntu-7.0.30`**, не `latest`.  
Compose: https://github.com/zabbix/zabbix-docker ветка **7.0**.  
Документация: https://www.zabbix.com/documentation/7.0/en/manual/

**Helm:** у вендора **нет** first-class чарта «весь server на Kubernetes». Официальный Helm в `git.zabbix.com` (ZT/kubernetes-helm) ставит **мониторинг Kubernetes из уже существующего** Zabbix. OpenShift Operator в публичных мануалах — линии 5.0/6.0, для 7.0 не брать. Прод: **официальные образы** + ваши манифесты.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Server HA и writer PostgreSQL — **внутри одного ЦОДа**. Другие ЦОДы: **прокси** (и при жёсткой изоляции — отдельный Zabbix, обычно хуже для on-call).
2. Прод — Kubernetes 1.36.x в каждом ЦОДе. Community Helm с Artifact Hub — не сертификат вендора на 7.0.30.
3. Бэкенд — PostgreSQL 13–18 (якорь платформы **18.6**; 18.x с Zabbix 7.0.20). Oracle deprecated. TimescaleDB Community — если retention/NVPS выйдут за стенд.
4. Dev — изолированная сеть. Пароль `Admin`/`zabbix` сменить в день установки.
5. Нагрузки/NVPS нет — нет ядер и ТБ PGDATA.
6. AGPLv3 с 7.0 — юридический факт; этот файл не заменяет заключение.
7. Для 2 ЦОДов: мозг в ЦОД-1, proxy group в ЦОД-2. Для 3 — прокси в каждом; третий writer server **не** появляется.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** host / item / trigger / action, один агент. **Не** цель: отказ ЦОДа и proxy group.

### Предпосылки

- Linux (Windows как *server* не заявлена) **или** Docker.
- Порт UI не в интернет. PostgreSQL для server; прокси на тесте можно не ставить.

### Установка (Compose — основной путь Dev)

Репозиторий `zabbix-docker`, ветка **7.0**, `compose_pgsql.yaml` (или Makefile README). Пин образов **7.0.30**.

```bash
# образы: zabbix-server-pgsql, zabbix-web-nginx-pgsql, zabbix-agent2 — тег alpine-7.0.30 (или ubuntu-7.0.30)
docker compose -f compose_pgsql.yaml ps
# UI: http://127.0.0.1:8080 (или как в compose) — не 0.0.0.0 без нужды
```

Порты: агент **10050**, server/proxy/trapper **10051**, Java gateway **10052**, web service **10053**, frontend **80/443**. TLS **не** открывает новый порт: тот же 10051 принимает plaintext и TLS. Firewall режет адресами и `TLSAccept`.

Сразу: войти `Admin` / `zabbix`, **сменить пароль**. Не публиковать :80. После 5 неудачных логинов UI молчит 30 секунд — это не WAF.

Пакетный путь: https://www.zabbix.com/download — версия **7.0**, PostgreSQL, nginx/Apache; server + frontend + agent 2 на одной машине.

### Установка (Kubernetes Dev)

Один Deployment **server** (`HANodeName` **не** задавать), один frontend, PostgreSQL с PVC, опционально DaemonSet agent 2. **Не** `replicas: 2` у server без HA-параметров: два писателя на одну БД или сломанный старт.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| HA server | выкл (standalone) | Нет требования пережить выкат |
| Прокси | выкл | Нечего разносить |
| TimescaleDB | выкл | NVPS стенда не греет housekeeper |
| TLS | можно HTTP в изоляции | Иначе PKI раньше триггеров |
| Java gateway | выкл, пока нет JMX | Лишние части |
| `AllowKey=system.run` | выкл | Не «чтобы удобно» |
| Connector → Kafka | выкл | Не цель стенда |

Чего **не** упрощать: **7.0.30** на всех компонентах; смена `Admin`/`zabbix`; один item не localhost и одно **дошедшее** уведомление; не класть в item тело ответа интеграционного API.

### Проверка Dev

1. Latest data растёт. Триггер умеет PROBLEM и OK. Action дошёл.
2. Версии server/frontend/agent совпадают по линии 7.0.30.
3. Успешный график CPU **не** доказывает standby и буфер прокси.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный compose | Нет proxy buffer, нет HA |
| Standalone — штатный режим | Compose-all-in-one уедет в прод |
| | HTTP + дефолтный Super Admin «чуть торчит» — инцидент |

Препрод: 2 HA-ноды **в одном ЦОДе**, отдельный PG с бэкапом, proxy group из 2, TLS, SSO — даже без боевого NVPS.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **active-процесса внутри ЦОДа** (standby рядом); пережить отказ **чужого ЦОДа** буфером прокси; пережить отказ **ЦОДа мозга** ценой тишины триггеров, пока restore PG+server. Переживание двух ЦОДов native HA **не обещает**.

### Почему не stretch

Vendor: HA *will work across sites*, *does not have specific requirements for the databases*. Перевод: **сервер** умеет разъехаться; **БД** — ваша. Sync PG + heartbeat через ту же БД при плохом RTT — лотерея флаппинга и убитого NVPS. Порога мс в доке Zabbix **нет**. Поэтому server+DB **не** размазываем.

Нельзя повесить TCP-LB «на все HA-поды»: standby **не слушает** 10051.

### Топология

**Внутри ЦОД-1 (мозг):**

- PostgreSQL HA **этого** ЦОДа (CNPG `instances: 3`, не stretch);
- Zabbix server native HA: минимум **2** ноды (active + standby) **в этом же ЦОДе**, разные `HANodeName` / `NodeAddress` (стабильный DNS, не случайные ReplicaSet-имена). Третья standby не ускоряет съём;
- Frontend ≥ 2, HTTPS, SSO; при HA **не** прописывать address:port сервера в `zabbix.conf.php` — фронт читает active из БД;
- TimescaleDB Community на длинной history (линейка **2.13.0–2.29.X** на 7.0.30) — осознанно, лицензия Timescale не «пакет из Debian».

**Съём на площадках:**

| Площадок | Что где | Отказ |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: server HA + PG. ЦОД-1 и ЦОД-2: **proxy group** (≥2 прокси, anti-affinity). Хосты ЦОДа — на группу | ЦОД-2 мёртв: нет съёма этой площадки. ЦОД-1 мёртв: триггеры/action молчат; прокси ЦОД-2 копят, пока хватает `ProxyOfflineBuffer` |
| **3 ЦОДа** | Proxy group в каждом | То же; третий ЦОД не даёт второго active server |

**Вариант «три независимых Zabbix»** — только при жёсткой изоляции контуров (три UI, три эскалации). По умолчанию хуже.

**Вариант «один server poll'ит три ЦОДа без прокси»** — не берём: обрыв межЦОД = массовый PROBLEM.

### Предпосылки прода

- Образы 7.0.30 со своего registry.
- NTP/chrony на server, proxy, agent, БД.
- Канал алертов (SMTP/webhook) **до** объявления HA.
- TLS на 10050/10051 **до** массовой раскатки агентов.
- Матрица с Prometheus/Wazuh: кто орёт диск, кто SNMP — до первой ночи дежурства.
- Прокси **не** в той же БД, что server. SQLite `:memory:` запрещён.

### Установка (манифесты, не вендорский Helm server)

Порядок:

1. PostgreSQL + schema Zabbix, бэкап, прогнанный restore.
2. Один server **standalone**, сменить Super Admin, HTTPS, NetworkPolicy.
3. TLS (`TLSConnect`/`TLSAccept` не `unencrypted`). Agent 2 на пару нод. PROBLEM/OK + action.
4. Proxy group в ЦОД-1, перевесить хосты, убить один прокси — хосты переезжают.
5. Вторая HA-нода **в ЦОД-1**, `HANodeName`, пустой адрес во frontend. Убить active — сверка `ha_status` и времени failover (дефолт delay **1 мин** + 5 с при исчезновении).
6. Раскатать proxy group на ЦОД-2 (и ЦОД-3). Версия прокси **=** версия сервера.
7. Официальный helm **мониторинга** K8s — поверх уже живого server, не вместо него.

Server: отдельные поды со стабильным DNS (`zabbix-server-a`, `zabbix-server-b`). **Не** Deployment с общим PVC и двумя репликами без HA-имён.

### Конфигурирование

| Параметр | Прод | Зачем |
|---|---|---|
| `HANodeName` | уникален на ноду | Без имени — standalone |
| Прокси `Server=` | все HA-имена (`,` passive / `;` active) | Standby не слушает порт |
| `ProxyBufferMode` | `hybrid` (новые 7.0) | `memory` при рестарте теряет буфер |
| `ProxyOfflineBuffer` | ≥ времени ремонта канала до мозга | Иначе дырка в history |
| Frontend :80 снаружи | нет | HTTPS |
| `EnableGlobalScripts` | 0, если нет регламента | Super Admin = скрипты на сервере |
| Elasticsearch как history | нет | В доке experimental |

Агенты: предпочитать **active checks**. Для proxy group active — агент **≥ 7.0**, доступ ко **всем** членам группы. Java gateway — только если JMX не закрыт Prometheus.

Housekeeper по умолчанию раз в час: на большой history без партиций сам становится аварией. TimescaleDB: Override item history/trend period **и** Enable internal housekeeping — иначе партиции не дропаются. Сжатие — Community Edition, как пишет requirements 7.0.30.

Секреты: макросы и пароль БД — Vault KV v2 (Zabbix только читает). При vault-макросах между server и proxy **нужен TLS**, иначе warning. Агент на той же машине, что server, — **другим** UNIX-пользователем: иначе читает `DBPassword`. `$ALLOW_HTTP_AUTH=false` во frontend, если HTTP-auth не нужен.

Failover delay: дефолт 1 мин. Ставить 10 с на WAN с джиттером — риск флаппинга (`zabbix_server -R ha_set_failover_delay=...`). Мониторить сами ноды: `zabbix[cluster,discovery,nodes]`, Reports → System information.

Upgrade major HA: **остановить все ноды**, бэкап БД, апгрейд **одной** в standalone, дождаться «server is running», вернуть HA. Не `kubectl rollout` обеих сразу.

### Масштабирование (когда появятся цифры)

1. Главный рычаг — **меньше NVPS** (interval, не плодить LLD «как Prometheus без relabel»).
2. Горизонталь съёма — **прокси**, не второй active server.
3. Вертикаль — процессы poller/trapper на **одной** active-ноде; БД отдельно.
4. Формула диска соседнего файла (~90 байт/значение как ориентир MySQL) — не смета «терабайт озера».

Таблица Small/Medium/Large вендора — **примеры старта**, не ваша смета.

### Проверка прода (пока это не пройдено — это не прод)

1. Все компоненты **7.0.30**. Нет двух server без `HANodeName` на одной БД.
2. Убить active — standby стал active, фронт нашёл его из БД, прокси не потеряли список нод.
3. Убить 1 прокси группы — хосты на живом. Рестарт прокси с `hybrid` — буфер не обнулён как при `memory`.
4. Restore PostgreSQL на стенде. Учение «ЦОД-1 недоступен»: что алертится, что только буферизуется.
5. Нет `Admin`/`zabbix`; 10051/443 не с мира; TLS не `unencrypted` на боевых агентах.

### Сильные / слабые стороны (мозг в одном ЦОДе + прокси на площадках)

| Сильное | Слабое |
|---|---|
| Официальная модель distributed monitoring | Мозг упирается в PostgreSQL этого ЦОДа |
| Buffer прокси при обрыве канала | Триггеры/action в это время могут молчать |
| HA server внутри ЦОДа, без stretch БД | Standby не принимает 10051; кривой LB ломает модель |
| Один UI, одна эскалация | Ошибка шаблона сразу на всех площадках |

**Не готов к проду**, если: пароль `zabbix`, HTTP UI снаружи, два server без `HANodeName`, прокси в БД server, `ProxyBufferMode=memory` как единственный буфер площадки, нет бэкапа PG, server+PG stretch на 2–3 ЦОДа, версии прокси ≠ сервер, community Helm выдан за вендорский оператор 7.0, тот же набор алертов без договора с Prometheus.

---

## Источники

- 7.0.30: https://www.zabbix.com/download_sources
- Native HA: https://www.zabbix.com/documentation/7.0/en/manual/concepts/server/ha
- Requirements, PostgreSQL 13–18: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
- Контейнеры: https://www.zabbix.com/documentation/7.0/en/manual/installation/containers
- Proxy / proxy groups: https://www.zabbix.com/documentation/7.0/en/manual/concepts/proxy
- Helm мониторинга K8s (не server): https://git.zabbix.com/projects/ZT/repos/kubernetes-helm/browse
- Образы: https://hub.docker.com/u/zabbix
- Правила: `Zabbix.md`
