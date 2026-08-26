# ClickHouse 26.7.5.10 — установка и конфигурирование

Связанный документ (глоссарий, Keeper, ReplicatedMergeTree, почему так): `ClickHouse.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного кластера (Raft Keeper 9234, interserver 9009) на несколько ЦОДов **не делаем**: heartbeat Keeper дефолт 500 мс; порога RTT у проекта нет.

Версии: **ClickHouse 26.7.5.10-stable**, образы `clickhouse/clickhouse-server:26.7.5.10` и `clickhouse/clickhouse-keeper:26.7.5.10`. На Kubernetes — **Altinity Operator 0.27.3** (`ClickHouseInstallation` / `ClickHouseKeeperInstallation`).  
Документация: https://clickhouse.com/docs/

Альтернатива политикой обновлений: весь кластер на LTS **26.3.21.7-lts**. **Не смешивать** 26.7 и 26.3 в одном кластере. Это OSS self-hosted, не ClickHouse Cloud/Private (SharedMergeTree).

---

## Допущения этой инструкции

1. **Stretch запрещён.** Кворум Keeper и реплики ReplicatedMergeTree **одного** кластера — **внутри одного ЦОДа**. Между ЦОДами — отдельный кластер **или** `BACKUP`/`RESTORE` (вариант B в `ClickHouse.md`: изолированный Kubernetes, **нет** 9009/9234 между ЦОДами при плохом RTT). Async replicated database — только если осознанно строите её как отдельный контур, не как один `remote_servers` на три площадки.
2. Прод — Kubernetes в каждом ЦОДе отдельно (`Kubernetes.install.md`).
3. Dev — изолированная сеть; combined Keeper допустим только там.
4. Нагрузки нет — нет числа шардов «хватит». Сначала вертикаль, шарды когда одна машина мала.
5. Object storage для `BACKUP` в проде **есть или будет**. Реплики не заменяют бэкап (`DROP` реплика повторит).
6. Для 2 ЦОДов: кластер в ЦОД-1; ЦОД-2 — DR restore **или** свой кластер (нет единого SELECT без прикладного слоя). Для 3 ЦОДов: то же. Третий ЦОД **не** третий Keeper чужого кворума.
7. Питание из Kafka в проде — Connect Sink с `exactlyOnce` (отдельное ПО); Kafka engine на стенде — at-least-once.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** SQL, `ORDER BY`, сжатие ваших событий. **Не** цель: отказ ЦОДа и quorum-INSERT.

### Предпосылки

- Docker Engine **или** однонодовый Kubernetes.
- Порты 8123 и 9000 свободны на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name ch-dev \
  --ulimit nofile=262144:262144 \
  -p 127.0.0.1:8123:8123 \
  -p 127.0.0.1:9000:9000 \
  -e CLICKHOUSE_PASSWORD=dev \
  -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 \
  clickhouse/clickhouse-server:26.7.5.10
```

Привязка к `127.0.0.1` обязательна. Без пароля в Docker `default` **не слушает сеть** — это не «сломанный образ». `CLICKHOUSE_SKIP_USER_SETUP=1` на стенд **не** включать: привычка уедет в прод.

Проверка:

```bash
curl -s "http://127.0.0.1:8123/?user=default&password=dev" --data-binary "SELECT version()"
```

В строке должна быть **26.7.5.10**. Затем **не**-`default` для приложения:

```sql
CREATE USER app IDENTIFIED BY 'dev-app';
CREATE DATABASE analytics;
GRANT SELECT, INSERT ON analytics.* TO app;
```

На Kubernetes Dev: один `ClickHouseInstallation` с 1 репликой, маленький PVC, без отдельного CHK. **Не** этот YAML в прод.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Число серверов | 1 | Нет требования пережить выкат |
| Keeper | нет (обычный MergeTree) | Иначе Raft раньше `ORDER BY` |
| TLS | нет, сеть закрыта | Иначе PKI раньше SQL |
| Kafka engine | можно | На стенде проще Connect |
| BACKUP | можно отложить на день | На препроде бакет обязателен |
| `::/0` в users | запрещён | Даже на Dev за firewall стенда |

Чего **не** упрощать: версия **26.7.5.10** (или сразу весь контур LTS); явный `ORDER BY` под ваши запросы; батчи INSERT, не строка из цикла; понимание — голый MergeTree **не** реплицируется.

### Проверка Dev

1. `SELECT version()` = 26.7.5.10.
2. Приложение ходит ролью `app`, не `default` без пароля в сеть.
3. Рестарт с volume: части на диске живы.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный Docker-гайд | Нет Keeper, нет 9009, нет `insert_quorum` |
| Single-node официален для development | Успешный SELECT **не** доказывает прод |
| | HTTP без TLS приучают «зайти как получится» |

Препрод: 3 Keeper + 1 шард × 3 реплики, TLS, `insert_quorum=2`, BACKUP — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри ЦОДа** (3 Keeper, 3 реплики шарда, `insert_quorum=2`). Отказ **целого ЦОДа** = нет этого кластера, пока `RESTORE` на другой площадке или пока не переключите клиентов на второй независимый кластер. RPO при архивном DR > 0.

### Почему не stretch

Выборы лидера Keeper и копирование parts по 9009 не быстрее сети. Если p99 RTT сопоставим с heartbeat 500 мс — ложные выборы и ReplicatedMergeTree в **read-only**. Документация **не задаёт** порог. «Keeper в трёх ЦОДах» при потере двух площадок всё равно снимает кворум.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один кластер:

- 3 dedicated Keeper (CHK), отдельные PVC, не на тех же дисках, что тяжёлый merge;
- старт: **1 шард × 3 реплики** на разных нодах **этого** ЦОДа (минимум размещения, не терабайты);
- только **ReplicatedMergeTree** (+ `ON CLUSTER`) для того, что должно пережить ноду;
- `remote_servers`: `internal_replication=true`;
- профиль ingest: `insert_quorum=2` или `'auto'` (при 3 репликах большинство = 2). `insert_quorum=3` остановит запись при падении одной реплики;
- клиенты: Service на 8443/9440, **не** на Keeper 9181/9234;
- образ **26.7.5.10** на server **и** Keeper;
- диск: local SSD/NVMe, **не NFS**, не `emptyDir` для `/var/lib/clickhouse`.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: кластер. ЦОД-2: нет 9009/9234 к ЦОД-1. Клиенты ходят SQL в ЦОД-1 **или** свой кластер. DR: `RESTORE` из бакета | Нет аналитики этого кластера, пока restore/второй кластер. RTO замерить |
| **3 ЦОДа** | ЦОД-3 аналогично ЦОД-2 | То же; три независимых кластера ≠ один SELECT «по стране» без прикладного слоя |

Не класть «все Keeper в ЦОД-1, реплики в трёх ЦОДах»: падение ЦОД-1 = данные без координации (read-only). Это не переживание площадки.

### Предпосылки прода

- Kubernetes ЦОД-1; CSI RWO с предсказуемым IOPS.
- Бакет `BACKUP`, доступный после гибели ЦОД-1 (иначе бэкап умер вместе с кластером).
- NetworkPolicy: клиенты → 8443/9440; 9009/9010 только серверы; 9181/9234 — server↔Keeper.
- SQL-driven пользователи (XML-учётки не входят в `BACKUP`).
- Оператор 0.27.3 прогнан с **26.7.5.10** на стенде.

### Установка (оператор, ЦОД-1)

1. Поставить Altinity Operator **0.27.3** (манифест/Helm тега релиза, не `latest`).
2. `ClickHouseKeeperInstallation` × 3, образ keeper **26.7.5.10**.
3. `ClickHouseInstallation`: ссылка на CHK по имени (`zookeeper.keeper.name`), version **26.7.5.10**, topology spread по нодам **этого** ЦОДа.
4. TLS: клиент 8443/9440, interserver 9010, Keeper 9281 + Raft secure — **до** боевых таблиц.
5. Создать таблицы `ENGINE = ReplicatedMergeTree` `ON CLUSTER`. Проверить `system.replicas`.
6. Прогнать `BACKUP` + `RESTORE` на другой namespace. Секреты S3 — named collection (в 26.7 пользовательский SQL по умолчанию **не** берёт server-side cloud credentials).

### Конфигурирование (смысл)

| Параметр | Прод | Зачем |
|---|---|---|
| `default` | пароль или localhost-only; нет SKIP_USER_SETUP | Туториал без пароля *discouraged* |
| `insert_quorum` | 2 (3 реплики) | Дефолт — подтверждение **одной** реплики; блок может пропасть |
| `internal_replication` | true | Иначе Distributed пишет во все реплики сам |
| Эмуляции 9004/9005/9100 | выкл, если не используете | Лишняя поверхность |
| TTL / ORDER BY | с первого дня | ORDER BY дешево не поменять |
| `secret` в `remote_servers` | задать | Иначе чужой 9000 прикинется шардом |

PDB: не снять сразу два Keeper или две реплики шарда. Выкат **по одной** ноде: две сразу при quorum=2 могут остановить запись.

### Масштабирование (когда появятся цифры)

1. Вертикаль реплик (CPU/RAM/диск). Вендор: не шардировать рано.
2. Шарды — когда одна машина мала; решардинг — отдельный проект.
3. Батчи INSERT; async insert с 26.3 дефолтен, но миллионы одиночных строк всё равно душат parts/Keeper.
4. RAM:ориентиры вендора 1:30–1:130 диск — не смета вашего прода.

### Проверка прода (пока это не пройдено — это не прод)

1. `SELECT version()` = 26.7.5.10 на всех серверах и Keeper ЦОД-1. Нет смешанных 26.3/26.7.
2. `mntr` / кворум Keeper. Убить одну реплику: SELECT жив, INSERT с quorum=2 проходит.
3. Restore на чистый CHI на стенде.
4. Учение «ЦОД-1 выключен»: restore в ЦОД-2 **по времени** или клиенты на второй кластер. 9009/9234 между ЦОДами закрыты — и это **ожидаемо**.

### Сильные / слабые стороны прод-схемы (кластер в одном ЦОДе + restore)

| Сильное | Слабое |
|---|---|
| Raft и 9009 локальные | Падение ЦОД-1 = нет аналитики, пока DR |
| Согласовано с запретом stretch | RPO > 0 при restore; второй кластер — вторая правда таблиц |
| Один SQL-кластер для разработчиков | Клиентский SQL через город, если приложения в ЦОД-2 |
| | Connect exactly-once — отдельный HA, не этот файл |

**Не готов к проду**, если: один MergeTree без Keeper; `insert_quorum` выкл на фактах; `internal_replication=false` на Replicated; SKIP_USER_SETUP; plaintext 8123/9009 «на три ЦОДа»; данные на NFS/`emptyDir`; смешение 26.7 и 26.3; ClickHouse назначен SoT карточек; Dev-манифест в бою.

---

## Источники

- Релиз 26.7.5.10: https://github.com/ClickHouse/ClickHouse/releases/tag/v26.7.5.10-stable
- Keeper: https://clickhouse.com/docs/guides/oss/deployment-and-scaling/keeper
- Replication, `insert_quorum`: https://clickhouse.com/docs/architecture/replication
- Docker, `default`, ulimit: https://clickhouse.com/docs/get-started/setup/self-managed/docker
- Altinity Operator 0.27.3: https://github.com/Altinity/clickhouse-operator/releases
- Правила: `ClickHouse.md`
