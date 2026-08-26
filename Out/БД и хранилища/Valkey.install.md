# Valkey 9.1.1 — установка и конфигурирование

Связанный документ (глоссарий, HA, безопасность, почему так): `Valkey.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Sentinel-набора или Cluster на несколько ЦОДов **не делаем**: межЦОДовый RTT для `cluster-node-timeout` / `down-after-milliseconds` неприемлем; порога миллисекунд у Valkey **нет**.

Версия: **Valkey 9.1.1** (не 9.1.0: SECURITY, CVE-2026-56684 / CVE-2026-63639). Образы: `valkey/valkey:9.1.1` (также `-alpine`, `-trixie`). Лицензия **BSD-3**.  
На Kubernetes: официальный Helm **`valkey-io/valkey-helm`**, чарт `valkey` **0.11.0**, `appVersion` **9.1.1** — standalone **или** primary+replicas. **Не** Cluster. **Не** Sentinel.  
Оператор `valkey-io/valkey-operator` (релизы до v0.5.0): README — *not ready for production use*. В прод **не берём**.

Документация: https://valkey.io/ · Helm: https://valkey.io/valkey-helm/

Valkey — слой **ускорения и координации** (кэш, сессии, rate-limit, локи), не SoT и не замена Kafka/PostgreSQL.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Primary, replica и (если ставите) Sentinel-кворум живут **внутри одного ЦОДа**. Между ЦОДами — независимые Valkey **или** клиенты ходят в «домашний» ЦОД. Stretch узлов по трём Kubernetes — та же цена RTT.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Упаковка не-Cluster: Helm **`valkey` 0.11.0**. Sentinel в чарт **не входит** — отдельные манифесты по `sentinel.conf` **или** сознательно принимаете ручной failover primary.
4. Dev — изолированная сеть; пароль в примере не секрет.
5. Нагрузки нет — нет цифры `maxmemory` «хватит для прода».
6. Для 2 ЦОДов: активный набор в ЦОД-1, в ЦОД-2 — независимый Valkey или только клиенты в ЦОД-1. Для 3 ЦОДов: то же + третья площадка. Третий ЦОД **не** добавляет второго writer на те же ключи.
7. Клиенты умеют Sentinel, **если** вы его ставите. Service `valkey` чарта — это writer; после failover без Sentinel DNS primary сам не «переедет».
8. Persistence в проде **включена**. На K8s «кэш без диска» конфликтует с безопасностью репликации: kubelet рестартит пустой primary, replica синхронизируются с пустым (вендор это запрещает).

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** SET/GET, ACL, шум приложения. **Не** цель: отказ площадки и шарды.

### Предпосылки

- Docker Engine **или** Dev-Kubernetes с Helm 3.
- Порт 6379 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

```bash
docker run -d --name valkey-dev \
  -p 127.0.0.1:6379:6379 \
  valkey/valkey:9.1.1
```

Привязка к `127.0.0.1` обязательна. В контейнере процесс обычно слушает `0.0.0.0` — без bind на loopback порт легко оказывается в LAN.

Проверка:

```bash
docker exec -it valkey-dev valkey-cli PING
docker exec -it valkey-dev valkey-cli INFO server
```

`PONG`; в `INFO` должна быть линия **9.1.1** (Valkey отвечает совместимым полем версии). Затем хотя бы один SET/GET из **вашего** сервиса, не только cli.

### Установка (Helm Dev, если стенд уже в K8s)

```bash
helm repo add valkey https://valkey.io/valkey-helm/
helm repo update
helm install valkey-dev valkey/valkey --version 0.11.0 \
  --set image.tag=9.1.1
```

Standalone, replica выкл., persistence по умолчанию чарта **выкл.** — рестарт пода = пустой кэш. Если сеть шире ноутбука: `auth.enabled=true` и пользователь `default` в ACL (иначе, цитата чарта, *anyone can access the database without credentials*).

Режим «Cluster на Minikube через valkey-operator» для знакомства **не нужен**: оператор не для прода, получите ложное чувство HA.

### Конфигурирование Dev

| Параметр | На тесте | Почему можно |
|---|---|---|
| Sentinel / Cluster | нет | Нет требования пережить primary |
| Replica | 0 | Нет нагрузки на чтение |
| PVC | emptyDir / persistence off | Данные стенда не жалко |
| TLS | нет | Иначе PKI раньше продукта |
| `maxmemory` | можно не ставить | Датасет игрушечный |
| `min-replicas-to-write` | 0 | Нет replica |
| `auth.enabled=false` | только loopback | Привычка без ACL уедет в прод — на общем Dev-кластере включайте |

Чего **не** упрощать: тег **9.1.1**; понимание, что Service `valkey` — writer, `valkey-read` после появления replica на запись не ходить.

Не включать replication **без** PVC «на поиграть»: Helm и вендор описывают уничтожение данных.

### Проверка Dev

1. PING / SET / GET с клиента приложения.
2. Рестарт контейнера без volume: ключи пропали — ожидаемо.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный образ и чарт 0.11.0 | Нет модели отказа primary |
| Совпадает с путём проекта (Helm standalone) | Успех GET ≠ Sentinel-клиент и ≠ `WAIT` |
| | Успешный Docker **не** есть готовность прода |

Препрод: тот же Helm **replication** с PVC + ACL + (если цель автоfailover) три Sentinel **в одном ЦОДе**.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **пода/ноды внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой miss в кэше и ручного DR. Цифр QPS нет.

### Почему не stretch

Репликация **async**: OK клиенту ≠ запись на replica. Sentinel/Cluster живут heartbeat'ом. При плохом RTT — ложный failover и потеря хвоста. Cluster: primary, который не видит majority других primary дольше `cluster-node-timeout` (в учебнике пример **5000** мс), **перестаёт принимать запросы**. Stretch Cluster по ЦОДам не предлагаем. Официальный Helm Cluster **не ставит**; оператор Cluster в прод нельзя.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — Helm replication + (для автоfailover) Sentinel:

- чарт **0.11.0**, образ **9.1.1**;
- `replica.enabled=true`, PVC **обязателен** (на primary и replica);
- replica на **других нодах**, чем primary (anti-affinity / topology spread **внутри** ЦОДа);
- **3 Sentinel** в этом же ЦОДе, разные ноды, quorum **2**. Не 2 Sentinel. Файл `sentinel.conf` writable;
- клиенты: discovery через Sentinel (26379), затем 6379. Не хардкодить DNS primary навсегда;
- Helm **не** делает смену primary сам: без Sentinel падение writer = простой записи, пока не `REPLICAOF` / ручной failover.

Cluster 3+3 **не** ставим этим чартом и **не** ставим valkey-operator. Если когда-нибудь понадобятся шарды — отдельное решение (VM по cluster tutorial **внутри одного ЦОДа** или ждать GA оператора), не эта инструкция.

**Между ЦОДами** (B1 «мозг в одном ЦОДе» / B3 независимые наборы из `Valkey.md`):

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: Helm replica + 3 Sentinel. ЦОД-2: **независимый** Valkey (свой кэш) **или** только клиенты в ЦОД-1 | Локальный независимый кэш жив, но холоден. Клиенты «в домашний ЦОД» — кэша нет, пока приложение не переживает miss → озеро/API |
| **3 ЦОДа** | ЦОД-1: активный. ЦОД-2 и ЦОД-3: независимые Valkey **или** клиенты в ЦОД-1 + бэкапы RDB | То же; три независимых набора = три правды сессий, если ключи не шардированы приложением регионально |

Не копировать «как DaemonSet в каждый кластер» одни и те же сессии: получите три кэша. Для SoT это нормально (истина в озере). Для сессий UI — разлогин. Для идемпотентности внешнего вызова — риск повтора, пока ключ не восстановится.

### Предпосылки

- Dedicated nodes (taint), локальный диск, не NFS.
- NetworkPolicy: 6379 приложениям; 26379 только Sentinel+клиентам Sentinel.
- PKI, если TLS (в доке Valkey при включённом TLS по умолчанию **mTLS** клиентов; Helm `tls.requireClientCertificate` дефолт **false** — ослабление, в проде решить сознательно).
- Не Docker port-mapping / NAT для Sentinel.

### Установка (Kubernetes, ЦОД-1)

```bash
helm repo add valkey https://valkey.io/valkey-helm/
helm repo update
helm install valkey valkey/valkey --version 0.11.0 \
  --namespace valkey --create-namespace \
  -f valkey-prod-values.yaml
```

Смысл values (не полный файл — сверять README чарта **0.11.0**):

```yaml
image:
  repository: valkey/valkey
  tag: "9.1.1"
auth:
  enabled: true
  usersExistingSecret: valkey-users
  aclUsers:
    default:
      permissions: "~* &* +@all"   # сузить; не +@all у приложения
    replication-user:
      permissions: "+psync +replconf +ping"
replica:
  enabled: true
  replicas: 2
  replicationUser: replication-user
  persistence:
    size: 10Gi   # заглушка: диск AOF может быть больше датасета; цифр нагрузки нет
  minReplicasToWrite: 1          # если потеря OK больнее, чем отказ записи; для TTL-кэша можно 0 осознанно
  minReplicasMaxLag: 10          # как в примере Helm/conf, не «ваш SLA»
tls:
  enabled: true
  existingSecret: valkey-tls
podDisruptionBudget:
  enabled: true
  maxUnavailable: 1
```

Затем **отдельно** — 3 Sentinel (манифесты не из этого чарта). `SENTINEL CKQUORUM` в мониторинге. Auth Sentinel — тот же класс секретов, что у data-узлов.

В ЦОД-2/3 при независимом кэше: тот же чарт **0.11.0** / **9.1.1**, свой Secret, свой endpoint. GitOps overlay: разный bootstrap, не один values на страну.

### Конфигурирование (ЦОД-1)

| Параметр | Прод | Зачем |
|---|---|---|
| `maxmemory` | Ниже cgroup limit, запас на replica buffers / AOF rewrite / fragmentation | Иначе OOMKill |
| `maxmemory-policy` | Кэш: явно LRU/LFU, не молчаливый `noeviction` | При заполнении `noeviction` **встанет запись** |
| Persistence | AOF `everysec` + RDB | Вендор: если данные важны — *use both*; AOF-only не советует |
| ACL | Нет общего `default` с `+@all` у приложений; replication-user минимальный | Helm без `default` при `auth.enabled` = дыра |
| `min-replicas-to-write` | По политике (дефолт Helm **0** — легко забыть) | 0 = пишем при обрыве replica |

Замки/сессии, которые нельзя вытеснить: отдельный инстанс, не смесь с кэшем UI.

### Масштабирование (когда появятся цифры)

1. Замерить горячий набор, evict, лаг replica, отказы записи.
2. Больше RAM → вертикаль primary **или** второй независимый набор по домену ключей. Озеро «терабайты» сюда не скейлится.
3. Больше чтения → replica (клиенты терпят stale).
4. Больше записи → не этот Helm (один writer). Горизонталь записи — Cluster, которого у GA-установщика нет, или несколько наборов.
5. `io-threads` — опция под замер, не слайд.

### Проверка прода (пока это не пройдено — это не прод)

1. Образ/INFO = **9.1.1** (не 9.1.0).
2. Рестарт primary: replica **не** обнуляются.
3. Kill primary: приложение (не только cli) пишет в новый, **если** Sentinel стоит; без Sentinel — зафиксированный простой.
4. Попытка без ACL — отказ. 6379 не с мира.
5. Учение «ЦОД-1 мёртв»: клиенты ЦОД-2 делают miss/локальный кэш так, как обещали бизнесу.

### Сильные / слабые стороны (Helm replica + Sentinel в одном ЦОДе + независимый DR)

| Сильное | Слабое |
|---|---|
| Официальный сервер и официальный чарт 0.11.0 | Sentinel собираете сами; чарт Cluster не умеет |
| Кворум не зависит от межЦОДового RTT | Падение ЦОД-1 = нет этого кэша |
| BSD, без RSALv2 | Async: OK ≠ запись на replica |
| CVE 9.1.1 закрыты | Забыть пин — снова RCE на TLS |

**Не готов к проду**, если: standalone на три ЦОДа «для HA»; Helm replica без PVC; два Sentinel; `valkey-operator` как прод; Cluster через этот чарт; `latest` / 9.1.0; 6379 с мира без ACL; SoT клиентов в Valkey; клиенты без Sentinel при обещании автоfailover; `auth.enabled` без пользователя `default`; stretch без (и вопреки) запрета RTT; Dev-values скопированы в бой.

---

## Источники

- Релиз 9.1.1: https://github.com/valkey-io/valkey/releases/tag/9.1.1
- Образы: https://valkey.io/download/
- Helm repo / чарт 0.11.0: https://valkey.io/valkey-helm/ · https://github.com/valkey-io/valkey-helm
- Sentinel / replication / persistence / TLS: см. список в `Valkey.md`
- Operator **not ready for production**: https://github.com/valkey-io/valkey-operator/blob/main/README.md

Порога RTT для stretch в документации Valkey **нет** — поэтому stretch в этой инструкции не предлагается.
