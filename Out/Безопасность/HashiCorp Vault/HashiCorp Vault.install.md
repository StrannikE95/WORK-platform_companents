# HashiCorp Vault 2.0.4 — установка (учебный контур)

**Допущение:** закрытый учебный Kubernetes одной площадки, Community **2.0.4**, Helm-чарт **hashicorp/vault 0.34.1**, HA + Raft на **3** подах. Это не бой и не `vault server -dev`.

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Учебный **Kubernetes** + Helm-чарт **0.34.1** (официальный путь на Kubernetes). **Helm** — программа, которая по шаблону (**чарт**) ставит набор объектов в кластер. **Под** — минимальная единица запуска в Kubernetes: контейнеры живут и умирают вместе.

Не ставим: Linux-VM «голым» бинарником, Docker Compose, `vault server -dev` (память, всё открыто, root на экране — не этот контур), дефолт чарта (один сервер + файловый бэкенд — Security Warning HashiCorp). Consul как хранилище не ставим.

Чарт задаёт anti-affinity: отдельная нода на каждый под. Три пода → лучше **три** ноды. На одной машине (kind) устойчивость не доказать; планировщик может вообще не разместить три пода.

Живой Raft **не** растягивать на 2–3 дата-центра: порты **8200/8201** между залами как «один кластер» не открывать. Community **не** реплицирует кластер на другой зал. Вашего замера RTT нет; порог HashiCorp «меньше 8 мс» — между зонами облака, не лицензия на stretch.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум, чтобы процесс поднялся | **в доке нет** | **в доке нет** | PVC чарта по умолчанию **10Gi** (Raft `path` = `/vault/data`) | чарт `resources: {}` — requests/limits не ставит; закомментированный пример чарта 256Mi/250m — не гарантия |
| Учебный ориентир HashiCorp *small* | **2–4** ядра | **8–16 ГБ** | **100+ ГБ**, SSD, от 3000 IOPS / 75 МБ/с | [reference architecture](https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture); это *starting point*, не расчёт ваших запросов |
| Пример в Kubernetes-гайде (small) | requests **2000m**, limit **2000m** | requests **8Gi**, limit **16Gi** | PVC отдельно | [kubernetes-raft-deployment-guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide) — пример, не смета этой платформы |

Нагрузки в запросе платформы нет — **не** обещать «хватит N ядер» и не считать small вашей сметой. Нагрузка Vault — число запросов к секретам, не объём озера.

**Сильная сторона:** официальный пример HA-with-raft, видны печать, кворум и смена лидера.  
**Слабая сторона:** нет auto-unseal, нет TLS, нет отказа зала, нет ёмкости. Успех трёх подов **не** доказывает бой.

**Критично:**

- **8200** (API) и **8201** (Raft / пересылка) в интернет не публиковать.
- NFS как единственный диск Raft: прямой фразы «NFS запрещён» на странице `storage/raft` **нет**. Есть: данные Integrated Storage на **той же машине**, что процесс; диск — SSD (магнитный — штраф); PVC чарта — **ReadWriteOnce** на под. Общую NFS/RWX-шару как `path` Raft не ставим. NFS в validated designs — приёмник **снимков**, не каталог Raft.
- Тег **`latest` не ставить**. Пинить `hashicorp/vault:2.0.4` (лучше digest).
- Stretch на несколько ЦОД — нет.
- Доли Shamir и root-токен **не** в git, чат, ConfigMap.

## Установка для новичка

Официальные страницы шагов: [Helm chart](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm), [HA + Raft](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft), [Run Vault on Kubernetes](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/run).

Режим **`vault server -dev` в эту инструкцию не входит.** Дефолт `helm install` без values = standalone + file — для «как будем жить» **нет**.

### Что должно быть до установки

Есть:

- Kubernetes. Вендор тестирует чарт на **1.32–1.36**; Helm **3.6+** (Helm 2 несовместим).
- Закрытая сеть; **8200/8201** не торчат в интернет.
- **kubectl** — программа, которая говорит API Kubernetes «создай объект / выполни команду в поде».
- Свободное пространство имён (ниже — `vault`).
- Три ноды желательны из-за anti-affinity.

Нет (и не нужно для этого стенда): лицензия Enterprise, облачный KMS, Consul, отдельный CA на 8200.

Чарт **не** делает за вас `init`, распечатку, бэкап и обновление.

### Этапы

**1. Репозиторий чарта**

**Что делаем:** подключаем каталог чартов HashiCorp и проверяем, что виден **0.34.1 / приложение 2.0.4**.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm search repo hashicorp/vault
```

Успех: в таблице `hashicorp/vault`, **CHART VERSION 0.34.1**, **APP VERSION 2.0.4**.

**2. Пространство имён и values**

**Что делаем:** создаём namespace и файл, который включает HA + Raft (встроенное хранилище: журнал состояния копируется между подами по протоколу Raft) и пинит образ.

```bash
kubectl create namespace vault
```

```yaml
# vault-ha.yaml — только стенд, не бой
server:
  image:
    repository: hashicorp/vault
    tag: "2.0.4"
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
```

Успех: файл на диске, тег **2.0.4**, не `latest`. В дефолтном `server.ha.raft.config` чарта: listener **8200/8201**, `tls_disable = 1`, `storage "raft" { path = "/vault/data" }`. Чарт сам дописывает `disable_mlock = true` (для Raft это обязательно задать явно; `mlock` плохо сочетается с BoltDB).

**3. Установка чарта**

**Что делаем:** ставим релиз `vault` версии **0.34.1**. HashiCorp просит сначала `--dry-run`, чтобы увидеть манифесты.

```bash
helm install vault hashicorp/vault --version 0.34.1 -n vault \
  -f ./vault-ha.yaml --dry-run
helm install vault hashicorp/vault --version 0.34.1 -n vault \
  -f ./vault-ha.yaml
kubectl -n vault get pods -l app.kubernetes.io/name=vault
```

Успех: три пода `vault-0`…`vault-2` в `Running`. Пока запечатаны — **READY 0/1**. Это норма: после старта файлы на диске есть, мастер-ключа в памяти нет, API почти не обслуживает (**sealed**).

**4. Init один раз**

**Что делаем:** готовим пустое хранилище. Команда создаёт ключи, режет мастер-ключ на доли **Shamir** (по умолчанию **5** долей, порог **3**) и выдаёт **Initial Root Token**. Делают **один раз** на первом поде.

```bash
kubectl -n vault exec -it vault-0 -- vault operator init
```

Успех: пять Unseal Key, один Initial Root Token, кластер `Initialized`. Доли и токен — людям в сейф, **не** в git. Повторный `init` на уже инициализированном кластере вендор запрещает.

**5. Распечатать vault-0, присоединить остальных**

**Что делаем:** на **каждом** поде вводим любые **3 из 5** долей (`vault operator unseal` без ключа в argv — ключ попадёт в историю shell). Затем `vault-1` и `vault-2` входят в Raft через внутренний DNS чарта.

```bash
kubectl -n vault exec -it vault-0 -- vault operator unseal
# повторить до порога 3; Sealed: false

kubectl -n vault exec -it vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl -n vault exec -it vault-1 -- vault operator unseal
# порог 3

kubectl -n vault exec -it vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl -n vault exec -it vault-2 -- vault operator unseal
```

При Shamir join **не** снимает печать: распечатка после join обязательна. Ключ аргументом команды вендор не рекомендует.

Успех: все три пода READY **1/1**.

**6. Проверка «стенд живой»**

```bash
kubectl -n vault exec -it vault-0 -- vault login
kubectl -n vault exec -it vault-0 -- vault status
kubectl -n vault exec -it vault-0 -- vault operator raft list-peers
```

Успех в `status`: **Version 2.0.4**, `Storage Type raft` (не file, не inmem), `HA Enabled true`, `Sealed false`. В `list-peers` — три узла, один `leader`, адрес вида `vault-N.vault-internal:8201`.

Запись KV (движок ключ-значение; на не-dev сервере KV v2 **не** включён сам):

```bash
kubectl -n vault exec -it vault-0 -- vault secrets enable -path=secret kv-v2
kubectl -n vault exec -it vault-0 -- vault kv put -mount=secret creds passcode=stand-only
kubectl -n vault exec -it vault-0 -- vault kv get -mount=secret creds
```

Успех: читается то же значение. Это **стендовый** секрет, не боевой пароль.

Health балансировщика (не «TCP открыт»): `GET /v1/sys/health` — **200** active, **429** standby, **503** sealed, **501** не init.

### Чего этот стенд не доказывает

Отказ дата-центра, stretch, пик Injector при массовом рестарте, TLS до клиента, auto-unseal, журнал audit (после init он **выключен**; если включить и некуда писать — Vault **отказывается** обслуживать), восстановление из снимка, нагрузка, что 3 пода ускоряют запись (запись у одного лидера). Community не даёт репликацию на другой зал.

## Первый запуск — URL, порт, учётка, смена пароля

Логина/пароля «из коробки» **нет**. После `init` — только **root token** (префикс с Vault 1.10: `hvs.…`). Это абсолютный ключ. Помечать: **только закрытый стенд**.

| Что | Значение на этом стенде |
|---|---|
| API / UI | **TCP 8200**. В конфиге чарта `ui = true`, отдельный Service UI выключен (`ui.enabled: false`). UI — тот же listener, путь `/ui` |
| Кластер Raft | **TCP 8201** — только между подами, клиентам не открывать |
| С стенда (ваша машина) | `kubectl -n vault port-forward vault-0 8200:8200` → **http://127.0.0.1:8200** |
| Из кластера | `http://vault.vault.svc:8200` (TLS в дефолте чарта выключен — **только** изоляция) |
| Учётка | Root token с экрана `init`. Пользователя `admin` / пароля `admin` **нет** |
| Печать | 5 долей Shamir, порог 3; после рестарта **каждого** пода — снова unseal |

Вход CLI после port-forward:

```bash
export VAULT_ADDR=http://127.0.0.1:8200
vault login
# вставить root token; не логировать его
```

**Смена / отзыв (стенд):** root «сменить пароль» нельзя — токен отзывают и при необходимости выпускают новый.

- Отозвать текущий: `vault token revoke <token>` или `vault token revoke -self`. После настройки Kubernetes-auth / политик HashiCorp требует отозвать начальный root.
- Новый root, если старый отозван: `vault operator generate-root` (нужен кворум **unseal**-долей и OTP/PGP). Не путать с ежедневным входом приложений.
- Новые unseal-доли: `vault operator rekey -init` (кластер должен быть распечатан; кворум старых долей). Старые доли после успешного rekey недействительны.

Доли и токены **не** класть в git. На стенде не держать root в приложении.

## Подключение к своей системе

| | |
|---|---|
| Протокол и порт | HTTP(S) API на **8200** (`/v1/...`). Заголовок `X-Vault-Token`. **8201** клиентам не нужен. На этом стенде TLS выключен; в бою HashiCorp требует TLS end-to-end, не обрывать TLS на Ingress и дальше plaintext |
| Кто клиент в платформе | Микросервисы, воркеры **Camunda**, **интеграционное API**. Шина — **Kafka**; эталон карточек — озеро/СУБД, не Vault. Клиент доказывает, кто он: **Kubernetes auth** (JWT сервисного аккаунта пода) или **AppRole** (машины вне Kubernetes). Root приложениям не раздавать |
| Что в секрет (не git) | Пароль БД, ключ API, сертификат, `role_id`/`secret_id` AppRole, CA listener. Не ConfigMap, не манифест Helm в репозитории |
| Что продукт **не** | Не озеро клиентских карточек, не Kafka, не Camunda, не WAF/Falco/NetworkPolicy, не Keycloak (люди и токены входа). Kubernetes Secret — не замена: base64 ≠ шифр Vault. Три независимых Community-кластера = **три** правды секретов. Падение Vault само по себе **не** убивает под приложения |

Типичный поток: login JWT → короткий токен → `kv get` / database / pki / transit по политике минимума путей. Sidecar Injector не обязателен. Терабайты озера в KV не класть.

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз **2.0.4** (4 августа 2026) | https://github.com/hashicorp/vault/releases/tag/v2.0.4 |
| Чарт **0.34.1**, app **2.0.4**, Helm 3.6+, k8s 1.32–1.36, Security Warning standalone | https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm |
| Команды HA + Raft: init, unseal, join, list-peers | https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-with-raft |
| `helm install --version`, init/unseal, port-forward UI, дефолт standalone | https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/run |
| `init`: доли по умолчанию 5 / порог 3, один раз | https://developer.hashicorp.com/vault/docs/commands/operator/init |
| `unseal`: ключ не в argv | https://developer.hashicorp.com/vault/docs/commands/operator/unseal |
| `status`: raft, HA, sealed | https://developer.hashicorp.com/vault/docs/commands/status |
| Root token; отозвать; generate-root | https://developer.hashicorp.com/vault/docs/concepts/tokens · https://developer.hashicorp.com/vault/docs/commands/token/revoke · https://developer.hashicorp.com/vault/docs/commands/operator/generate-root |
| Rekey долей Shamir | https://developer.hashicorp.com/vault/docs/commands/operator/rekey |
| KV v2 enable / put / get | https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2/setup · https://developer.hashicorp.com/vault/docs/commands/kv |
| Kubernetes auth (JWT пода) | https://developer.hashicorp.com/vault/docs/auth/kubernetes |
| Порты 8200/8201, small 2–4 CPU / 8–16 GB / 100+ GB, RTT &lt; 8 мс, запись не масштабируется узлами | https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture |
| Пример requests 8Gi / 2000m | https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide |
| Raft: кворум, 5 voter = tolerance 2, Autopilot | https://developer.hashicorp.com/vault/docs/internals/integrated-storage |
| `disable_mlock` обязателен явно при Integrated Storage | https://developer.hashicorp.com/vault/docs/configuration#disable_mlock · https://developer.hashicorp.com/vault/docs/configuration/storage/raft |
| Listener 8200 / cluster 8201 | https://developer.hashicorp.com/vault/docs/configuration/listener/tcp |
| `/v1/sys/health` коды | https://developer.hashicorp.com/vault/api-docs/system/health |
| Нет PR/DR в Community | https://developer.hashicorp.com/vault/tutorials/get-started/available-editions |
| Снимки Community вручную | https://developer.hashicorp.com/vault/docs/commands/operator/raft |
| Audit: нет записи → отказ API | https://developer.hashicorp.com/vault/docs/audit |
| Production hardening (TLS, не root, короткий TTL) | https://developer.hashicorp.com/vault/docs/concepts/production-hardening |
| values чарта 0.34.1 (tag 2.0.4, replicas 3, PVC 10Gi, `resources: {}`) | https://github.com/hashicorp/vault-helm/blob/v0.34.1/values.yaml |
| Правила / порты / железо | `HashiCorp Vault.md` |
| Словарь | `HashiCorp Vault.info.md` |
| Стыковка с платформой | `HashiCorp Vault.shema.md` |
| Роль консультанта | `HashiCorp Vault.consultant.md` |

**В доке вендора нет (не угадывать):** минимум CPU/RAM «чтобы контейнер просто встал»; IOPS/RPS учебного контура; RTT между вашими ЦОДами; логин/пароль по умолчанию; прямой запрет NFS одной фразой на странице Raft (есть «данные на хосте процесса» и RWO PVC).
