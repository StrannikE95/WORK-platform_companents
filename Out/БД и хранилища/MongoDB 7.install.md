# MongoDB Community Server 7.0.40 — установка и конфигурирование

Связанный документ (глоссарий, HA, безопасность, почему так): `MongoDB 7.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного replica set (голосующие члены в разных ЦОДах, схема 1-1-1 / 2-2-1 / PSA) **не делаем**: межЦОДовый RTT для heartbeat (дефолт **2 с**) и выборов (`electionTimeoutMillis` **10000**) неприемлем; порога миллисекунд у MongoDB **нет**.

Версия: **MongoDB Community Server 7.0.40**. Образ: `mongodb/mongodb-community-server:7.0.40-ubi9-slim` (не `7.0-ubi9-slim`, не `latest`, не устаревший `library/mongo` как единственный ориентир). Лицензия сервера — **SSPL**.  
На Kubernetes: оператор **MCK 1.10.0** (30 июля 2026), CR **`MongoDBCommunity`**, тип официально **только `ReplicaSet`**. Старый Community Operator — EOL, не начинать с него.  
Документация линейки: https://www.mongodb.com/docs/v7.0/ · MCK: https://www.mongodb.com/docs/kubernetes/current/

Это **не** замена Kafka и **не** кэш (кэш — Redis/Valkey). Роль озера за Mongo в ТЗ не закреплена.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Все **голосующие** члены replica set живут **внутри одного ЦОДа** (одного Kubernetes). Между ЦОДами — hidden/`priority: 0` копии **или** только бэкапы/restore. Автоfailover primary в чужой ЦОД **не** цель.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Юристы **принимают SSPL** для закрытого контура. Если нет — этот файл не разрешение ставить Community 7.0.
4. Dev — изолированная сеть; пароль в примере не секрет.
5. Нагрузки нет — нет цифры «N шардов и M гигабайт cache». Шарды на старте **не** ставим: `MongoDBCommunity` их не описывает.
6. Для 2 ЦОДов: PSS-3 voting в ЦОД-1, в ЦОД-2 — hidden или бэкап. Для 3 ЦОДов: то же + вторая hidden/только бэкапы. Третий ЦОД **не** добавляет второго writer.
7. PSA (arbiter) в прод **не** берём: молчаливый дефолт write concern `{ w: 1 }`, проблемы `w: majority`.
8. Native encryption at rest / аудит / LDAP — **Enterprise**, не Community.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** документы, индексы, драйвер микросервиса. **Не** цель: отказ площадки и терабайты.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 27017 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name mongo-dev \
  -p 127.0.0.1:27017:27017 \
  mongodb/mongodb-community-server:7.0.40-ubi9-slim
```

Привязка к `127.0.0.1` обязательна: внутри контейнера процесс слушает шире localhost. `-p 27017:27017` на все интерфейсы рабочей станции **нельзя**.

Проверка:

```bash
docker exec -it mongo-dev mongosh --eval "db.runCommand({ ping: 1 })"
docker exec -it mongo-dev mongosh --eval "db.version()"
```

`ok: 1` и версия **7.0.40**. Затем завести **не**-root пользователя на БД приложения (на Dev без `--auth` это SQL-привычка впрок; на общем кластере включайте SCRAM сразу):

```bash
docker exec -it mongo-dev mongosh --eval '
db.getSiblingDB("app").createUser({
  user: "app",
  pwd: "dev-app",
  roles: [ { role: "readWrite", db: "app" } ]
})
'
```

Клиент — `localhost:27017`, БД `app`, роль `app`. Не `root` из приложения. Лимит документа **16 МБ** поймать до боя.

### Установка (Kubernetes Dev)

MCK 1.10.0, затем `MongoDBCommunity` с `members: 1` только если команда понимает: это **не** HA. Для препрода лучше сразу `members: 3` **в одной зоне** — чтобы драйвер учился `replicaSet=` URI.

```bash
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes/1.10.0/public/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes/1.10.0/public/mongodb-kubernetes.yaml
```

Либо Helm репозиторий `https://mongodb.github.io/helm-charts`, чарт `mongodb/mongodb-kubernetes`, пин **1.10.0** (`--version` / `operator.version` — сверить с каталогом чарта на дату установки, не `latest`).

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Один `mongod` / members: 1 | да | Нет требования пережить выкат |
| Auth | можно без, **только loopback** | Дефолт сервера auth выкл. — не тащить это в прод |
| TLS | нет | Иначе PKI раньше схемы |
| `cacheSizeGB` | 256–512 МБ явно, если K8s | Иначе процесс считает RAM **хоста** |
| Шарды / mongos | нет | Иначе shard key раньше модели |
| Change stream | на чистом standalone **нет** | Для прототипа «по событию документа» на препроде нужен replica set |

Чего **не** упрощать: версия **7.0.40**; имена полей и индексы под реальные запросы; клиент ходит на hostname.

### Проверка Dev

1. `db.version()` = 7.0.40.
2. insert/find ролью `app`.
3. Рестарт с volume: данные живы.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный Community-образ | Нет выборов, нет `w: majority`, нет initial sync |
| Совпадает с development-практикой MongoDB | Успешный `insert` **не** доказывает прод |
| | Открытый 27017 без пароля на Wi-Fi = дыра |
| | Change stream на standalone недоступен |

Препрод: PSS-3 в **одном** ЦОДе, SCRAM, TLS внутри набора, `w: majority` + `wtimeout`, `cacheSizeGB`.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа** (выборы primary); пережить отказ **целого ЦОДа** ценой RPO>0 и **ручного** reconfigure / restore. Цифр TPS нет.

### Почему не stretch

Выборы идут по majority **голосующих** членов. На схеме «по одному voting в ЦОД» каждый `w: majority` платит RTT до чужой площадки; при плавающем RTT — ложные выборы и rollback хвоста `w: 1`. Официальные geo-примеры 1-1-1 и 2-2-1 — это как раз stretch кворума. Пользовательский запрет RTT их снимает. PSA «чтобы сэкономить диск» в другой ЦОД — ещё хуже (см. `MongoDB 7.md`).

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один PSS replica set (вариант B «мозг в одном ЦОДе» из `MongoDB 7.md`):

- `members: 3`, все **voting**, `priority > 0`, на **разных нодах** этого ЦОДа;
- anti-affinity / topology spread по залам **внутри** площадки, если они разные домены отказа;
- клиенты: replica set URI (`replicaSet=...`) или SRV, `retryWrites`; пишут на primary. Балансировщик «как HTTP на все 27017» **ломает** модель;
- `w: "majority"` + **ненулевой** `wtimeout` на факты, которые нельзя rollback;
- PVC локальные, XFS на dbPath настоятельно рекомендуется, **не NFS** как основной диск;
- `storage.wiredTiger.engineConfig.cacheSizeGB` **явно**, меньше memory limit пода;
- образ / `spec.version`: **7.0.40**.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: voting PSS-3. ЦОД-2: член с `priority: 0` (часто **hidden**) — копия через oplog, **без** голоса **или** только снимок/CSI/`mongodump` в object storage | Записи нет, пока restore или ручной reconfigure (вендор описывает reconfigure при недоступных членах — процедура аварии, не автоfailover). Hidden не спасает кворум: голосов в ЦОД-2 нет |
| **3 ЦОДа** | ЦОД-1: voting. ЦОД-2: hidden/`priority: 0`. ЦОД-3: вторая такая копия **или** только бэкапы | То же; третий ЦОД не даёт второго primary |

`MongoDBCommunity` живёт в **одном** Kubernetes: три voting пода — в кластере ЦОД-1. Hidden в чужом K8s — это уже отдельный `mongod` с hostname в replica set (mesh 27017, TLS, keyfile), не «ещё один members: в CR другого ЦОДа». Если mesh между площадками не хотите держать открытым — **только бэкапы**, как PITR у PostgreSQL.

Клиенты ЦОД-2/3 в штате ходят в URI ЦОД-1 по городу (**TLS обязателен**) **или** читают hidden (лаг, вендор **не** рекомендует secondary как основной паттерн чтения; для эталона опасно).

Несколько **разных** Cluster (операционка сервиса ≠ метаданные Grafana) — отдельные HA.

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не stretch-кластер).
- CSI RWO, `WaitForFirstConsumer`; хост: THP выкл., NUMA interleave, `vm.swappiness` 0 или 1 — production notes, не ConfigMap «на глаз».
- NetworkPolicy: 27017 только приложениям и членам набора.
- Секреты/keyfile — Secret/Vault, не Git.
- NTP. Имена — DNS, не голый IP (с 5.0 узел только с IP в конфиге replica set не стартует).

### Установка оператора (на Kubernetes ЦОД-1, где будет replica set)

```bash
kubectl create namespace mongodb
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes/1.10.0/public/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes/1.10.0/public/mongodb-kubernetes.yaml
kubectl -n mongodb rollout status deployment/mongodb-kubernetes-operator
```

Оператор — не в одном поде с `mongod`. Совместимость Helm-тега с 1.10.0 **сверять** на стенде.

### Конфигурирование активного набора (ЦОД-1)

Смысл CR (не полный манифест — сверять спецификацию `MongoDBCommunity` MCK 1.10):

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongo-app
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.40"
  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: true
      # сертификаты — по доке MCK 1.10 (secure.md), не «tls: false как в примере»
  additionalMongodConfig:
    storage.wiredTiger.engineConfig.cacheSizeGB: 1  # заглушка: меньше лимита пода; не формула хоста
  users:
    - name: app
      db: app
      passwordSecretRef:
        name: mongo-app-password
      roles:
        - name: readWrite
          db: app
      scramCredentialsSecretName: app-scram
```

Обязательно рядом:

1. Пользователь приложения **без** `userAdminAnyDatabase` / `root`. Сначала user administrator, потом остальные.
2. Внутренний auth: keyfile или X.509 между членами (оператор обычно закрывает — **проверить**, не поверить).
3. `--noscripting`, если нет явной нужды в server-side JS.
4. PDB: не эвакуировать 2 из 3 voting одним drain.
5. Бэкап: снимок тома/CSI **с** journal, предпочтительно со hidden secondary; `mongodump --oplog` вендор относит к **небольшим** деплоям. Проверяют **restore**.
6. Клиенты: `sslmode`/TLS verify, короткий timeout, reconnect. Не IP пода.

Hidden в ЦОД-2: `priority: 0`, `hidden: true`, `votes: 0`. Это копия для DR/отчётности, не участник выборов. Promote в voting — **ручной** reconfigure.

### Масштабирование (когда появятся цифры)

1. Замерить размер данных/индексов, working set vs cache, IOPS, p99, лаг oplog, connections.
2. Упёрлись CPU/IOPS primary — вертикаль или шарды (**не** через `MongoDBCommunity`). Replica **не** ускоряет запись.
3. Упёрлись RAM cache — сначала индексы/проекции, потом RAM.
4. Тяжёлые отчёты — не primary SoT (склад/ClickHouse).
5. Journaling **не** выключать.

### Проверка прода (пока это не пройдено — это не прод)

1. `db.version()` = 7.0.40 на всех членах ЦОД-1.
2. Запись `w: majority` + `wtimeout`; чтение; запись на secondary — не как основной путь.
3. `rs.stepDown()`: приложение переподключилось, строки на месте.
4. Restore снимка на чистый набор **на стенде**.
5. Учение «ЦОД-1 недоступен»: запись стоит; restore/reconfigure в ЦОД-2 прогнали хотя бы раз (RTO замерить).
6. Authorization включён; приложение без роли не пишет.

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + hidden/бэкап)

| Сильное | Слабое |
|---|---|
| Выборы не зависят от межЦОДового RTT | Падение ЦОД-1 = нет записи, пока restore/reconfigure |
| Согласовано с «не stretch» и с MCK Community (ReplicaSet в одном K8s) | RPO между ЦОДами > 0 (скрытая копия async по oplog; чисто архив — ещё больше) |
| PSS-3 — официальный минимум прода **внутри** площадки | Ручной DR; нет native at-rest и аудита в Community |
| | Клиентский 27017 через город — TLS и сеть ваши |

**Не готов к проду**, если: `members: 1` выдают за HA; PSA; voting размазаны по ЦОДам; `authorization` выкл.; нет `cacheSizeGB` в контейнере; клиенты без replica set URI; бэкап = «есть secondary»; шарды обещаны через `MongoDBCommunity`; образ не 7.0.40 / `latest`; SSPL не закрыт юристами; чтение эталона с secondary «для скорости» без принятого лага; Dev-CR скопирован в бой.

---

## Источники

- Релиз 7.0.40: https://www.mongodb.com/docs/v7.0/release-notes/7.0/
- Образ: https://hub.docker.com/r/mongodb/mongodb-community-server
- MCK 1.10.0 манифесты: https://github.com/mongodb/mongodb-kubernetes/tree/1.10.0/public
- Установка оператора: https://www.mongodb.com/docs/kubernetes/current/tutorial/install-k8s-operator/
- `MongoDBCommunity` только ReplicaSet: https://www.mongodb.com/docs/kubernetes/current/reference/k8s-operator-community-specification/
- Replica set / geo / PSA / WiredTiger / security: см. список в `MongoDB 7.md`

Порога RTT для stretch в документации MongoDB **нет** — поэтому stretch (в том числе multi-DC PSA) в этой инструкции не предлагается.
