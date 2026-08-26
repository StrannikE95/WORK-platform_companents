# HashiCorp Vault 2.0.4 — установка и конфигурирование

Связанный документ (глоссарий, Raft, Shamir/auto-unseal, Community vs Enterprise, почему так): `HashiCorp Vault.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch одного Raft на несколько ЦОДов **не делаем**: HashiCorp для Integrated Storage пишет RTT между зонами **< 8 мс**, но межЦОДовый ping у вас для Raft **запрещён** независимо от замера. Community **не** умеет Performance Replication: три независимых Vault — это три SoT секретов, не «единые ключи».

Версии: **Vault 2.0.4**, Helm-чарт **hashicorp/vault 0.34.1** (app 2.0.4), образ **`hashicorp/vault:2.0.4`** (pin по digest, не `latest`).  
Документация: https://developer.hashicorp.com/vault/docs · HA Raft в чарте: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Один Raft-кластер живёт **внутри одного ЦОДа** (одного Kubernetes). Между ЦОДами — только снимки. Это **вариант D** из `HashiCorp Vault.md`.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе отдельно (см. `Kubernetes.install.md`).
3. Редакция — **Community 2.0.4**. Enterprise (PR/DR, HSM PKCS#11, namespaces, автоснимки) в контексте нет; если лицензия появится — это **другой** путь, не «галка в values».
4. Dev — изолированная сеть; доли Shamir в примере не секрет, но в git их всё равно не кладём.
5. Нагрузки нет — поэтому **нет** цифры CPU/RAM «хватит для прода». Ориентир HashiCorp small (2–4 CPU, 8–16 ГиБ) — starting point, не расчёт RPS.
6. Storage — только **Integrated Storage (Raft)**. Consul как бэкенд не ставим.
7. Для 2 ЦОДов: мозг Vault в ЦОД-1, снимки в ЦОД-2. Для 3 ЦОДов: то же + копия снимка в ЦОД-3. Третий ЦОД **не** добавляет второй SoT.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** login приложений (K8s JWT / AppRole), KV, политики, увидеть seal и смену лидера. **Не** цель: отказ площадки и пик Injector.

### Предпосылки

- Dev-Kubernetes (или kind, понимая, что anti-affinity на одной ноде не докажет HA).
- Helm 3.6+.
- Namespace свободный; порты 8200/8201 внутри кластера.
- Сеть стенда не торчит в интернет.

Дефолт чарта — **standalone + file backend**. HashiCorp помечает это Security Warning. Для «потыкать CLI» сгодится. Для «как будем жить в проде» — **нет**: нет выборов лидера, нет PVC Raft, нет join. Режим `vault server -dev` в эту инструкцию не входит.

### Установка (Helm HA Raft — основной путь Dev)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
kubectl create namespace vault
```

Смысл values (не полный production-файл — сверять с докой чарта 0.34.1):

```yaml
server:
  image:
    repository: hashicorp/vault
    tag: "2.0.4"
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
injector:
  replicas: 1
```

```bash
helm install vault hashicorp/vault --version 0.34.1 -n vault \
  -f ./vault-ha.yaml
kubectl -n vault rollout status statefulset/vault
```

`./vault-ha.yaml` — локальная копия блока выше, не файл репозитория документации. Pin тега **2.0.4** в values обязателен: не полагаться на «чарт сам подтянет».

`init` **один раз** на первом поде (обычно `vault-0`), когда все три в `Running`, но ещё sealed:

```bash
kubectl -n vault exec -it vault-0 -- vault operator init
```

Доли Shamir и root token — в секретницу людей, **не** в git, не в общий чат, не в ConfigMap. Затем unseal **каждого** пода (Shamir: K из N долей на каждый рестарт). Join остальных в чарте обычно через `retry_join`.

Проверка:

```bash
kubectl -n vault exec -it vault-0 -- vault status
```

В выводе: версия линии **2.0.4**, `HA Enabled: true`, Storage **raft**, не file. Unsealed на кворуме.

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| 3 Raft voter | да | Увидеть смену лидера; не standalone |
| Shamir | да | Не строить KMS ради песочницы |
| TLS listener | можно отложить **только** в закрытой сети | Иначе PKI раньше политик |
| Injector | 1 реплика | Sidecar на тестовом поде |
| VSO | можно не ставить | Сначала API + одна политика + `vault kv` |
| Audit | хотя бы file | Увидеть, что секрет в логе хеширован |

Чего **не** упрощать: образ **2.0.4**; `init` один раз; доли не в git; Kubernetes auth с Role на ServiceAccount тестового пода; не класть озеро клиентов в KV.

### Проверка Dev

1. `vault status` — 2.0.4, Raft, unsealed.
2. `vault kv put/get` по политике роли приложения; root из приложения — отказ (после настройки auth root лучше отозвать и на тесте).
3. Убить под-лидер: кластер выбирает active, health сходится. Успех трёх подов **не** доказывает прод.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный HA-with-raft пример чарта | Нет auto-unseal, нет отказа ЦОДа, нет ёмкости |
| Команда видит seal и кворум до боя с KMS | Успешный KV get **не** доказывает прод |
| | UI под root приучает кликать секреты |

Препрод: TLS, 3–5 Raft, auto-unseal **или** отрепетированный Shamir runbook, два audit, snapshot Job — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **машины/пода внутри домашнего ЦОДа**; пережить отказ **целого ЦОДа** ценой RPO = интервал снимка и **ручного** restore. Цифр RPS нет — ниже минимум HA внутри площадки, не смета железа.

### Почему не stretch

Запись в Raft не быстрее, чем commit на кворум followers. HashiCorp для зон требует RTT **< 8 мс**; у вас ping между ЦОДами для Raft **запрещён**. Stretch дал бы таймауты и ложные выборы, а не защиту. Три Community-кластера «по ЦОДу» **без** PR — три SoT: политики и секреты разъедутся, единого отзыва не будет. PR есть только в Enterprise.

### Топология

**Внутри домашнего ЦОДа (ЦОД-1)** — один Raft-кластер Community:

- **5** voter (официальный «ideal size», failure tolerance **2** узла внутри площадки). Минимум **3**, если явно принимаете failure tolerance **1**;
- нечётное число; 2/4/6 как цель не использовать;
- anti-affinity: **отдельная нода на каждый под** (5 серверов → ≥ 5 нод пула);
- PVC `dataStorage` + отдельно `auditStorage`, локальный/зонный диск, **не NFS**;
- один active; standby форвардят на лидера (Community **не** performance standby);
- клиенты всех ЦОДов в штате ходят на этот кластер (TLS обязателен).

**Между ЦОДами — только снимки, не Raft:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: 5 (мин. 3) Raft. ЦОД-2: шифрованные snapshot (object store / лента), **не** второй кластер-SoT | Vault нет, пока поднять 3–5 узлов в ЦОД-2 + unseal + `raft snapshot restore`. RPO > 0 |
| **3 ЦОДа** | То же + копия снимка в ЦОД-3 | То же; третий ЦОД не даёт второго writer и не даёт PR |

Это надо **записать в ожидания руководства**, не прятать за «у нас три ЦОДа приложений».

Helm без overrides (standalone+file) в прод **не** идёт.

### Предпосылки прода

- Kubernetes в каждом ЦОДе (не один stretch-кластер). Vault-серверы — только в домашнем.
- Node pool: swap **выключен** (проверить). `disable_mlock=true` при Raft — норма чарта; не вешать IPC_LOCK на дефолтный образ вслепую.
- CSI с local/зональным диском. Object storage для снимков **есть или будет**.
- PKI для listener TLS. Health балансировщика: **`/v1/sys/health`**, не «TCP 8200 открыт». TLS end-to-end; терминировать TLS на Ingress и дальше plaintext HashiCorp для прода **не** рекомендует.
- NetworkPolicy: клиенты → 8200; узлы Vault ↔ 8200/8201; исходящие только куда надо (KMS, БД database engine, IdP, syslog).
- Решение по seal **до** init (см. таблицу ниже).
- Секреты и доли — не в Git.

### Установка (Kubernetes, домашний ЦОД)

1. Node pool + StorageClass + PKI + seal.
2. Helm **hashicorp/vault 0.34.1**, образ **`hashicorp/vault:2.0.4`**.
3. Values: `server.ha.enabled=true`, `server.ha.raft.enabled=true`, `replicas: 5` (или 3 с принятым риском), PVC data+audit, TLS listener, Injector **≥ 2** реплики.
4. Дождаться Running → `vault operator init` **один раз**.
5. Раздать Shamir **или** recovery keys (при auto-unseal). Root — включить Kubernetes auth + политики админов → **revoke**.
6. Join через `retry_join` / Autopilot. Dead server cleanup включать *после* init.
7. Два audit device сразу после init, **до** боевых секретов (по умолчанию audit выключен).
8. CronJob/`vault operator raft snapshot save` → шифрованный store в других ЦОДах. Restore прогнать на стенде. Автоснимки вендора — **Enterprise**.

Обновление: **последовательно**, сохраняя кворум. Не rolling «все сразу».

### Seal и unseal

Без auto-unseal Kubernetes **не** считается прод-готовым: reschedule = sealed, пока люди не внесут доли на **каждый** под.

| Механизм | Редакция | Когда |
|---|---|---|
| Cloud KMS | Community и Enterprise | Есть KMS и сеть до него. Креды не в HCL plaintext |
| PKCS#11 HSM | **Только Enterprise** | Требование СКЗИ |
| Transit (другой Vault) | Оба | On-prem без HSM: маленький unseal-кластер. Shamir переезжает туда |
| Shamir | Оба | Только 24/7 процедура split-knowledge. На проде K8s — слабое место |

Init один раз. Доли и recovery **не в git**. Все доли в одном сейфе одного админа = один человек = весь контур.

### Конфигурирование (смысл, не полный HCL)

| Параметр | Прод | Зачем |
|---|---|---|
| Raft voter | 5 (мин. 3) в **одном** ЦОДе | Кворум не зависит от межЦОДового RTT |
| Listener | TLS, не PLAIN на общей сети | Hardening HashiCorp |
| Audit | ≥ 2 (file на отдельном томе + syslog/socket в SIEM) | Нельзя записать ни на одно → Vault отказывается обслуживать |
| UI | выкл или только admin-сеть | Не интернет |
| TTL токенов | короткие | Отзыв при компрометации пода |
| Политики | least privilege, без широких `*` | Иначе SoT ключей = дыра |
| Injector | ≥ 2 | Иначе выкат webhook = нельзя создать аннотированные поды |
| VSO | только с encryption-at-rest etcd | Иначе обошли Vault через API Kubernetes |

Клиенты (микросервисы, Camunda, интеграционное API) переживают короткую 5xx при смене лидера — ретраи в **коде**, не в Helm.

### Масштабирование (когда появятся цифры)

1. Замерить login/s, KV, transit, пик Injector при массовом рестарте подов.
2. Упёрлись в active — вертикаль лидера (диск IOPS, RAM по small/large). **Ещё voter не ускоряет запись.**
3. Чтения в другом ЦОДе без езды на лидера — только Enterprise PR; в этой схеме клиенты едут в ЦОД-1.
4. Терабайты озера на CPU Vault не давят, пока вы не гоняете их через transit побайтово.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия **2.0.4** на всех voter; Storage raft; TLS.
2. Убить active: выборы, клиенты живы после ретрая.
3. Drain одной ноды: кворум жив, PVC «свой».
4. Диск audit 100% на стенде — понять, что API встаёт; мониторинг места обязателен.
5. Restore snapshot на **чистый** кластер в другом ЦОДе хотя бы раз (замер RTO).
6. Root из приложения не проходит; доли не в репозитории.
7. Суперширокая политика `*` не выдана сервисам.

### Сильные / слабые стороны прод-схемы (мозг в одном ЦОДе + снимки)

| Сильное | Слабое |
|---|---|
| Raft не зависит от межЦОДового RTT | ЦОД-1 — SPOF для *всех* секретов платформы |
| Один SoT; согласовано с запретом stretch и с Community (нет PR) | RPO = интервал снимка; RTO = ручной restore + unseal |
| Проще сеть, чем 2-2-1 | Легко забыть, что «3 ЦОДа приложений» ≠ «3 ЦОДа Vault» |
| | Клиенты ЦОД-2/3 в штате идут через город — TLS и сеть ваши |

**Не готов к проду**, если: дефолтный standalone/file; `latest`; один узел Raft; stretch «потому что в доке 8 мс»; три независимых Community как SoT; Shamir без runbook на K8s; один audit на диске OS; нет снимков вне кластера; PLAIN listener; root живёт «на всякий случай»; доли в git; Dev-values скопированы в бой.

---

## Источники

- Релиз 2.0.4: https://github.com/hashicorp/vault/releases/tag/v2.0.4
- Helm 0.34.1, предупреждение pro standalone: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm
- HA + Raft в чарте: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft
- Reference architecture: 5 узлов, RTT < 8 ms, запись не масштабируется числом узлов: https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture
- Integrated Storage, кворум, Autopilot: https://developer.hashicorp.com/vault/docs/internals/integrated-storage
- Production hardening: https://developer.hashicorp.com/vault/docs/concepts/production-hardening
- Редакции (нет PR/DR в Community): https://developer.hashicorp.com/vault/tutorials/get-started/available-editions
- Снимки Community save/restore: https://developer.hashicorp.com/vault/docs/commands/operator/raft
- Audit (отказ, если нельзя записать): https://developer.hashicorp.com/vault/docs/audit
- Правила и пробелы: `HashiCorp Vault.md`

Порог **8 мс** у HashiCorp есть; stretch в этой инструкции всё равно не предлагается — запрет площадки на Raft между ЦОДами.
