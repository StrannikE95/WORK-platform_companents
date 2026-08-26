# Zabbix 7.0.30 LTS — термины и сокращения

Словарь к файлу `Zabbix.md`. Список линейный, не таблица. Сверху — слова, без которых не читаются остальные. Аналогий нет: только что это делает в коде и на диске.

---

**Процесс** — запущенная программа. Zabbix — несколько процессов, не один бинарь: server, frontend (PHP), proxy, agent, опционально Java gateway / web service / Kafka connector.

**Файл / каталог на диске** — история, тренды, события и таблица HA-нод живут в **PostgreSQL**, не в процессе server. Прокси имеет **свою** БД. In-memory SQLite (`:memory:`) официально запрещён (мультипроцесс).

**TCP-порт** — контракт сети: агент **10050** (passive checks); server/proxy/trapper **10051** (active proxy/agent, trapper, связь прокси↔сервер); Java gateway **10052**; web service **10053**; frontend **80/443**. TLS **не** открывает новый порт: тот же 10050/10051 принимает plaintext и TLS. Firewall режет по адресам и `TLSAccept`.

**ЦОД** — отдельная площадка. Пока RTT неизвестен, честный сбор — **прокси в каждом ЦОДе**, а не один server, который poll'ит три площадки вслепую.

**RTT** — время туда-обратно. Порога в мс у Zabbix нет. Влияет на item timeout, HA heartbeat через БД, sync-реплику PostgreSQL. Мерить путь прокси→server:10051, server→PostgreSQL, frontend→PostgreSQL, не ping ICMP.

**TLS 1.2/1.3** — GnuTLS/OpenSSL. `TLSConnect` / `TLSAccept`: `unencrypted`, `psk`, `cert`. Для PFS на PSK: GnuTLS или OpenSSL **≥ 1.1.0**. LibreSSL для agent 2 **не** поддерживается. ГОСТ не заявлено. К БД: `DBTLSConnect=verify_full`.

**NTP / chrony** — вендор *strongly recommended*. Без общего времени триггеры и HA-heartbeat живут в кривой вселенной.

**LTS** — линия долгой поддержки. 7.0.30: полная поддержка до **30 июня 2027**, ограниченная — до **30 июня 2029**. **7.4.14** — standard-линия, не LTS (полная до выхода 8.0 LTS; ограниченная заявлена на **Q4 2026**). Линия **8.0 LTS** в политике Q3 2026, на странице исходников на дату файла её **нет**. В прод файла — **7.0.30**.

**AGPLv3** — лицензия с 7.0. Для контура с госинтеграциями юридический факт; файл не заменяет юрзаключение.

**Хост (host)** — то, что наблюдаете: сервер, под, коммутатор, URL интеграционного API. Не путать с hostname Linux.

**Элемент данных (item)** — одна проверка на хосте: CPU, `net.tcp.service`, SNMP OID, HTTP-код. Payload ответа ведомства в item класть не надо: ПДн и раздувание history.

**Триггер (trigger)** — условие «это плохо»: выражение над item'ами. Сработал → **проблема (problem)**.

**Событие (event)** — факт: триггер сработал / восстановился, авторегистрация и т.д.

**Медиа / action** — куда и как слать: почта, webhook, скрипт. Без action триггер виден только в UI. On-call без media делает HA бессмысленной.

**NVPS** — New Values Per Second: сколько новых значений в секунду падает в сервер. Грубо `число item / средний интервал`. Главный рычаг нагрузки на БД.

**History** — сырые значения item'ов за N дней. Потом housekeeper (или drop partition в TimescaleDB) их вычищает.

**Trends** — почасовые min/max/avg/count. Период часа **не** настраивается. Нужны для длинных графиков.

**Housekeeper** — процесс сервера: удаляет устаревшие history/trends/events. По умолчанию раз в час. На большой БД без партиций он сам становится аварией. Не крутить `HousekeepingFrequency` в ноль и забывать.

**Zabbix server** — считает триггеры, пишет историю, шлёт action. **Не** кластер с кворумом шардов. Горизонтально сбор — **прокси**, не «ещё один active server». Вертикаль: процессы poller/trapper/db syncer на **одной** active-ноде.

**Native HA** — несколько процессов `zabbix_server` на **одной** БД. Живой сбор и триггеры — только у **одной** active-ноды; остальные **standby**. Цитата: *Only one node can be active at a time*. Opt-in, *will work across sites*, без особых требований к СУБД, которые Zabbix recognizes. Два `replicas: 2` без `HANodeName` = два писателя на одну БД или два независимых сервера.

**HANodeName** — уникальное имя ноды HA. Без него сервер стартует **standalone**. Стабильный DNS (`zabbix-server-a`), не случайные ReplicaSet-имена.

**NodeAddress** — `адрес:порт` ноды; фронтенд ходит на **активную** по записи в БД. При HA **не** прописывать `address:port` сервера в `zabbix.conf.php`.

**Failover delay** — сколько ждать, прежде чем standby станет active, если старая active исчезла молча. Диапазон **10 с – 15 мин**, дефолт **1 мин**. Менять: `zabbix_server -R ha_set_failover_delay=...`. Ставить 10 с на WAN с джиттером — риск флаппинга.

**Heartbeat HA** — active пишет last access каждые **5 секунд**. Standby смотрит на timestamp. Корректная остановка active (статус stopped) → другая нода примерно за **5 с**. Исчезновение (kill -9, сеть, ЦОД) → **failover delay + 5 с**. Связь active с БД потеряна дольше `failover delay − 5 с` → active **обязан** бросить работу и уйти в standby.

**Standby** — нода HA, которая **не** собирает данные, **не** слушает порты, почти не держит коннекты к БД. Живёт только HA-manager. Нельзя повесить TCP-балансировщик «на все HA-поды». Прокси и агенты получают **список имён нод** (`Server=` через запятую для passive, через `;` для active proxy / `ServerActive`).

**Frontend** — PHP-UI + JSON-RPC API. Данных мониторинга в нём нет — они в БД. Несколько копий = несколько PHP-процессов на ту же БД. Сессии PHP по умолчанию локальные: без sticky на LB или общего session store дежурный будет вылогиниваться. PHP **8.0–8.5** (8.5.x с 7.0.25), Apache 2.4+ или Nginx 1.20+.

**Zabbix proxy** — посредник: собирает у хостов **своего** контура, буферизует, отдаёт серверу. Своя **отдельная** БД (не серверная!). Версия прокси = версия сервера. Не считает глобальные триггеры и не шлёт action'ы.

**Active proxy** — прокси сам стучится на сервер (исходящее из ЦОДа через firewall).

**Passive proxy** — сервер стучится на прокси.

**Proxy group** — группа прокси 7.0+: сервер сам раскладывает хосты и делает failover, если прокси молчит дольше failover period (дефолт **1 мин**, диапазон 10 с – 15 мин). Хосты вешают **на группу**. Агент 7.0+ в active mode должен достучаться **до всех** членов группы. Passive: в `Server=` агента перечислить все прокси группы. Окружение (Java/SNMP) на членах группы — одинаковое.

**ProxyBufferMode** — куда прокси кладёт ещё не отгруженные данные: `disk` / `memory` / `hybrid`. Новые установки 7.0: **hybrid**. Режим `memory` при рестарте **теряет** буфер.

**ProxyMemoryBufferSize** — размер RAM-части hybrid-буфера. Ненулевой на проде.

**ProxyOfflineBuffer** — сколько часов прокси хранит данные, если сервер недоступен. ≥ времени, за которое чините канал до HA-нод server. Дефолт смотреть в `zabbix_proxy.conf` сборки.

**Agent / Agent 2** — агент на машине. Порт **10050**. Agent 2 — на Go, с плагинами (PostgreSQL, MongoDB, Redis и др.). Предпочтителен. Агенты — Linux и Windows (agent 2: Windows 10 / Server 2016+). Сервер: UNIX; Windows как платформа *server* не заявлена. Агент на той же машине, что server/proxy — **другим UNIX-пользователем**: иначе читает `DBPassword`.

**Passive check** — сервер/прокси **спрашивает** агента. Нужен доступ к 10050 хоста.

**Active check** — агент **сам** забирает список item'ов и присылает значения. Список нод — в `ServerActive`. Для proxy group в active-режиме нужен агент **≥ 7.0**. Предпочитать active: исходящие из ЦОДа проще, чем открыть 10050 всех подов наружу.

**Trapper** — приём значений, которые **пушат** (`zabbix_sender`). Тот же порт **10051**.

**Java gateway** — отдельный процесс для JMX. Порт **10052**. Упал = слепой JMX, не весь Zabbix. Kafka, Camunda, Tomcat — если JMX не закрыт Prometheus JMX exporter.

**Web service** — генерация PDF-отчётов (нужен Chrome). Порт **10053**. К HA сбора не относится.

**LLD** — Low-level discovery: сам нашёл диски/интерфейсы/поды и завёл item'ы. Не плодить LLD «по каждому поду каждую метрику как в Prometheus без relabel».

**Template** — набор item/trigger/LLD, который вешают на хосты.

**PSK** — общий секрет для TLS. Identity **не** секрет: его шлют открытым текстом. Ротация — ручная дисциплина.

**TLSConnect / TLSAccept** — чем шифруем исходящее / что принимаем входящим.

**Vault (в Zabbix)** — подтягивание секретов из HashiCorp Vault KV v2 или CyberArk. Zabbix только **читает**. Между server и proxy при vault-макросах **нужен TLS**, иначе warning. Кэш `$DB['VAULT_CACHE']` — осознанно (TTL, PrivateTmp). Макросы не обновятся до `secrets_reload` / cache, если Vault лёг.

**Connector** — поток item values / events наружу по HTTP NDJSON. Для Kafka вендор даёт отдельный **Kafka connector** (Go), это не брокер и не часть `zabbix_server`. URL, TLS verify, ACL на приёмнике; не «в Kafka без ACL».

**Super Admin** — учётка, которой можно всё, включая скрипты на сервере. Дефолт: пользователь **`Admin`**, пароль **`zabbix`**. Сменить **сразу** после первого логина. После 5 неудачных логинов UI молчит **30 секунд**.

**Internal auth** — дефолт. Пароль: минимум 8, обрезка после 72 символов; сложность включается в Users → Authentication. `$ALLOW_HTTP_AUTH=false` в `zabbix.conf.php` выключает HTTP auth (setup.php параметр сотрёт).

**SAML / SSO** — вход людей (в доке Entra ID / Okta / OneLogin). API-токены для автоматизации, не пароль Super Admin в CI.

**EnableGlobalScripts=0** — выключить глобальные скрипты на сервере, если не нужны. На прокси remote commands по умолчанию выкл. На агенте `system.run` / `AllowKey=system.run` по умолчанию выкл.

**iframe / same-origin** — frontend в iframe с другого домена режется; страница **того же** домена внутри фрейма получит JS-доступ к UI.

**PostgreSQL 13–18** — СУБД сервера в этом файле (18.x с 7.0.20). Oracle с 7.0 **deprecated**. Native HA Zabbix БД **не** кластеризует. Heartbeat идёт **через БД**: мёртвая БД = не из чего выбрать лидера. Роли least privilege (`zbx_srv` / `zbx_web`). `work_mem` вендор предлагает поднимать с дефолта **4 МБ** на больших инсталляциях — по плану запросов.

**TimescaleDB Community Edition** — линейка **2.13.0–2.29.X** на 7.0.30 (2.29.X — с 7.0.30). Отдельная лицензия Timescale, не «просто расширение из Debian». Партиции + compression; housekeeper **дропает партиции**, если включены Override history/trend period **и** internal housekeeping. Без Community нет официально требуемого compression path. Elasticsearch как боевое history — **experimental**, в прод файла не кладём.

**SQLite3 / MySQL на прокси** — допустимы для прокси. Нельзя указать ту же БД, что у сервера. SQLite3 на маленьком прокси (PVC); PostgreSQL — если прокси крупный.

**Формулы диска** — ориентир **~90 байт/значение** (MySQL; для PostgreSQL «может отличаться»): `history ≈ days × (items / refresh_rate) × 24 × 3600 × ~90`; trends с делением items/3600; events ~330 + tags×100; config обычно ≤ 10 МБ. Пример доки: 3000 item / 60 с = 50 NVPS, 30 дней history ≈ 10.9 ГиБ. Текст/лог: ориентир **~500 байт** на значение, «точно предсказать нельзя».

**Sizing-примеры вендора** — Small 1 000 метрик / 2 CPU / 8 ГиБ; Medium 10 000 / 4 / 16; Large 100 000 / 16 / 64; Very large 1 000 000 / 32 / 96. 1 «метрика» = 1 item + 1 trigger + 1 graph. Amazon EC2 в таблице — иллюстрация класса. *each installation is unique*.

**Официальные образы** — `zabbix-server-pgsql`, `zabbix-web-nginx-pgsql` (или Apache), `zabbix-proxy-sqlite3` / `zabbix-proxy-pgsql`, `zabbix-agent2`, `zabbix-java-gateway`, `zabbix-web-service`. Пин `alpine-7.0.30` / `ubuntu-7.0.30`, не `latest`. Compose: github.com/zabbix/zabbix-docker, ветка **7.0**.

**Helm ZT/kubernetes-helm** — ставит компоненты **мониторинга Kubernetes из уже существующего** Zabbix, не сам сервер. OpenShift Operator в старых мануалах — линии 5.0/6.0, для 7.0 не брать. Официального оператора «весь Zabbix 7.0 в прод» нет.

**CSI / RWO / emptyDir** — том PostgreSQL не NFS на два primary и не emptyDir.

**PDB / topology spread** — не снять сразу оба HA-сервера и не снять всю proxy group одной зоны; два прокси группы не на одной ноде.

**OOM active-ноды** — failover через delay, не «мгновенно».

**Major upgrade HA** — остановить все ноды, бэкап БД, апгрейд **одной** в standalone (`HANodeName` закомментировать), дождаться «server is running» в System information, вернуть HA, потом остальные. Minor: по очереди, начиная с живой.

**Мониторинг HA** — `zabbix[cluster,discovery,nodes]`, Reports → System information, `ha_status`.

**SNMP / ICMP / HTTP checks** — съём железа и доступности «снаружи процесса». Интервал и timeout HTTP ведомств завязать на известные лаги интеграционного API.

**Prometheus** — другой контур (отдельный документ). Два полных контура «снимаем всё и орём обо всём» = двойные инциденты. Типичная нарезка файла: Prometheus — кардинальность микросервисов и K8s `/metrics`; Zabbix — железо, SNMP, ICMP, сертификаты, «порт ведомства открыт», эскалации для NOC. Это допущение, не требование вендора.

**Wazuh / SIEM / Falco** — другой контур, не Zabbix.

**SoT клиентских данных** — не Zabbix. Он хранит *метрики наблюдения*.

**Вариант A** — server HA + PostgreSQL в одном логическом контуре, съём локальный: proxy group в каждом ЦОДе. A1: primary+sync replica в двух ЦОДах после замера; A2: primary в одном ЦОДе + async replica + прогнанный failover (RPO > 0).

**Вариант B** — три независимых Zabbix. Обычно хуже для on-call.

**Вариант C** — один server без прокси, poll напрямую в три ЦОДа. Только если RTT стабильно меньше item timeout.

**RPO / RTO** — сколько данных готовы потерять / как быстро восстановиться. «Система 24/7» ≠ «ни один item не пропущен»: прокси буферизует; триггеры на сервере в это время могут не считаться. Пережить 2 из 3 ЦОДов native HA **не обещает**.

**Guest** — учётка UI, если включена. На проде не нужна без регламента.

**152-ФЗ** — срок хранения history и состав item'ов (не класть ПДн в текст item).

Источники формулировок: глоссарий и тело `Zabbix.md`. Новых порогов RTT и размеров диска здесь нет.
