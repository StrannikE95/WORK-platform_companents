# HashiCorp Vault 2.0.4 — развёртывание и настройка

Версия ПО: **Vault 2.0.4** (релизная дата 4 августа 2026; на дату подготовки документа это последний стабильный патч линии 2.0).  
Документация: https://developer.hashicorp.com/vault/docs  
Релиз: https://github.com/hashicorp/vault/releases/tag/v2.0.4  
Официальный путь на Kubernetes: Helm-чарт **hashicorp/vault 0.34.1** (app version **2.0.4**).  
Образы: Community — `hashicorp/vault:2.0.4`; Enterprise — `hashicorp/vault-enterprise` с тегом линии `2.0.4-ent` (pin по digest, не `latest`).

Этот текст — не мануал «скопируй helm install», а правила, без которых экземпляр **не** будет одновременно отказоустойчивым, масштабируемым и безопасным.

Vault **не было** в исходном описании архитектуры (Kafka, Camunda, озеро данных, интеграционное API). Ниже — как поставить секреты/ключи на ту же платформу. Это не замена WAF, Falco, NetworkPolicy и не «кнопка безопасность включена».

---

## Глоссарий терминов

| Термин | Простыми словами |
|---|---|
| **Секрет** | То, что нельзя светить в git/ConfigMap: пароль БД, ключ API ведомства, сертификат, токен. Vault хранит и *выдаёт* их по политике. |
| **Secrets engine (движок секретов)** | Модуль внутри Vault: KV (просто ключ-значение), database (динамический пароль в Postgres), pki (выпускает сертификаты), transit (шифрует байты, сам данные не хранит). |
| **Auth method** | Как клиент доказывает, кто он: Kubernetes (JWT пода), AppRole (id+secret для не-K8s), OIDC/LDAP для людей. Без auth любой с адресом Vault — никто. |
| **Policy (ACL)** | Текст «этому токену можно `read` по пути `secret/data/app/*`, и больше ничего». Это не NetworkPolicy Kubernetes. |
| **Token / lease** | Короткоживущий пропуск после login. У динамических секретов есть **lease**: срок жизни; Vault умеет отозвать. |
| **Seal / unseal (печать)** | После старта Vault **запечатан**: данные на диске есть, но мастер-ключ в памяти нет, API почти мёртв. Unseal — расшифровать этот ключ и начать работу. |
| **Root key (мастер-ключ)** | Ключ, которым Vault расшифровывает всё остальное. В Shamir его режут на доли; при auto-unseal его оборачивает KMS/HSM/другой Vault. |
| **Shamir** | Ручной unseal: при `init` получают N долей, чтобы открыть — нужны любые K из них (часто 5 долей, порог 3). Каждую долю после рестарта пода нужно снова вводить. |
| **Auto-unseal** | Unseal без людей: Cloud KMS, HSM (PKCS#11) или Transit другого Vault. На Kubernetes HashiCorp **настоятельно** рекомендует auto-unseal: поды переезжают часто. |
| **Recovery keys** | При auto-unseal вместо «unseal-долей» выдают recovery-ключи: для generate-root и чрезвычайных операций, не для каждого рестарта. |
| **Integrated Storage / Raft** | Встроенное хранилище Vault. Данные реплицируются между узлами по протоколу Raft. Отдельный Consul **не нужен**. |
| **Consul storage** | Старый внешний бэкенд HA. Официально ещё жив, но для нового кластера HashiCorp ведёт на Integrated Storage. Это *ещё один* кластер для сопровождения. |
| **Кворум (quorum)** | Большинство голосов Raft. Для 3 узлов = 2, для 5 = 3. Нет кворума → **нет записи** (новые секреты, login, смена политик). |
| **Active / standby** | В кластере **один** активный (лидер): принимает запись. Остальные — standby. В Community standby **пересылает** почти все запросы лидеру. |
| **Request forwarding** | Standby получил запрос клиента → отправил лидеру по порту кластера (обычно 8201) → вернул ответ. Клиенту «всё равно, на какой узел попал». |
| **Performance standby** | **Только Enterprise:** standby сам обслуживает *чтения* (KV get, transit encrypt/decrypt), запись всё равно у лидера. |
| **Performance replication (PR)** | **Только Enterprise:** второй кластер в другом ЦОДе отдаёт чтения локально; запись, меняющая общее состояние, уходит на primary. |
| **Disaster Recovery (DR) replication** | **Только Enterprise:** тёплый кластер-копия. Не обслуживает клиентов, пока его не повысят. Это провал *целого* кластера/региона, не падение одного пода. |
| **Autopilot** | Механизм Raft: новый узел сначала non-voter, догоняет лог, потом получает голос. Dead server cleanup (вычищать мёртвые узлы) **по умолчанию выключен** — его включают отдельно. |
| **Snapshot** | Снимок Raft-состояния. Community: вручную `vault operator raft snapshot save/restore`. Автоснимки по расписанию — **Enterprise**. |
| **Audit device** | Журнал «кто что спросил и что ответили». Чувствительные поля хешируются. Если Vault **не может записать аудит хотя бы на одно** включённое устройство — **отказывается обслуживать запрос**. Это задумано. |
| **Listener 8200 / cluster 8201** | 8200 — API клиентов. 8201 — Raft, forwarding, репликация между узлами. Между узлами Vault сам поднимает mTLS. |
| **`/v1/sys/health`** | Официальная точка health-check для балансировщика. Коды различают sealed / standby / active. |
| **mlock / disable_mlock** | `mlock` запрещает выгружать память процесса в swap. С Integrated Storage HashiCorp **рекомендует `disable_mlock = true`**: иначе BoltDB (файл Raft) раздувает RSS и легко ловит OOM. Тогда swap на ноде должен быть **выключен или зашифрован**. Helm-чарт сам ставит `disable_mlock = true`, если не переопределить. |
| **Vault Agent Injector** | Admission webhook: по аннотации в под вставляют sidecar Agent, секреты появляются как файлы в volume. Приложение может не знать про Vault. |
| **Vault Secrets Operator (VSO)** | **Отдельный** Helm-чарт (не тот же, что сервер). Кладёт секреты Vault в обычные Kubernetes Secret. Удобно, но секрет оказывается в etcd. |
| **CSI provider** | Секреты как volume через Secrets Store CSI. В чарте сервера по умолчанию **выключен**. |
| **AppRole** | Auth для машин не из Kubernetes: `role_id` + `secret_id`. Типично для батчей, внешних воркеров. |
| **Transit** | «Шифрование как сервис»: приложение шлёт plaintext, получает ciphertext. Данные озера **живут в озере**, ключ — в Vault. |
| **Namespace (Vault)** | Логический «мини-Vault» внутри одного кластера. **Только Enterprise.** Не путать с namespace Kubernetes. |
| **Root token** | Абсолютный ключ после `init`. Его используют для первичной настройки и **отзывают**. Держать root в проде «на всякий случай» — дыра. |
| **Stretch-кластер** | Один Raft-кластер, узлы физически в нескольких ЦОДах, **один** кворум. |
| **RPO / RTO** | Сколько секретов/ключей готовы потерять / как быстро поднять Vault после аварии. Это не RPO озера данных. |
| **Community / Enterprise** | Две редакции HashiCorp. Community — self-hosted, без PR/DR-репликации, без HSM PKCS#11, без namespaces. Enterprise — коммерческая лицензия. |

---

## Основные элементы системы и зависимости

### Что входит в Vault 2.0.4 (это одно ПО)

- **Сервер `vault`** — API, политики, движки, Raft, seal.
- **Конфиг HCL** (`listener`, `storage "raft"`, опционально `seal`).
- **CLI `vault`** — init/unseal/policy/snapshot; тот же бинарник.
- **UI** — веб-консоль (включается отдельно; не обязательна для работы API).
- **Плагины** движков и auth (часть встроена, часть — внешние процессы).

### Экосистема (отдельные компоненты, без них прод приложений обычно неполный)

| Компонент | Зачем | Это не «сам Vault» |
|---|---|---|
| **Helm-чарт hashicorp/vault** | Ставит StatefulSet серверов, Injector, опционально CSI/UI | Не оперирует кластером: init, unseal, бэкап, апгрейд — ваша ответственность (прямо сказано в deployment guide) |
| **Vault Agent Injector** | Sidecar-секреты в поды | Webhook; если он мёртв — новые поды с аннотациями не создаются |
| **Vault Secrets Operator** | Sync в Kubernetes Secret | Другой чарт, другой operational burden |
| **Балансировщик / Ingress с TLS passthrough** | Клиенты не ходят на один под | Нужен health `/v1/sys/health`; завершать TLS на Ingress HashiCorp для прода **не** рекомендует |
| **KMS / HSM / Unseal-Vault** | Auto-unseal | Внешний корень доверия. Если его нет — Shamir и люди |
| **Сбор логов / SIEM** | Куда уезжает audit | Без приёмника audit на PVC рано или поздно забьёт диск → Vault встанет |

### Чего в Vault нет (частая путаница)

| Нужно системе | Это не Vault | Зачем помнить |
|---|---|---|
| Озеро эталонных / клиентских данных | Ваше хранилище (БД, object store) | Vault — секреты и ключи, не SoT клиентов. Терабайты персоналки в KV — антипаттерн и путь к смерти BoltDB. |
| Шина событий | Kafka | Vault не заменяет топики. |
| Процессы | Camunda | Vault может выдать пароль воркеру, не оркестрирует BPMN. |
| Интеграции с ведомствами | Ваше интеграционное API | Vault хранит клиентские сертификаты/ключи к внешним API, сам HTTP в госорганы не ходит. |
| Репликация кластера на 3 ЦОДа «из коробки» в Community | Performance / DR replication | В таблице редакций HashiCorp у Community: **нет** DR и **нет** PR. Stretch Raft — это *один* кластер, не репликация. |
| Горизонтальный масштаб *записи* | Не обещается ни одной редакцией внутри одного кластера | Пишет **один** active. Добавление узлов Raft не ускоряет запись. |
| ГОСТ TLS / СКЗИ | Не входит | Если требование ИБ — отдельный контур, это критичный пробел. |
| Шифрование etcd Kubernetes | Конфиг API-сервера | VSO кладёт секреты в K8s Secret: без encryption-at-rest etcd они лежат как есть (base64 ≠ шифр). |

### Зависимости окружения (обязательны)

- **Диск:** современный SSD/NVMe. HashiCorp прямо пишет: магнитные диски дают «dramatic performance penalty». **Не NFS / не RWX-шара как `storage "raft" { path }`.** Нужен PVC на узел (Helm: `dataStorage`).
- **Сеть между узлами Vault:** TCP **8200** (join/API) и **8201** (Raft, forwarding) в обе стороны. Клиенты — до балансировщика на 8200/443.
- **Часы:** NTP. TTL токенов и даты в PKI-сертификатах завязаны на время; сильный skew ломает failover.
- **Kubernetes (у вас заявлен):** StatefulSet, anti-affinity в чарте = **отдельная нода на каждый под Vault**. 5 серверов Vault → минимум 5 нод в пуле (с учётом зон — больше). Helm 3.6+.
- **PKI** для listener TLS (свой CA). Между узлами mTLS на 8201 Vault согласует сам после join.
- **Место для audit**, лучше отдельный том (`auditStorage` в чарте) + второй audit (syslog/socket в центральный сборщик).

### Как Vault стыкуется с вашей архитектурой

```
Микросервисы / Camunda-воркеры / интеграционное API
        │  login (K8s JWT / AppRole) + read secret / transit / PKI
        ▼
   Vault (этот документ)
        │  динамический пароль, cert, decrypt
        ▼
   Postgres / Kafka SASL / TLS к ведомствам / шифрополя в озере
```

Vault — **источник истины для секретов и ключей**, не для клиентских карточек. Озеро остаётся SoT данных; при необходимости чувствительные поля шифруются через **transit**, ciphertext лежит в озере.

---

## Краткие вводные

### Зачем вам Vault в этой архитектуре

У вас много сервисов, 30+ интеграций, Kafka, процессный движок. Без центра секретов получается:

1. Пароли БД и ключи к ведомствам в ConfigMap/CI.
2. Долгоживущие учётки, которые никто не ротирует.
3. Нет единого отзыва при компрометации пода.

Vault закрывает выдачу **коротких** учёток (database engine), сертификатов (pki), ключей шифрования (transit) и аудит доступа.

### Как устроена отказоустойчивость (идея, не магия)

Это **похоже на etcd/Kafka-контроллеры**, не на Falco.

| Что падает | Community, один Raft-кластер |
|---|---|
| Active-под | Выборы лидера, standby становится active. Клиенты, попавшие на standby, форвардятся. Короткий разрыв записи. |
| 1 из 3 узлов | Кворум 2 — жив. |
| 1 из 5 узлов (или 2 из 5) | Кворум 3 — жив. Официальный ориентир «идеальный размер» — **5**, чтобы пережить **2** узла. |
| ЦОД, в котором **большинство** голосов | Кворум потерян → **запись и смена лидера невозможны**. Чтение с единственного выжившего без кворума тоже не «штатный HA». |
| Весь кластер / все 3 ЦОДа города | Community **не** даёт DR-реплику. Остаётся snapshot + процедура restore. |
| Sealed после рестарта без auto-unseal | Пока люди не внесут доли Shamir на **каждый** под — сервиса нет. На Kubernetes это почти гарантированный простой. |

Следствие: HA Vault = **нечётное число voter ≥ 3**, размазанных по доменам отказа так, чтобы потеря **одного ЦОДа** оставляла кворум, плюс **auto-unseal**, плюс **снимки вне кластера**.

### Как устроено масштабирование

- **Запись** в одном кластере не масштабируется числом узлов. Узкое место — диск лидера + сеть до кворума followers. HashiCorp: extra members **не** увеличивают производительность операций, которые пишут в storage.
- **Чтение** в Community всё равно часто идёт через forwarding на active → active остаётся бутылкой.
- Горизонтальные чтения внутри кластера — **performance standby (Enterprise)**.
- Чтения в другом ЦОДе без езды на лидера — **performance replication (Enterprise)**.
- Официальная таблица «business requirements» HashiCorp для Community: *Scale to meet business demand* — **нет**; *Disaster recovery support* — **нет**. Это их формулировка для выбора редакции, не замер вашего RPS.

Ориентиры железа (HashiCorp reference architecture, **минимум без вашего профиля нагрузки**):

| Размер | CPU | RAM | Диск | IOPS / throughput |
|---|---|---|---|---|
| Small | 2–4 ядра | 8–16 ГБ | 100+ ГБ | 3000+ IOPS, 75+ МБ/с |
| Large | 4–8 ядер | 32–64 ГБ | 200+ ГБ | 10000+ IOPS, 250+ МБ/с |

Для облака они же запрещают burstable типы (t2/t3 и аналоги). В Helm K8s-гайде для small-кластера пример requests: **8Gi RAM / 2000m CPU**, limit memory **16Gi**. Цифры «хватит под ваши терабайты озёра» **нет**: нагрузка Vault — *число запросов к секретам*, не объём озера.

### Сеть между ЦОДами — жёсткое условие HashiCorp

В reference architecture Integrated Storage: задержка между availability zone **должна быть меньше 8 мс**.  
Ваш RTT **не измерен**. Пока нет замера — stretch на 3 ЦОДа это **гипотеза**, не решение. Если RTT 8 мс и выше или «плавает», выборы и запись будут страдать (Raft bound by disk **and** network latency).

### Безопасность самого Vault

Это **единая точка**, зная которую, можно получить ключи ко всем БД и интеграциям. Компрометация Vault хуже компрометации одного микросервиса.

Поэтому `helm install` в standalone (дефолт чарта: **один** сервер, **file** backend, часто без TLS) — это не прод. HashiCorp помечает это Security Warning.

---

## Допущения

Ниже то, чего **не было** в контексте, но без чего нельзя дать конкретную схему. Если допущение неверно — схему надо пересмотреть.

1. **Целевая редакция для текста — Community 2.0.4**, потому что лицензия Enterprise в контексте не названа. Если купите Enterprise — для 3 ЦОДов правильный путь другой (PR + DR), см. прод. Юридические ограничения BSL Community ваша служба должна проверить отдельно; здесь только техническое различие редакций из документации HashiCorp.
2. **Прод крутится в Kubernetes**, установка — **Helm hashicorp/vault 0.34.1**. HashiCorp в production hardening предпочитает «железо → VM → контейнер» (меньше соседей на ноде). На K8s это осознанный компромисс: выделенный namespace, лучше выделенный node pool / отдельный кластер (их Kubernetes deployment guide: *Install Vault on a dedicated Kubernetes cluster when possible*).
3. **Storage — только Integrated Storage (Raft).** Consul как бэкенд не предлагаю: лишний HA-кластер. Третьесторонние storage в Enterprise **не** поддерживаются (в матрице редакций: third party storage — Community да, Enterprise нет).
4. **Три ЦОДа = три зоны отказа**, если это реально независимые площадки. Три зала с общим питанием — не три ЦОДа.
5. **Stretch Community принимается только при подтверждённом RTT < 8 мс** (порог HashiCorp для AZ). Иначе — кластер в одном (или двух) ЦОДах + снимки, с явным RPO.
6. **Нагрузки на Vault нет** — нет числа «RPS login/secret». Железо ниже — официальный *starting point*, не расчёт.
7. **Формального SLA 24/7 нет.** «Беспрерывная работа» ≠ переживание двух ЦОДов и ≠ Community DR.
8. **Корпоративный PKI есть или будет.** TLS на listener в проде обязателен (hardening + K8s guide).
9. **Облачного KMS может не быть** (3 ЦОДа в одном городе, on-prem). Тогда auto-unseal: PKCS#11 HSM (**Enterprise**), либо **Transit auto-unseal** через маленький отдельный Vault (переносит Shamir на «unseal-кластер», официальный паттерн HashiCorp), либо Shamir (операционно опасно на K8s).
10. **Тестовый стенд изолирован.** На нём допустимы Shamir и даже TLS off *только в закрытой сети*. Прод-профиль unseal/TLS на тест один в один не копируем, но **Raft HA и audit** лучше включить сразу, иначе тест не показывает «диск audit забил — API встал».
11. **Camunda, Kafka, озеро, интеграционное API** — клиенты Vault. Их политики и auth method проектируются отдельно; от кластера им нужны адрес, CA, роль.
12. **Не кладём в Vault терабайты клиентских данных.** KV/transit keys — гигабайты метаданных, не озеро.

---

## Критически важные условия, которых нет в исходном контексте

Их нужно закрыть **до** «поставили Helm и забыли».

| Пробел | Почему это ломает решение |
|---|---|
| **Community vs Enterprise (бюджет/лицензия)** | Без Enterprise нет PR/DR, нет HSM auto-unseal, нет namespaces, нет автоснимков, нет performance standby. Три ЦОДа «как у вендора» тогда **недоступны**. |
| **RTT p50/p95/p99 между всеми парами ЦОДов** | Порог HashiCorp для Raft между зонами — **< 8 мс**. Выше — stretch Community будет источником аварий, а не защиты. |
| **Один Kubernetes на 3 ЦОДа или три кластера** | Stretch Raft требует pod-to-pod **8200/8201** между ЦОДами. Три изолированных K8s без L3 — один Raft не собрать. |
| **RPO/RTO секретов** и что значит «упал ЦОД» | 5 узлов 2-2-1 переживают **один** ЦОД. Два ЦОДа из трёх при 3 или 5 voter обычно **убивают кворум**. Потеря города — только snapshot (Community). |
| **Чем unseal'ить on-prem** | Нет KMS в облаке + нет Enterprise HSM = либо Transit-Vault, либо Shamir. Без решения каждый `kubectl drain` — ночной звонок держателям долей. |
| **Профиль запросов к Vault** (login/s, KV/transit/PKI, пик деплоя) | Без этого нельзя выбрать small vs large и понять, упрётесь ли в одного active. |
| **Требования ИБ: ГОСТ, 152-ФЗ, КИИ, HSM, срок хранения audit, запрет контейнеров для СКЗИ** | Меняют редакцию, seal, место установки (K8s vs выделенные VM), куда можно писать audit. |
| **Есть ли уже секреты в K8s Secret / CI** | Миграция и политика «секреты только через Vault» — процесс, не Helm values. |
| **Кто держит Shamir/recovery и root** | Без процедуры split-knowledge init бесполезен: все доли в одном сейле админа = один человек = весь контур. |

---

## Краткое описание ключевых этапов и элементов развертывания

Общий каркас (и тест, и прод):

1. Зафиксировать **2.0.4** + чарт **0.34.1**, pin образа по digest.
2. Выбрать редакцию и seal (Shamir / Cloud KMS / Transit / HSM).
3. Поднять **не standalone**: HA + `server.ha.raft.enabled=true`.
4. PVC под data и отдельно под audit; StorageClass локальный/зонный, не NFS.
5. TLS listener; сеть 8200/8201; NTP.
6. `init` **один раз** на первом поде → раздать доли/recovery → **отозвать root** после включения auth.
7. Join остальных (в чарте обычно `retry_join`); Autopilot включён по умолчанию для стабилизации новых voter.
8. **Два** audit device; метрики; снимки Raft **вне** кластера.
9. Auth: Kubernetes для подов, AppRole/OIDC по необходимости; политики least privilege; короткие TTL.
10. Способ доставки секретов в приложения (Injector и/или VSO) — *после* живого API, не вместо HA сервера.
11. Учение: убить active, убить зону, заполнить диск audit, restore из snapshot.

Дальше — два режима.

---

### 1 инстанс: тестовый стенд, 1 ЦОД, без нагрузки

**Цель стенда:** понять login приложений, шум политик, как выглядит seal, куда едет audit. **Не** цель: доказать, что прод переживёт ЦОД и пик деплоя.

#### Топология

Минимально осмысленно (уже не дефолт чарта):

- namespace `vault`;
- Helm 0.34.1, образ `hashicorp/vault:2.0.4`;
- `server.ha.enabled=true`, `server.ha.raft.enabled=true`, **3** реплики (кворум есть, failure tolerance = 1);
- Shamir unseal допустим: рестартов мало, людей рядом;
- TLS можно отложить **только** если сеть стенда закрыта; иначе сразу свой CA — меньше сюрпризов в проде;
- Injector 1 реплика; UI только внутри сети;
- один file-audit + `kubectl logs` как второй «глаз» на время учёбы — но понимать, что в проде file-only = риск блокировки.

`helm install` без overrides = **standalone + file storage**. Для знакомства с CLI сгодится. Для «как будем жить в проде» — **не** сгодится: нет выборов лидера, нет модели PVC, нет join.

Режим `dev` (`vault server -dev`) в этот документ не входит: память, всё unsealed, root на экране.

#### Что сознательно упрощаем

| Параметр | На тесте | Почему можно |
|---|---|---|
| 5 узлов / 3 ЦОДа | 3 пода, 1 ЦОД | Нет требования пережить площадку |
| Auto-unseal | Shamir | Не строить KMS ради песочницы |
| Performance/DR | нет | Community и так без них |
| Жёсткий small/large sizing | меньше RAM, чем 8 Ги | Нет нагрузки; следить, чтобы не OOM на Raft |
| VSO | можно не ставить | Сначала API + одна политика + `vault kv` |

#### Чего на тесте **не** стоит упрощать

- Raft, не file-backend, если цель — учиться провалу пода.
- `init` один раз, доли не в git и не в общем чате.
- Хотя бы один audit device — увидеть, что секрет в логе **хеширован**.
- Проверка: убить `vault-0`, пока он leader → кластер выбирает нового, `vault status` / health сходятся.
- Auth Kubernetes с одной Role, привязанной к ServiceAccount тестового пода — иначе в проде внезапно окажется, что JWT review не настроен.

#### Сильные стороны такой схемы

- Поднимается за часы, совпадает с официальным HA-with-raft примером чарта (там как раз 3 пода).
- Команда видит seal, кворум и политики до боя с PKI и KMS.

#### Слабые стороны (обязательно понимать)

- Нет модели отказа ЦОДа, нет доказанной ёмкости, нет auto-unseal.
- Успешный failover трёх подов в одном ЦОДе **не** доказывает stretch на трёх площадках.
- UI приучает «ходить кликать секреты»; в инциденте нужен audit + политики, не клики под root.

Практическая рекомендация: препрод = маленький **прод-профиль** (TLS, 3–5 Raft, auto-unseal или отрепетированный Shamir runbook, два audit, snapshot Job), даже без боевого трафика.

---

### Прод: 3 ЦОДа, нагрузка

Цифр «ядер на Vault под терабайты озера» **нет**. Ниже правила, без которых экземпляр не считается готовым.

#### Шаг 0. Макроархитектура (сделать до установки)

Сначала редакция и сеть, потом Helm.

**Вариант C — Community, stretch Raft на 3 ЦОДа (один кластер).**

Условие: измеренный RTT **< 8 мс**, стабильный L3, порты 8200/8201.

- **5** voter (официальный «ideal size», failure tolerance **2** узла).
- Раскладка по ЦОДам **2-2-1** (не 3-1-1: потеря ЦОДа с тремя голосами = потеря кворума при 5 узлах, quorum=3).
- Anti-affinity + `topology.kubernetes.io/zone` = id ЦОДа. Чартовый anti-affinity требует **разных Kubernetes-нод**; нод в пуле должно хватить (5+).
- Один active; клиенты всех ЦОДов ходят на него (напрямую или forwarding). Запись не быстрее, чем диск лидера + RTT до кворума.
- Переживает **один** ЦОД. Не переживает два. Не переживает «весь город» без снимка.

**Вариант D — Community, кластер не растягивать.**

Если RTT неизвестен, > 8 мс, или K8s изолированы:

- 5 (минимум 3) узлов **внутри одного ЦОДа** (или двух — с пониманием, что потеря одной из двух зон при неудачном кворуме = 50% шанс, это буквально пример HashiCorp).
- Снимки Raft копировать в другие ЦОДы (Job/`vault operator raft snapshot save`, шифрованный object store / лента).
- Падение «домашнего» ЦОДа = Vault нет, пока не restore. RTO — время поднять 3–5 новых узлов + unseal + snapshot restore. Это надо **записать в ожидания руководства**, не прятать.

Три независимых Community-кластера «по одному на ЦОД» **без** синхронизации политик/секретов — три источника истины. HashiCorp для этого сделал PR, его в Community нет. Так не закрыть «единые ключи для 30 интеграций».

**Вариант E — Enterprise (если лицензия будет).**

Это вендорский ответ на 3 ЦОДа и нагрузку чтений:

- Primary HA-кластер (5 Raft) + **performance secondary** в других ЦОДах (локальные чтения, KV/transit).
- **DR secondary** (горячий простой) на отказ primary.
- Performance standby внутри кластера — горизонталь чтений.
- HSM PKCS#11 auto-unseal, namespaces, автоснимки, seal wrap/FIPS — по требованиям ИБ.

Пока лицензии нет, планировать E как «включим галку» нельзя: другие образы, другая операционка, другие сетевые потоки репликации.

**Пока не закрыты редакция + RTT + топология K8s, нельзя честно выбрать C, D или E.** Дальше — общее для серверного кластера; расхождение — только в числе кластеров и в том, есть ли PR/DR.

##### Сильные / слабые стороны (C, stretch Community)

| Сильное | Слабое |
|---|---|
| Один SoT секретов, переживание 1 ЦОДа при 2-2-1 | Жёсткий порог 8 мс; запись всегда у одного лидера |
| Нет лицензии Enterprise | Нет DR на потерю кластера/города |
| Проще, чем три кластера | Ошибка политики/выката сразу на всех площадках |
| | RTT каждого write включает удалённых followers |

##### Сильные / слабые стороны (D, один ЦОД + снимки)

| Сильное | Слабое |
|---|---|
| Raft не зависит от межЦОДового RTT | ЦОД с Vault — SPOF для *всех* секретов |
| Проще сеть | RPO = интервал снимка; RTO = ручной restore |
| | Легко забыть, что «3 ЦОДа приложений» ≠ «3 ЦОДа Vault» |

##### Сильные / слабые стороны (E, Enterprise PR+DR)

| Сильное | Слабое |
|---|---|
| Локальные чтения, вендорский DR | Лицензия, сложность, два/три кластера сопровождать |
| HSM, namespaces, автоснимки | Репликация не копирует токены/lease (у PR); это надо понимать в runbook |
| Соответствует их таблице «scale / DR» | Не лечит плохой RTT внутри *одного* Raft-кластера |

#### Узлы Vault (ядро отказоустойчивости)

- Нечётное число **voter**: прод-ориентир HashiCorp — **5**. 3 допустимо, если явно принимаете failure tolerance **1** (один узел *или* один ЦОД при раскладке 1-1-1).
- 2, 4, 6 узла как целевой размер не использовать: кворум не лучше, лишняя сложность (таблица quorum в docs Integrated Storage).
- Autopilot: не добавлять пачку новых узлов одновременно «как Deployment». Dead server cleanup включить *после* init, иначе мёртвые поды остаются в peer set.
- Обновление: официальная идея — **последовательно**, сохраняя кворум (для 5: временно 7, вывести 2 старых, и т.д.). Не rolling «все сразу».
- PVC `dataStorage` + `auditStorage`; Recreate пода должен найти **свой** том. Не общий диск на троих.
- Swap на нодах пула Vault: **выключен** (K8s часто так и есть — **проверить**, не верить). `disable_mlock=true` при Raft — норма для контейнера HashiCorp; IPC_LOCK на дефолтный образ не вешать (capability на бинарнике нет, admission может отвергнуть под).

#### Seal и операционка рестартов

Без auto-unseal Kubernetes **не** считается прод-готовым: reschedule = sealed.

| Механизм | Редакция | Когда уместен |
|---|---|---|
| Cloud KMS (AWS/GCP/Azure) | Community и Enterprise | Есть облачный KMS и сеть до него. Креды **не** в HCL plaintext — instance profile / env. |
| PKCS#11 HSM | **Только Enterprise** | Требование СКЗИ/HSM. |
| Transit (другой Vault) | Оба (это engine) | On-prem без HSM: маленький unseal-кластер. Shamir переезжает туда; его тоже надо уметь открывать. |
| Shamir | Оба | Только если есть 24/7 процедура и люди с долями. На проде K8s — слабое место. |

После `init` с auto-unseal сохранить **recovery keys** так же серьёзно, как Shamir. Root token — настроить Kubernetes auth + политики админов → **revoke**.

#### Балансировщик и клиенты

- Health: **`/v1/sys/health`**, не «TCP 8200 открыт». Standby может отвечать 429/472 в зависимости от query (`standbyok` и т.д.) — это надо согласовать: либо LB шлёт только на active, либо на всех и работает forwarding.
- TLS end-to-end. Если Ingress не умеет passthrough — TCP LB / NLB. Терминировать TLS на прокси и дальше plaintext до Vault HashiCorp для прода не рекомендует.
- Клиенты (микросервисы) должны переживать короткую 5xx при смене лидера (ретраи с backoff). Это код приложений, не values Helm.

#### Масштаб и производительность

- Сначала вертикаль лидера (диск IOPS, RAM по таблице small/large), не «добавим ещё 10 Vault-подов».
- Метрики и нагрузочный тест **вашего** профиля (login + KV + transit) на препроде. Официального «N RPS на 2 ядра» под вашу систему нет.
- Пик: массовый рестарт подов с Injector (тысячи sidecar одновременно логинятся) — отдельный сценарий, его надо прогнать.
- Resource quotas Vault (ограничение lease/rate) — защита от DoS своих же клиентов; часть лимитов — Enterprise. Не замена ёмкости.
- Терабайты озера на CPU Vault не давят, пока вы не читаете их *через* transit побайтово без кэша на стороне приложения.

#### Безопасность (минимум hardening, который относится к настройке экземпляра)

- Не root в контейнере (официальный образ — пользователь `vault`).
- End-to-end TLS, по возможности TLS 1.3; HSTS через custom response headers.
- Firewall / NetworkPolicy: клиенты → 8200; узлы Vault ↔ 8200/8201; исходящие только куда надо (БД для database engine, IdP, KMS, syslog).
- Два audit device (file на отдельном томе + socket/syslog в SIEM). Мониторинг места на томе audit: **100% = Vault может перестать отвечать**.
- Audit включить **сразу после init**, до боевых секретов. По умолчанию на новом кластере audit **выключен**.
- Короткие TTL токенов и динамических паролей; least privilege, без широких `*` в политиках.
- User lockout для userpass/approle/ldap включён по умолчанию — сверить порог с ИБ.
- UI не в интернет. Лучше выключен в проде или только из admin-сети.
- Снимки шифрованы, лежат **не** только на тех же дисках кластера. Restore прогонять. Community: автоматизации вендора нет — нужен свой CronJob/оператор с правами snapshot.
- Injector: **≥ 2 реплики** webhook (в чарте `injector.replicas`; >1 требует Vault K8s 0.7.0+, в текущем чарте это давно так). Иначе выкат Injector = нельзя создать новые поды с аннотациями.
- VSO: включать только вместе с encryption-at-rest etcd и жёстким RBAC на Secret. Иначе обошли Vault, читая API Kubernetes.
- Не хранить cloud/HSM PIN в git и в ConfigMap.

#### Порядок вывода в прод (этапы, не команда за командой)

1. Измерить RTT между ЦОДами; зафиксировать один/три Kubernetes; выбрать C / D / E.
2. Выделить node pool (swap off), StorageClass, PKI, решение по seal.
3. Helm 0.34.1, pin 2.0.4, HA Raft 5 (или 3 с принятым риском), TLS, PVC data+audit.
4. Init → раздача ключей → auth + политики → revoke root.
5. Два audit, метрики, snapshot во второй ЦОД/S3-совместимое хранилище.
6. Kubernetes auth + одна боевая роль на препрод-поде; проверка Injector.
7. Учения: kill leader, drain ноды, недоступность одной зоны, диск audit 100%, restore snapshot на чистый кластер.
8. Только потом — подключать Camunda/Kafka/интеграции и ротацию паролей БД.

Без пунктов 1, 5 и 7 у вас нет секретницы на 3 ЦОДа, есть StatefulSet.

---

## Сводка: что считать «экземпляр готов»

| Требование | Тест (1 ЦОД) | Прод (3 ЦОДа) |
|---|---|---|
| Отказоустойчивость | 3 Raft, не standalone; увидеть смену лидера | 5 voter (или явно 3); раскладка по зонам с живым кворумом при потере **1** ЦОДа **или** честный SPOF + снимки; auto-unseal; Autopilot; учение restore |
| Производительность / масштаб | Не требуется | Диск/RAM от small/large и **свой** нагрузочный профиль; не ждать ускорения записи от лишних узлов; пик Injector; Enterprise — если нужны локальные чтения по ЦОДам |
| Безопасность | Shamir и HTTP только в изоляции | TLS e2e; два audit; revoke root; короткие TTL; NetworkPolicy; секреты не в git; UI закрыт; pin 2.0.4 |

**Не готов к проду**, если: дефолтный standalone/file; `latest`; PLAIN listener на общей сети; один узел Raft; Shamir без runbook на K8s; один audit на том же диске, что OS; нет снимков вне кластера; stretch при неизвестном RTT; ждут, что Community «сам» реплицируется в три ЦОДа как Enterprise; кладут озеро клиентов в KV; считают, что 5 подов Vault ускорят запись.

---

## Источники (чтобы не принимать на веру)

- Релиз 2.0.4: https://github.com/hashicorp/vault/releases/tag/v2.0.4  
- Helm-чарт 0.34.1 = app 2.0.4, предупреждение про standalone: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm  
- HA + Raft в чарте (init, join, 3 пода в примере): https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft  
- Kubernetes deployment guide (5 узлов, anti-affinity, TLS, dedicated cluster, Helm не оперирует за вас): https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide  
- Режимы чарта (dev/standalone/ha), auto-unseal KMS, production checklist: https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/run  
- Reference architecture Raft: 5 узлов × 3 AZ, **RTT < 8 ms**, порты 8200/8201, `/v1/sys/health`, small/large sizing, запись не масштабируется числом узлов: https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture  
- Integrated Storage, кворум, Autopilot, «≥ 5 для production / failure tolerance 2»: https://developer.hashicorp.com/vault/docs/internals/integrated-storage  
- Production hardening (TLS, swap, root token, audit, mlock/Raft): https://developer.hashicorp.com/vault/docs/concepts/production-hardening  
- Vault на Kubernetes, mlock, disable_mlock в чарте, swap: https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-security-concerns  
- `disable_mlock` обязателен к явному значению при Raft: https://developer.hashicorp.com/vault/docs/configuration  
- Редакции Community vs Enterprise (PR, DR, HSM, namespaces, Cloud KMS): https://developer.hashicorp.com/vault/tutorials/get-started/available-editions  
- Репликация Enterprise: https://developer.hashicorp.com/vault/docs/enterprise/replication  
- Performance standby: https://developer.hashicorp.com/vault/docs/enterprise/performance-standby  
- PKCS#11 auto-unseal = Enterprise: https://developer.hashicorp.com/vault/docs/configuration/seal/pkcs11  
- Transit auto-unseal (on-prem без HSM): https://developer.hashicorp.com/vault/docs/configuration/seal/transit-best-practices  
- Снимки: Community save/restore; автоснимки = Enterprise: https://developer.hashicorp.com/vault/docs/commands/operator/raft и https://developer.hashicorp.com/vault/docs/sysadmin/snapshots/automate  
- Audit: отказ обслуживать, если нельзя записать ни на одно устройство; рекомендация ≥ 2: https://developer.hashicorp.com/vault/docs/audit  
- Параметры чарта (ha.replicas дефолт 3, injector, CSI): https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/configuration  

Утверждения про «Vault Community переживёт два ЦОДа» или «N RPS хватит на 30 интеграций» в документации HashiCorp **отсутствуют** — поэтому в этом файле их нет. Порог **8 мс** между зонами — есть, и он здесь назван; вашего замера RTT в контексте нет.
