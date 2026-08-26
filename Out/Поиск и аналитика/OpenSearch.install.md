# OpenSearch 3.8.0 — установка и конфигурирование

Связанный документ (глоссарий, кворум manager, CCR, почему так): `OpenSearch.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного кластера (transport **9300**) на несколько ЦОДов **не делаем**: cluster state и репликация шардов ходят по 9300; порога RTT у проекта нет.

Версии: **OpenSearch 3.8.0**, образ `opensearchproject/opensearch:3.8.0`. На Kubernetes целевой путь — **OpenSearch Kubernetes Operator** (репозиторий `https://opensearch-project.github.io/opensearch-k8s-operator/`). Helm-чарты проекта — запасной ручной путь. Dashboards — тот же тег, см. `OpenSearch Dashboards.install.md`.  
Документация: https://docs.opensearch.org/latest/

Это **не** Amazon OpenSearch Service и **не** тот же кластер, что Wazuh indexer.

---

## Допущения этой инструкции

1. **Stretch запрещён.** Кворум cluster-manager и data-ноды **одного** кластера — **внутри одного ЦОДа**. Между ЦОДами — отдельный кластер + **CCR** (follower read-only, если берёте плагин с обоих сторон) **или** поиск только в ЦОД-1 (клиенты ходят на 9200 туда).
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`).
3. Dev — изолированная сеть; demo-сертификаты допустимы только там.
4. Нагрузки нет — нет числа data-нод «хватит». Есть сигналы (heap, rejected, watermark) и рычаги (ноды, шарды, ISM).
5. Snapshot-репозиторий в проде **есть или будет**. Replica не бэкап (`DELETE index` реплика не спасёт).
6. Для 2 ЦОДов: кластер в ЦОД-1; ЦОД-2 — CCR follower **или** только клиенты 9200 / restore snapshot. Для 3 ЦОДов: то же. Третий ЦОД **не** третий manager чужого кворума.
7. Java в бандле линии 3.6.1+: таблица вендора 21/25/26; свой JDK — через `OPENSEARCH_JAVA_HOME`. На хосте `vm.max_map_count ≥ 262144`.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** mapping, bulk, Dashboards, пайплайн из Kafka. **Не** цель: отказ ЦОДа.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порт 9200 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name os-dev \
  -p 127.0.0.1:9200:9200 \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_INITIAL_ADMIN_PASSWORD=DevAdmin_12Str0ng" \
  opensearchproject/opensearch:3.8.0
```

Привязка к `127.0.0.1` обязательна. С 2.12 без пароля admin demo-конфиг **не стартует**. `discovery.type=single-node` **обходит** нормальный discovery — только dev/test.

Проверка:

```bash
curl -sk -u admin:DevAdmin_12Str0ng https://127.0.0.1:9200
```

В JSON — `number` / version **3.8.0**. Replica на single-node = **0**, иначе вечный yellow.

`DISABLE_SECURITY_PLUGIN=true` не включать «чтобы curl без пароля»: привычка уедет в прод.

На Kubernetes Dev: один `OpenSearchCluster` с 1 replica, маленький PVC; либо Helm `singleNode: true`. **Не** этот YAML в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Нод | 1 | Нет требования пережить выкат |
| Replica | 0 | Копию некуда положить |
| PKI | demo / self-signed | Иначе cert раньше mapping |
| Snapshot | можно отложить на день | На препроде репозиторий обязателен |
| ISM | минимум | Сначала увидеть объём тестовых логов |
| Индекс | явный `_id` из Kafka | Идемпотентность ingest |

Чего **не** упрощать: тег **3.8.0** = Dashboards; primary shards задаются при создании; compose вендора **не для production**.

### Проверка Dev

1. `curl` к 9200, version 3.8.0.
2. Документ записался и ищется. Рестарт без PVC — индекс пуст (если не было диска).
3. 9200 не с мира.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный Docker-гайд | Нет кворума, нет 9300, нет awareness |
| Single-node официален для test | Успешный поиск **не** доказывает прод |
| | Demo PKI приучает `admin` |

Препрод: 3 dedicated manager + data, свои cert, replica ≥ 1, snapshot — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **ноды внутри ЦОДа** (3 manager, replica ≥ 1). Отказ **целого ЦОДа** = нет этого кластера для записи; чтение возможно с CCR-follower (устаревшее, RPO > 0) или после restore snapshot.

### Почему не stretch

Transport 9300 — не HTTP-LB «на все ноды». Высокий/плавающий RTT → медленные выборы, delayed unassigned, yellow. Официальный паттерн «3 manager в 3 зонах» в доке — про зоны **доступной** сети; мы не принимаем городской WAN за такую сеть без замера, а замер порога проект **не даёт**. Forced awareness на три ЦОДа при stretch ещё и оставляет yellow, пока мёртвая зона не вернётся.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один кластер:

- 3 dedicated `cluster_manager` (`node.roles: [ cluster_manager ]`), не принимают клиентский трафик;
- data (`data, ingest` на старте) **внутри ЦОДа**, anti-affinity по ноде;
- `cluster.initial_cluster_manager_nodes` — **один раз** на bootstrap, одинаковый список; новые manager присоединяются, не бутстрапят второй UUID;
- replica ≥ 1; диск local SSD, **не NFS**, не `emptyDir`;
- клиенты на Service **9200** (coordinating/ingest);
- HTTP TLS вкл (`plugins.security.ssl.http.enabled: true` — дефолт plugin **false**);
- transport TLS обязателен при Security plugin;
- `DISABLE_INSTALL_DEMO_CONFIG=true`, своих demo-cert нет;
- куча ~½ RAM, `Xms = Xmx`; лимит пода **выше** кучи (Lucene mmap);
- образ **3.8.0**.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: leader-кластер (запись). ЦОД-2: **CCR follower** (read-only) **или** клиенты ходят на 9200 ЦОД-1 + snapshot DR | Нет записи в leader. Follower не становится writer сам — отдельная процедура. Restore — замерить RTO |
| **3 ЦОДа** | ЦОД-3: ещё follower / только клиенты / только snapshot | То же; три независимых «правды индекса» без CCR хуже |

CCR: плагин на **обоих** кластерах; Security включён на обоих или выключен на обоих. Wildcard replication сопровождать. Это **не** прозрачный failover записи.

Wazuh indexer — **отдельный** кластер, другой Security realm.

### Предпосылки прода

- Kubernetes ЦОД-1; CSI RWO; sysctl `vm.max_map_count` на хосте (оператор часто ставит init-контейнером — в restricted-кластере решить явно).
- Snapshot repository (S3-совместимый), доступный после гибели ЦОД-1. В **3.8.0** дефолт SSE `AES256` — если шлюз отвергает заголовок, явно `bucket_default` (breaking 3.8.0).
- PKI свои. `enforce_hostname_verification: true`.
- NetworkPolicy: 9200 — клиенты и Dashboards; 9300 — только ноды **этого** кластера (+ CCR, если B2); 9600 не в интернет.

### Установка (оператор, ЦОД-1)

1. Поставить оператор (Helm тега релиза линейки 3.x; таблица проекта: operator **3.0.0+** → OpenSearch до latest 3.x). Перед боем — прогон **именно 3.8.0** на стенде.
2. CR: isolated pool `cluster_manager` × 3 + `data`; `version: 3.8.0`.
3. Security config: свои cert, свой admin, не demo. `securityadmin.sh` только с admin DN по HTTPS 9200.
4. Зарегистрировать snapshot repository, прогнать create **и restore**.
5. Index templates (shards, replica, mapping) и **ISM до** боевого потока. Primary потом дешево не поменять.
6. Dashboards — `OpenSearch Dashboards.install.md`, тот же ЦОД, тот же кластер.

### Конфигурирование (смысл)

| Параметр | Прод | Зачем |
|---|---|---|
| Demo certs | запрещены | *well known and therefore unsafe* |
| Dual-mode SSL | только миграция | Не постоянный прод |
| Клиенты на manager | нет | Вендор: трафик на ingest/coordinating/data |
| ISM | горячий → delete/snapshot | Иначе диск съедят «терабайты» сами |
| `admin` у приложений | нет | Отдельные internal users / JWT / mTLS |

PDB: не снять сразу два manager. Rolling upgrade оператора ждёт **green** после каждой ноды.

### Масштабирование (когда появятся цифры)

1. Объём/поиск → data-ноды + размер шарда (бенчмарк, не слайд).
2. Индексация → bulk, не одиночные POST; при необходимости отдельный ingest pool.
3. Тяжёлые aggregations → coordinating, чтобы не душить heap data.
4. Manager почти не масштабируют «от QPS»; им RAM под cluster state.

### Проверка прода (пока это не пройдено — это не прод)

1. Version 3.8.0 на всех нодах. `_cluster/health` green (или yellow осознанный).
2. Убить data-под: документ читается (replica на другой ноде).
3. Snapshot restore на другой namespace.
4. Без TLS/без роли — отказ. 9300 с мира закрыт.
5. Учение «ЦОД-1 выключен»: follower только читает **или** честный простой поиска; промоушен CCR — по runbook, не «сам». Кластер не общий с Wazuh.

### Сильные / слабые стороны прод-схемы (кластер в одном ЦОДе + CCR/DR)

| Сильное | Слабое |
|---|---|
| 9300 не ходит между ЦОДами | Падение ЦОД-1 = нет записи (и поиска, если нет follower) |
| Проще кворум и диск | CCR: RPO > 0, follower read-only |
| Согласовано с запретом stretch | Клиентский 9200 через город — ваша сеть |
| | Два Security realm при CCR |

**Не готов к проду**, если: `single-node` в бою; replica 0 на нескольких нодах; demo-cert; `DISABLE_SECURITY_PLUGIN`; NFS/`emptyDir`; один кластер 9300 на 2–3 ЦОДа; общий с Wazuh indexer; OpenSearch назначен SoT; нет snapshot.

---

## Источники

- Релиз 3.8.0: https://docs.opensearch.org/latest/version-history/
- Docker, single-node, admin password: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/
- Роли, 3 manager, awareness: https://docs.opensearch.org/latest/tuning-your-cluster/
- CCR: https://docs.opensearch.org/latest/tuning-your-cluster/replication-plugin/
- Operator: https://docs.opensearch.org/latest/install-and-configure/install-opensearch/operator/
- Правила: `OpenSearch.md`
