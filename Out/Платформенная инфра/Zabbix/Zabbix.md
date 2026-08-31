# Zabbix 7.0.30 LTS — назначение и состав

```mermaid
stateDiagram-v2
    state "Docker Engine" as Docker_Engine
    state "Docker Compose" as Docker_Compose
    Zabbix --> Docker_Engine
    Zabbix --> Docker_Compose
    Zabbix --> PostgreSQL
    Zabbix --> PHP
    Zabbix --> Nginx
```

Zabbix — self-hosted-система наблюдения за инфраструктурой и сервисами. Для платформы зафиксирована версия **7.0.30 LTS**; это не Zabbix Cloud. Серверная часть использует модель active-passive: в каждый момент сбор, обработку и расчёт триггеров выполняет только одна активная нода.

Документация: https://www.zabbix.com/documentation/7.0/en/manual/  
Официальные образы: `zabbix/zabbix-server-pgsql`, `zabbix/zabbix-web-nginx-pgsql` или вариант с Apache, `zabbix/zabbix-proxy-sqlite3` / `zabbix/zabbix-proxy-pgsql`, `zabbix/zabbix-agent2`, `zabbix/zabbix-java-gateway`, `zabbix/zabbix-web-service`. Для воспроизводимости используется метка **7.0.30**, а не `latest`.

Zabbix **не** заменяет Prometheus, SIEM Wazuh, хранилище логов OpenSearch или шину Kafka.

---

## Назначение системы

Zabbix нужен, чтобы:

- проверять доступность машин, сетевых устройств, портов, URL и прикладных интерфейсов;
- получать технические метрики и состояние ОС;
- выявлять отклонения по заданным правилам;
- сохранять историю наблюдений и агрегированные значения;
- показывать состояние в веб-интерфейсе и передавать проблемы дежурным.

Система хранит **данные мониторинга**, а не эталонные клиентские карточки. Она не исполняет бизнес-процессы и не получает бизнес-данные напрямую из государственных систем. Zabbix может проверить доступность интеграционного API и опционально передать значения или события во внешнюю Kafka через отдельный коннектор.

Метрики Kubernetes и микросервисов уже собирает Prometheus. Границы ответственности нужно разделить: например, Zabbix отвечает за ОС, SNMP и доступность портов, а Prometheus — за высококардинальные метрики приложений. Иначе один отказ породит дублирующиеся оповещения.

---

## Перечень функций

Self-hosted Zabbix 7.0 умеет:

1. **Получать данные** через Agent/Agent 2, SNMP, ICMP, HTTP, JMX через Java gateway и `zabbix_sender`.
2. **Вычислять триггеры** — правила, переводящие значения в состояние проблемы и обратно.
3. **Эскалировать проблемы** через почту, webhook или скрипт.
4. **Хранить историю** исходных значений и trends — почасовые минимум, максимум, среднее и количество значений.
5. **Работать в native HA**: несколько экземпляров `zabbix_server` используют одну базу, но активен только один.
6. **Собирать данные через proxy** и сохранять их в локальном буфере при разрыве связи с server.
7. **Объединять proxy в группы**, чтобы перераспределять наблюдаемые хосты при недоступности одного proxy.
8. **Автоматически обнаруживать объекты** и назначать им шаблоны проверок.
9. **Шифровать взаимодействие** Agent/Proxy/Server с помощью PSK или сертификатов на тех же портах 10050/10051.
10. **Читать секреты** из HashiCorp Vault или CyberArk.
11. **Передавать значения и события наружу** по HTTP; Kafka используется через отдельный коннектор.
12. **Формировать PDF-отчёты** через отдельный Zabbix web service.

Zabbix не создаёт многопишущий кластер наподобие Kafka: запасные server-ноды не делят нагрузку с активной. История и состояние HA находятся в СУБД.

---

## Основные элементы системы и зависимости

### Схема инстансов и потоков

**Допущение:** одна площадка. Мозг (native HA server + PostgreSQL) и наблюдаемые хосты живут в одном дата-центре. Stretch server и синхронный Postgres на 2–3 зала на этой схеме **нет**: порога RTT у вендора нет ([HA](https://www.zabbix.com/documentation/7.0/en/manual/concepts/server/ha)).

Имена внутри блоков — роли/инстансы, не обязательные DNS-имена. Сплошная стрелка — рабочий поток; пунктир — опциональный путь или нерабочий standby. Направление стрелки — кто открывает соединение.

**Сильная сторона:** агенты ходят к server напрямую, сеть короче, HA остаётся внутри одного зала. **Слабая:** нет буфера «чужого зала»; падение площадки вместе с writer Postgres останавливает обработку. **Критично:** TCP **10051** не балансировать на ACTIVE и STANDBY сразу; UI и 10051 не публиковать в интернет.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#d9ead3", "primaryTextColor": "#000000", "primaryBorderColor": "#38761d", "lineColor": "#5b5b5b", "secondaryColor": "#cfe2f3", "tertiaryColor": "#fff2cc"}}}%%
flowchart LR
  subgraph obs["Наблюдаемые системы — внешние"]
    host1["host-01<br/>наблюдаемая ОС"]
    agent1["zabbix-agent2-host01"]
    snmp1["net-snmp<br/>SNMP-устройства"]
    api1["api-endpoint<br/>HTTP"]
    jvm1["jvm-app<br/>JMX endpoint"]
  end

  subgraph core["Контур Zabbix — площадка 1"]
    srv1["zabbix-server-01<br/>ACTIVE"]
    srv2["zabbix-server-02<br/>STANDBY"]
    fe1["zabbix-web-01<br/>PHP frontend / API"]
    fe2["zabbix-web-02<br/>PHP frontend / API"]
    jg["zabbix-java-gateway-01<br/>опционально"]
    ws["zabbix-web-service-01<br/>PDF, опционально"]
    kc["zabbix-kafka-connector-01<br/>опционально"]
    pxy["zabbix-proxy-01<br/>active proxy, опционально"]
    pdb1[("proxy-db-01<br/>SQLite или PostgreSQL")]
  end

  subgraph ext["Отдельно развёрнутые системы"]
    lb["web-lb-01<br/>внешний балансировщик"]
    db[("zabbix-db<br/>PostgreSQL 13–18")]
    vault["vault<br/>хранилище секретов"]
    alert["mail / webhook<br/>канал оповещений"]
    kafka["Kafka<br/>шина событий"]
  end

  user["Браузер / API-клиент"]

  agent1 ---|"установлен на"| host1
  srv1 -->|"пассивная проверка: TCP 10050"| agent1
  agent1 -->|"активная проверка: TCP 10051"| srv1
  srv1 -->|"SNMP polling: UDP 161"| snmp1
  snmp1 -->|"SNMP trap: UDP 162"| srv1
  srv1 -->|"HTTP(S): 80/443"| api1

  pxy --- pdb1
  pxy -.->|"альтернатива: TCP 10050"| agent1
  agent1 -.->|"альтернатива: TCP 10051"| pxy
  pxy -.->|"TCP 10051 TLS<br/>соединение открывает proxy"| srv1

  srv1 -->|"SQL: TCP 5432"| db
  srv2 -.->|"HA manager: SQL / TCP 5432"| db
  fe1 -->|"SQL: TCP 5432"| db
  fe2 -->|"SQL: TCP 5432"| db
  fe1 -->|"active NodeAddress: TCP 10051"| srv1
  fe2 -->|"active NodeAddress: TCP 10051"| srv1

  user -->|"HTTPS 443"| lb
  lb -->|"HTTP(S), порт развёртывания"| fe1
  lb -->|"HTTP(S), порт развёртывания"| fe2

  srv1 -->|"JMX-запрос: TCP 10052"| jg
  jg -->|"JMX, настроенный порт"| jvm1
  srv1 -->|"TCP 10053"| ws
  ws -->|"страница отчёта: HTTPS 443"| lb
  srv1 -->|"HTTP(S), NDJSON"| kc
  kc -->|"Kafka protocol: 9092/9093*"| kafka
  srv1 -->|"SMTP или HTTPS webhook"| alert
  srv1 -->|"HTTPS API"| vault

  zbxLegend["Zabbix: обязательный компонент"]
  optLegend["Zabbix: опциональный компонент"]
  extLegend["Внешняя система / отдельное ПО"]
  dataLegend[("Отдельное хранилище данных")]

  style srv1 fill:#d9ead3,stroke:#38761d,color:#000000,stroke-width:2px
  style srv2 fill:#d9ead3,stroke:#38761d,color:#000000,stroke-width:2px
  style fe1 fill:#d9ead3,stroke:#38761d,color:#000000,stroke-width:2px
  style fe2 fill:#d9ead3,stroke:#38761d,color:#000000,stroke-width:2px
  style zbxLegend fill:#d9ead3,stroke:#38761d,color:#000000,stroke-width:2px

  style agent1 fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3
  style pxy fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3
  style jg fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3
  style ws fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3
  style kc fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3
  style optLegend fill:#fff2cc,stroke:#bf9000,color:#000000,stroke-width:2px,stroke-dasharray:5 3

  style host1 fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style snmp1 fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style api1 fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style jvm1 fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style lb fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style vault fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style alert fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style kafka fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style user fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px
  style extLegend fill:#cfe2f3,stroke:#0b5394,color:#000000,stroke-width:2px

  style db fill:#ead1dc,stroke:#741b47,color:#000000,stroke-width:2px
  style pdb1 fill:#ead1dc,stroke:#741b47,color:#000000,stroke-width:2px
  style dataLegend fill:#ead1dc,stroke:#741b47,color:#000000,stroke-width:2px
```

\* `9092/9093` — частые listeners Kafka; фактический адрес и TLS задаёт кластер Kafka, это не порты Zabbix. Порты Zabbix по умолчанию: [requirements](https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements).

**Легенда:**

- <span style="color:#38761d">■</span> **Зелёный** — процессы Zabbix, без которых принятый контур server/frontend не работает.
- <span style="color:#bf9000">■</span> **Жёлтый, пунктирная рамка** — компоненты поставки Zabbix, которые на одной площадке не обязательны (Agent 2, proxy, Java gateway, PDF, Kafka connector).
- <span style="color:#0b5394">■</span> **Синий** — отдельно развёрнутые системы и наблюдаемые цели. Native HA Zabbix их не кластеризует.
- <span style="color:#741b47">■</span> **Розовый цилиндр** — хранилище данных, которое не является процессом `zabbix_server`.

### Как читать схему

1. **Всё, кроме браузера оператора, нарисовано в одной площадке.** Второй и третий дата-центр сюда не входят: для них потребовались бы proxy в чужом зале, а не второй writer server.
2. **Цвет блока важнее рамки subgraph.** Зелёный — продукт. Жёлтый — тот же продукт, но его можно не ставить. Синий и розовый — чужой жизненный цикл (СУБД, брокер, балансировщик, наблюдаемый хост).
3. **Стрелка читается как «кто звонит».** `zabbix-server-01 → zabbix-agent2-host01` значит: server открывает TCP **10050** к агенту. Обратная сплошная стрелка — другой режим: агент сам открывает **10051**.
4. **Основной съём на одной площадке — напрямую на ACTIVE server.** Хост, SNMP-устройство и HTTP endpoint не обязаны ходить через proxy. Proxy на схеме жёлтый и пунктирный: это **альтернативное назначение хоста**, не второй параллельный съём того же item.
5. **Пассивная проверка Agent 2:** server (или proxy, если хост отдан proxy) соединяется с агентом на **10050**, запрашивает ключ item и забирает значение. ([active/passive](https://www.zabbix.com/documentation/7.0/en/manual/appendix/items/activepassive))
6. **Активная проверка Agent 2:** агент сам соединяется на **10051**, получает список проверок и отправляет значения. Слова «активный/пассивный» говорят, кто начал TCP-сессию, а не насколько агент важен.
7. **SNMP и HTTP — agentless.** Server сам опрашивает устройство по UDP **161** или HTTP(S). Trap (сообщение от устройства) приходит на UDP **162**, обычно через `snmptrapd` рядом с server; это не порт процесса `zabbix_server` из таблицы вендора.
8. **Опциональный active proxy** сам открывает TLS на **10051** к текущей ACTIVE-ноде, забирает конфигурацию и отдаёт буфер. У каждого proxy своя база (`proxy-db-01`): SQLite-файл или отдельный PostgreSQL/MySQL. Её нельзя совместить с `zabbix-db`. ([proxy](https://www.zabbix.com/documentation/7.0/en/manual/concepts/proxy))
9. **STANDBY не параллельный сборщик.** `zabbix-server-02` держит только HA manager и смотрит пульс в той же PostgreSQL. Он **не слушает** рабочие порты и не принимает агентов. После переключения те же стрелки съёма ведут уже к нему как к новой ACTIVE; отдельная «линия на будущее» на схеме не рисуется. ([HA](https://www.zabbix.com/documentation/7.0/en/manual/concepts/server/ha))
10. **Frontend не ходит в server за историей.** PHP читает конфигурацию, историю и таблицу `nodes` из `zabbix-db`. К server на **10051** frontend обращается только чтобы узнать, что ACTIVE жив, по `NodeAddress`. В `zabbix.conf.php` адрес:порт server при HA **не** зашивают — иначе UI останется на мёртвой ноде.
11. **Балансировщик — только для людей и API, не для 10051.** `web-lb-01` распределяет HTTPS между копиями frontend. Поставить его TCP-балансировщиком на ACTIVE+STANDBY ломает модель: standby порт не слушает.
12. **Отказ UI не останавливает съём.** Упал frontend или балансировщик — оператор не видит экран; ACTIVE server при живой БД продолжает проверки, триггеры и действия.
13. **JMX — отдельная жёлтая ветка.** Server спрашивает Java gateway на **10052**; gateway говорит с JVM по настроенному JMX-порту. Без Java-приложений этот квадрат не ставят.
14. **PDF — тоже отдельная ветка.** Server зовёт web service на **10053**; тот открывает страницу frontend (через LB) и печатает отчёт. Это не путь сбора метрик.
15. **Kafka connector — не брокер и не часть server.** Server шлёт ему NDJSON по HTTP(S); коннектор уже публикует в внешнюю Kafka. Падение Zabbix шину Kafka не роняет; отсутствие коннектора штатный мониторинг не ломает. ([streaming](https://www.zabbix.com/documentation/7.0/en/manual/config/export/streaming))
16. **Vault и почта/webhook — исходящие вызовы ACTIVE server.** Секреты макросов читаются из внешнего хранилища; проблема уходит дежурному по SMTP или HTTPS webhook. Их доступность native HA не обеспечивает.

Термины схемы вынесены в глоссарий в конце документа.

### Zabbix server: активная нода

- **Экземпляр на схеме:** `zabbix-server-01`.
- **Назначение:** центральная обработка проверок, расчёт триггеров, создание событий, выполнение действий, запись истории и раздача конфигурации proxy.
- **Технологии и варианты:** процесс `zabbix_server`; пакет Linux либо официальный контейнер `zabbix/zabbix-server-pgsql`. Для выбранной архитектуры — вариант с PostgreSQL.
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** обязателен; активной одновременно может быть только одна server-нода.

### Zabbix server: standby-нода

- **Экземпляр на схеме:** `zabbix-server-02`.
- **Назначение:** следить за состоянием native HA через общую БД и принять роль ACTIVE при переключении.
- **Технологии и варианты:** тот же бинарь и версия, что у активной ноды; задаются уникальный `HANodeName` и доступный `NodeAddress`.
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** в продукте native HA — opt-in (`HANodeName`). На принятой схеме площадки standby показан как часть контура. Он не делит нагрузку и до переключения не слушает рабочие порты.

### Zabbix proxy

- **Экземпляр на схеме:** `zabbix-proxy-01`.
- **Назначение:** выполнять проверки рядом с целями, локально буферизовать результаты и передавать их server. На одной площадке это способ разгрузить ACTIVE server или пережить короткий обрыв до него, а не замена второго дата-центра.
- **Технологии и варианты:** процесс `zabbix_proxy` в active- или passive-режиме; официальные контейнеры/пакеты `zabbix-proxy-sqlite3` / `zabbix-proxy-pgsql` (есть и вариант с MySQL). На схеме выбран active proxy: соединение к server на TCP **10051** открывает сам proxy. Версия proxy должна совпадать с server (**7.0.30**).
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** опционален. На одной площадке хосты могут быть назначены напрямую ACTIVE server. Группа из нескольких proxy — отдельное решение, на схеме не нарисована.

### Локальная база proxy

- **Экземпляр на схеме:** `proxy-db-01`, отдельная база этого proxy.
- **Назначение:** хранить локальную конфигурацию proxy и очередь данных до отправки server.
- **Технологии и варианты:** встроенный файл SQLite либо отдельно запущенные PostgreSQL/MySQL. SQLite in-memory вендор для боя не предлагает.
- **Принадлежность:** хранилище обслуживает proxy; выбранная СУБД — отдельное ПО, SQLite — библиотечная зависимость варианта proxy.
- **Обязательность:** обязательна, **если** proxy включён. Базу proxy нельзя совмещать с `zabbix-db`.

### Zabbix Agent 2

- **Экземпляр на схеме:** `zabbix-agent2-host01`, установлен на внешнем `host-01`.
- **Назначение:** читать метрики ОС и приложений через плагины и отдавать их в passive-режиме (server/proxy стучится на **10050**) или active-режиме (агент сам идёт на **10051**).
- **Технологии и варианты:** `zabbix_agent2` для Linux и Windows; классический `zabbix_agentd` остаётся альтернативой, для новых интеграций предпочтителен Agent 2. Официальный образ `zabbix/zabbix-agent2`, метка **7.0.30**.
- **Принадлежность:** входит в Zabbix, хотя ставится на чужую машину.
- **Обязательность:** опционален: хост можно наблюдать по SNMP, HTTP, ICMP или другим agentless-проверкам.

### Zabbix frontend и API

- **Экземпляры на схеме:** `zabbix-web-01` и `zabbix-web-02`.
- **Назначение:** веб-интерфейс операторов и HTTP API для автоматизации. Собственного хранилища истории не имеет.
- **Технологии и варианты:** PHP с Nginx или Apache; официальные образы `zabbix-web-nginx-pgsql` и соответствующие Apache-варианты. Несколько frontend-инстансов могут работать с одной БД.
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** формально server может собирать данные без frontend, но для администрирования и штатной работы интерфейс практически необходим. Горизонтальное дублирование frontend опционально.

### Zabbix DB

- **Экземпляр на схеме:** логическая точка `zabbix-db`.
- **Назначение:** хранить конфигурацию, историю, trends, события, проблемы и таблицу состояния HA server.
- **Технологии и варианты:** в принятой схеме PostgreSQL **13–18** ([requirements](https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements)); опционально TimescaleDB Community для партиционирования и сжатия истории. Вендор также поддерживает MySQL/Percona, MariaDB и Oracle (Oracle deprecated с 7.0); на схеме они не показаны.
- **Принадлежность:** PostgreSQL и TimescaleDB — отдельное ПО, не процессы Zabbix.
- **Обязательность:** постоянная БД обязательна. Native HA server не создаёт HA базы и не заменяет её резервирование.

### Внешний балансировщик

- **Экземпляр на схеме:** `web-lb-01`.
- **Назначение:** единая HTTPS-точка входа и распределение запросов между frontend-инстансами.
- **Технологии и варианты:** платформенный ingress, HAProxy, Nginx, аппаратный или облачный балансировщик.
- **Принадлежность:** не входит в Zabbix.
- **Обязательность:** опционален при одном frontend; нужен для единой точки входа к нескольким frontend. Он не должен балансировать TCP 10051 между ACTIVE и STANDBY server-нодами.

### Java gateway

- **Экземпляр на схеме:** `zabbix-java-gateway-01`.
- **Назначение:** посредник между server/proxy и JMX-интерфейсом JVM-приложения.
- **Технологии и варианты:** процесс Zabbix Java gateway, пакет или контейнер `zabbix-java-gateway`; слушает **TCP 10052** по умолчанию.
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** опционален, нужен только для JMX-проверок.

### Zabbix web service

- **Экземпляр на схеме:** `zabbix-web-service-01`.
- **Назначение:** формировать запланированные PDF-отчёты, открывая страницу frontend в headless-браузере.
- **Технологии и варианты:** отдельный процесс/контейнер `zabbix-web-service`; слушает **TCP 10053** по умолчанию.
- **Принадлежность:** входит в Zabbix.
- **Обязательность:** опционален, если PDF-отчёты не нужны.

### Kafka connector

- **Экземпляр на схеме:** `zabbix-kafka-connector-01`.
- **Назначение:** принять поток значений и событий от Zabbix server по HTTP в формате NDJSON и опубликовать его в Kafka.
- **Технологии и варианты:** отдельный лёгкий сервер на Go из примеров/интеграций Zabbix; endpoint, аутентификация и Kafka bootstrap servers задаются отдельно.
- **Принадлежность:** не входит в основной server/frontend/proxy и развёртывается отдельным процессом.
- **Обязательность:** опционален; для штатного мониторинга не нужен.

### HashiCorp Vault или CyberArk

- **Экземпляр на схеме:** `vault`.
- **Назначение:** хранить секреты, на которые ссылаются secret macros Zabbix.
- **Технологии и варианты:** HashiCorp Vault KV v2 или CyberArk Central Credential Provider.
- **Принадлежность:** внешняя система.
- **Обязательность:** опциональна для Zabbix; в платформе предпочтительнее хранения паролей в открытых макросах.

### Наблюдаемые системы

- **Экземпляры на схеме:** `host-01`, `net-snmp`, `api-endpoint`, `jvm-app`.
- **Назначение:** источники метрик либо цели проверок на этой же площадке.
- **Технологии и варианты:** Agent 2, SNMP, ICMP, HTTP(S), JMX и другие поддерживаемые типы item.
- **Принадлежность:** сами системы внешние; установленный на хост Agent 2 относится к Zabbix.
- **Обязательность:** нужна хотя бы одна цель наблюдения; конкретный протокол и агент опциональны.

### Каналы оповещений

- **Экземпляр на схеме:** `mail / webhook`.
- **Назначение:** доставить информацию о проблеме дежурному или внешней системе управления инцидентами.
- **Технологии и варианты:** SMTP, HTTP(S) webhook, пользовательский media type или скрипт.
- **Принадлежность:** почтовая система и получатели webhook внешние; механизмы action/media type входят в Zabbix.
- **Обязательность:** технически опциональны, но без них проблема остаётся только в интерфейсе.

### Kafka

- **Экземпляр на схеме:** внешний кластер `Kafka`.
- **Назначение:** принять опубликованные коннектором значения или события для дальнейшей обработки.
- **Технологии и варианты:** Apache Kafka с настроенными bootstrap servers и принятой в платформе схемой защиты.
- **Принадлежность:** внешняя система; Zabbix не является брокером.
- **Обязательность:** опциональна и не влияет на основной цикл мониторинга.

### Браузер / API-клиент

- **Экземпляр на схеме:** `Браузер / API-клиент`.
- **Назначение:** человек или автоматизация, которые открывают UI и HTTP API.
- **Технологии и варианты:** браузер из [списка вендора](https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements) или HTTP-клиент к JSON-RPC API frontend.
- **Принадлежность:** внешний участник, не часть поставки Zabbix.
- **Обязательность:** без него система может собирать данные, но оператор экран не видит.

---

## Порты и сетевые направления

### Порты Zabbix по умолчанию

| Порт | Кто слушает | Кто подключается | Назначение |
|---|---|---|---|
| **10050/TCP** | Agent / Agent 2 | Server или proxy | Пассивная агентская проверка |
| **10051/TCP** | Активный server; active-check endpoint proxy; passive proxy | Active agent, active proxy, `zabbix_sender`; либо server для passive proxy | Приём значений, конфигурация и обмен server↔proxy |
| **10052/TCP** | Java gateway | Server или proxy | JMX-проверки через Java gateway |
| **10053/TCP** | Zabbix web service | Server | Формирование отчётов |
| **80/443 TCP** | Web server frontend или внешний вход | Браузер, API-клиент, web service | UI и HTTP API |

TLS не создаёт отдельный порт: защищённое и незащищённое взаимодействие Agent/Proxy/Server использует те же 10050/10051, а разрешённый режим задаётся настройками компонентов.

### Связанные порты внешних технологий

| Порт | Технология | Примечание |
|---|---|---|
| **5432/TCP** | PostgreSQL | Значение по умолчанию; фактический порт задаёт администратор БД |
| **161/UDP** | SNMP polling | Proxy/server отправляет запрос устройству |
| **162/UDP** | SNMP traps | Приём обычно организует `snmptrapd` рядом с server/proxy |
| **9092/9093 TCP** | Kafka | Частые варианты без TLS/с TLS; фактические listeners определяет Kafka |
| **443/TCP** | HTTPS | Webhook, Vault API, Kafka connector endpoint и пользовательский вход — по конфигурации |

---

## Глоссарий

**Active agent check** — режим, в котором Agent сам соединяется с server/proxy на 10051, получает перечень проверок и отправляет значения.

**Active proxy** — proxy, который сам открывает соединение к server на 10051 для получения конфигурации и отправки данных.

**ACTIVE server** — единственная server-нода native HA, которая сейчас выполняет сбор, обработку, триггеры и действия.

**Agent / Agent 2** — программа на наблюдаемой машине, которая читает метрики ОС и приложений. Agent 2 — более новый вариант с плагинной архитектурой.

**Frontend** — PHP-приложение с пользовательским интерфейсом и HTTP API; историю хранит не у себя, а в Zabbix DB.

**HA manager** — процесс `zabbix_server`, который координирует состояние server-нод через базу.

**History** — исходные значения item за заданный срок хранения.

**Item** — одна настроенная проверка или показатель, например свободное место на диске.

**JMX** — интерфейс управления и мониторинга Java-приложений; Zabbix обращается к нему через Java gateway.

**NDJSON** — поток JSON-объектов, где каждый объект занимает отдельную строку; используется для экспорта значений и событий.

**Native HA** — встроенный active-passive-механизм Zabbix server. Он переключает server-ноды, но не кластеризует PostgreSQL, frontend или proxy.

**NodeAddress** — адрес server-ноды, сохранённый для HA; frontend находит адрес текущей ACTIVE-ноды через таблицу `nodes`.

**Passive agent check** — режим, в котором server/proxy соединяется с Agent на 10050 и запрашивает значение.

**Passive proxy** — proxy, к которому server сам подключается на 10051 за данными и для передачи конфигурации.

**Problem** — зафиксированное событие о переходе триггера в проблемное состояние.

**Proxy** — промежуточный процесс сбора с собственной локальной базой и буфером; глобальные триггеры и действия остаются на server.

**Proxy group** — группа proxy, между которыми Zabbix может перераспределять наблюдаемые хосты.

**PSK** — предварительно согласованный секрет для TLS-аутентификации компонентов.

**Server** — центральный процесс Zabbix, который обрабатывает данные, вычисляет триггеры, создаёт события и управляет конфигурацией.

**SNMP** — протокол наблюдения за сетевыми и инфраструктурными устройствами; polling — запросы к устройству, trap — сообщение от устройства.

**STANDBY server** — резервная server-нода, где работает только HA manager; до переключения она не собирает данные и не слушает рабочие порты.

**Trend** — почасовая агрегированная запись: минимум, максимум, среднее и количество значений.

**Trigger** — логическое выражение над item, определяющее нормальное или проблемное состояние.

**Webhook** — исходящий HTTP(S)-вызов во внешнюю систему при выполнении действия.

---

## Источники

- Версия 7.0.30: https://www.zabbix.com/download_sources
- Жизненный цикл и лицензия: https://www.zabbix.com/life_cycle_and_release_policy
- Zabbix server и native HA: https://www.zabbix.com/documentation/7.0/en/manual/concepts/server/ha
- Proxy и группы proxy: https://www.zabbix.com/documentation/7.0/en/manual/concepts/proxy; https://www.zabbix.com/documentation/7.0/en/manual/distributed_monitoring/proxies/ha
- Agent, active и passive checks: https://www.zabbix.com/documentation/7.0/en/manual/appendix/items/activepassive
- Frontend: https://www.zabbix.com/documentation/7.0/en/manual/installation/frontend
- Поддерживаемые СУБД и порты: https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements
- Java gateway: https://www.zabbix.com/documentation/7.0/en/manual/concepts/java
- Web service: https://www.zabbix.com/documentation/7.0/en/manual/appendix/config/zabbix_web_service
- Поток во внешние системы и Kafka connector: https://www.zabbix.com/documentation/7.0/en/manual/config/export/streaming
- Секреты: https://www.zabbix.com/documentation/7.0/en/manual/config/secrets
- Шифрование: https://www.zabbix.com/documentation/7.0/en/manual/encryption
