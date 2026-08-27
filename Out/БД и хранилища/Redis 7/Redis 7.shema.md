# Redis Community Edition 7.4.11 — схемы устройства

Связанные документы: правила — `Redis 7.md`; установка — `Redis 7.install.md`.

Ниже **C4** (четыре уровня: контекст → контейнеры → компоненты → код). В запросе это «D4»: тот же каркас. Уровень кода пропускаем — для эксплуатации важнее роли, порты и ручки HA/масштаба.

Допущения (их не было в исходном ТЗ платформы, без них схема врёт):

1. Stretch одного Sentinel-набора или Redis Cluster на 2–3 ЦОДа **нет** (RTT не позволяет стабильный gossip и кворум). HA — **внутри ЦОД-1**.
2. Community **7.4.11**, лицензия RSALv2 или SSPL (это уже не BSD как линия 7.2). Официального OSS-оператора у Redis Ltd **нет**.
3. Redis **не** источник истины (SoT). Репликация **асинхронная** (`replicaof`): клиент получает OK до догона replica.
4. Нагрузки нет — на схемах нет «N ядер». Есть *что крутить*, когда цифры появятся.

---

## 1. Контекст (C4 system context)

Где Redis в вашей картине. Он **не** шина, **не** озеро и **не** Camunda.

```mermaid
flowchart LR
  subgraph people["Люди и сервисы"]
    MS["Микросервисы / интеграционное API"]
    CW["Camunda job workers"]
  end

  RD["Redis 7.4.11\nкэш, сессии, локи, не SoT"]
  PG["PostgreSQL / озеро\nистина карточки"]
  KF["Apache Kafka\nшина событий"]
  K8["Kubernetes\nодин кластер = один ЦОД"]

  MS -->|"SET GET TTL"| RD
  CW -->|"идемпотентность, короткий лок"| RD
  RD -.->|"не замена"| PG
  KF -.->|"не Pub/Sub Redis"| RD
  K8 --> RD
```

| Стрелка | Зачем помнить |
|---|---|
| Сервис → Redis | Кэш проекции, rate-limit, ключ идемпотентности с TTL. Не карточка клиента навсегда |
| Redis не озеро | Eviction, `FLUSHALL` или failover с async-хвостом истину не берегут |
| Kafka отдельно | Streams и Pub/Sub Redis не заменяют топик с consumer group |

**Слабое место контекста:** «положим клиентов только в Redis» — RAM × реплики. Эталон остаётся в озере/БД.

---

## 2. Контейнеры (из чего состоит решение)

Один логический набор = **один master** (пишет) + replica (копии) + Sentinel (следят и переключают). Sentinel и Cluster **вместе не ставят**. Cluster, размазанный по ЦОДам, **запрещён**.

```mermaid
flowchart TB
  subgraph k8s["Один Kubernetes = ЦОД-1"]
    subgraph data["Данные порт 6379"]
      M["Pod master\nзапись"]
      R1["Pod replica"]
      R2["Pod replica"]
    end
    subgraph sent["Sentinel порт 26379"]
      S1["Sentinel"]
      S2["Sentinel"]
      S3["Sentinel"]
    end
    PVC["PVC RWO\nRDB снимок и AOF журнал"]
  end

  APP["Приложение умеет Sentinel"]
  EXP["redis-exporter\nдругое ПО рядом"]

  APP -->|"кто сейчас master"| S1
  APP -->|"6379 TLS"| M
  M -->|"async replicaof"| R1
  M -->|"async replicaof"| R2
  M --> PVC
  S1 <--> S2
  S2 <--> S3
  EXP --> M
```

Порты: **6379/TCP** — клиенты и репликация; **26379/TCP** — Sentinel; **16379/TCP** — Cluster bus (командный порт + 10000), только если Cluster **целиком** в ЦОД-1. Балансировщик HTTP «на все 6379» без Sentinel/Cluster-aware клиента **ломает** модель: запись попадёт на replica.

**Сильное:** 3 Sentinel в разных залах ЦОД-1 переживают падение пода master. **Слабое:** master без persistence (диск выключен) + авторестарт пустым **стирает** replica — официальный failure mode. Совместимость стороннего оператора с 7.4.11 проверять на стенде.

---

## 3. Компоненты внутри инстанса

```mermaid
flowchart TB
  subgraph inst["Один процесс redis-server"]
    CMD["Один поток команд"]
    IO["io-threads\nсеть, не исполнение"]
    MEM["maxmemory\nрабочий набор в RAM"]
    ACL["ACL пользователи"]
    PER["RDB / AOF"]
  end

  CLI["Клиент"]
  DISK["SSD локально, не NFS"]

  CLI -->|"6379 TLS"| ACL
  ACL --> CMD
  CMD --> MEM
  CMD --> PER
  PER --> DISK
  IO --> CMD
```

| Компонент | Для чего настраивать |
|---|---|
| `maxmemory` + policy | В дефолтном `redis.conf` потолка **нет**, политика `noeviction`. Кэш: `allkeys-lru`. Локи/idem, которые нельзя выкинуть — **другой** инстанс |
| AOF `everysec` + RDB | Дефолт AOF выключен. `everysec`: около 1 с записей можно потерять при аварии (формулировка доки) |
| ACL | Не один `requirepass` на всех. Приложению запретить `FLUSHALL`, `CONFIG`, `EVAL` |
| `replica-read-only` | Дефолт yes. Случайная запись в replica |

Команды в основном однопоточны. В Cluster работает только логическая DB 0: «db=1 кэш, db=2 сессии» туда не переедет.

---

## 4. Поток записи

```mermaid
sequenceDiagram
  participant App as Приложение
  participant Sen as Sentinel
  participant M as Master
  participant R as Replica
  participant Disk as AOF или RDB

  App->>Sen: кто сейчас master
  Sen-->>App: адрес 6379
  App->>M: SET
  M->>M: записать в RAM
  M-->>App: OK
  Note over M,R: replicaof async: OK раньше, чем replica догнала
  M->>R: поток репликации 6379
  M->>Disk: fsync AOF later
```

`WAIT` / `WAITAOF` (с 7.2) сужают окно потери, **не** делают Redis системой с сильной согласованностью. Active-Active (мультимастер в регионах) — Redis Software, не Community 7.4.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    I3["1 master + 2 replica\nразные ноды"]
    SQ["3 Sentinel, кворум 2"]
    PS["AOF и RDB на master и replica"]
    AFF["anti-affinity / PDB"]
  end

  subgraph dr["Между ЦОДами — не stretch"]
    AR["replicaof async\nвне кворума Sentinel"]
    RB["копия RDB + проверка restore"]
  end

  inside -->|"падение пода"| FA["Sentinel promote"]
  inside -->|"падение ЦОД-1"| dr
```

| Ручка | Если не настроить |
|---|---|
| 3 Sentinel, не 2 | Два официально *DON'T DO THIS*. Нет majority — failover не стартует, даже если master мёртв |
| Persistence на master **и** replica | Пустой рестарт master съедает копии |
| Клиент умеет Sentinel | Redis переключил master, приложение пишет в старый IP |
| `min-replicas-to-write` | Дефолт 0: пишем в одиночку. ≥1 — запись встанет без живой replica |
| Cluster на 3 ЦОДа | Stretch **запрещён**: majority master и `cluster-node-timeout` живут межплощадочным RTT |

Падение **двух** ЦОДов при мозге в одном — нет записи, пока restore. Это цена запрета stretch.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись"]
  Q --> RAM["RAM / maxmemory"]
  Q --> W["Запись, один поток"]
  Q --> Rd["Чтение"]
  Q --> HK["Горячий ключ"]

  RAM --> RAM1["TTL и eviction\nне класть эталон"]
  RAM --> RAM2["Вертикаль или Cluster внутри ЦОД-1"]
  W --> W1["Replica запись не ускоряет"]
  Rd --> R1["Чтение с replica, есть лаг"]
  HK --> K1["Слот Cluster один VIP не спасёт"]
```

Горизонталь записи — только шарды Cluster (16384 слота) **внутри ЦОД-1**, когда один master доказанно мал. Новые ноды пустые, пока не reshard. `io-threads` — сеть, не «многоядерный Redis как СУБД». Терабайты озера сюда не кладут.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1 актив"]
    CL["Sentinel x3 + master + replica\nкворум внутри площадки"]
  end
  subgraph dc2["ЦОД-2"]
    DR1["replicaof async\nили холодный RDB"]
  end
  subgraph dc3["ЦОД-3 если есть"]
    DR2["Вторая копия бэкапа\nне третий master кворума"]
  end

  CL -->|"async 6379 или файл RDB"| DR1
  CL -->|"архив RDB"| DR2
```

Клиенты в штате спрашивают Sentinel ЦОД-1. Replica в ЦОД-2 **не** голосует в Sentinel: иначе сеть моргнула — ложный failover и разъезд датасетов. Раскладку Cluster «по master на ЦОД» **не рисуем**. Несколько независимых Redis (кэш прогревается с озера) — вариант C из `Redis 7.md`, не общий кворум.

**Сильное:** детект Sentinel не прибит межЦОДовым RTT. **Слабое:** смерть ЦОД-1 = простой Redis до ручного DR; RPO между площадками больше 0.

---

## 8. Безопасность и сильное / слабое

Слои на 6379 и 26379:

1. NetworkPolicy — кто видит порт. Не LoadBalancer в интернет.
2. ACL + TLS — кто ты. AUTH без TLS сниффер читает пароль как обычную команду.
3. Категории команд — что можно. Боевым ролям лучше запретить Lua/`EVAL` (линия 7.4 закрывает RCE в скриптах, в том числе 7.4.11).

At-rest — шифрование тома CSI, не «галочка encrypt dataset». Protected mode — страховка дефолта, не модель прода.

| Сильное | Слабое |
|---|---|
| Микролатентность in-memory, простые структуры | Не SoT; async потеря хвоста при failover |
| Sentinel HA без шардов внутри ЦОДа | Один writer; потолок = RAM одной машины |
| ACL из коробки 7.4 | RSALv2/SSPL; нет официального OSS-оператора |
| | Stretch Cluster запрещён — нет «одного Redis на страну» |

Источники фактов: `Redis 7.md` (роли, порты, Sentinel vs Cluster, persistence, лицензия). Порога RTT у проекта Redis **нет** — stretch на схемах не рисуем как целевой.
