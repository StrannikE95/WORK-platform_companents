# Kubernetes 1.36.4 — установка и конфигурирование

Связанный документ (глоссарий, кворум etcd, RBAC/PSA, почему так): `Kubernetes.md`.

Этот файл — **как поставить и настроить**. Stretch одного Kubernetes на несколько ЦОДов **не делаем**: межЦОДовый RTT для Raft etcd неприемлем (документация etcd: члены по возможности в одном ЦОДе).

Версия: **Kubernetes 1.36.4**. Путь проекта: **kubeadm**.  
Документация: https://kubernetes.io/docs/home/  
HA kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

На дату соседнего файла тега v1.37.0 GA ещё не было — в прод не брать RC.

---

## Допущения этой инструкции

1. **По одному независимому кластеру на ЦОД.** 2 ЦОДа = 2 Kubernetes. 3 ЦОДа = 3 Kubernetes. Общего etcd нет.
2. On-prem, Linux, CRI **containerd** (CRI-O допустим; на шаги HA не влияет).
3. CNI **ещё не выбран** конкретно, но **обязан** реализовывать NetworkPolicy. Без этого объекты NetworkPolicy — декорация.
4. Нагрузки нет — нет числа worker-нод «хватит для прода».
5. Реестр образов: доступ к `registry.k8s.io` **или** заранее пролитые образы / зеркало.
6. Дистрибутивы (RKE2, Talos, OpenShift) меняют плейбук, **не** отменяют кворум etcd. Если выберете дистрибутив — те же топологические правила, другие команды.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** деплой сервисов, отладка манифестов. **Не** цель: доказать отказ площадки.

### Предпосылки

Официальный минимум kubeadm (чтобы **вообще завелось**, не прод): порядка **2 CPU и 2 ГиБ RAM** на машину, уникальные hostname / MAC / `product_uuid`. Swap — по текущим требованиям kubeadm/kubelet (не «как привыкли на Ubuntu»).

Варианты:

| Путь | Когда |
|---|---|
| kind / k3d / minikube | Только API и поды на ноутбуке |
| kubeadm: 1 control-plane (+ 1–2 worker) | Стенд ближе к проду, всё ещё один ЦОД |

Ниже — kubeadm. kind-команды зависят от локального tooling и сюда не копируются как «тот же прод».

### Установка (однонодовый kubeadm)

Порядок официальный: CRI **до** `kubeadm init`.

1. Поставить containerd, включить нужные sysctl (`ip_forward`, bridge-netfilter) — как в гайде containerd/kubeadm текущей линии 1.36.
2. Пакеты `kubelet`, `kubeadm`, `kubectl` версии **1.36.4** (пин патч, не `1.36` «какой скачается»).
3. `kubeadm config images list` — зафиксировать фактический патч образа etcd 3.6.x на этот релиз, не запоминать «навсегда».
4. Init без HA-endpoint (один API):

```bash
kubeadm init --kubernetes-version=v1.36.4
```

(CIDR pod/service — свои, не пересекающиеся потом с другими ЦОДами; на Dev достаточно дефолтов дистрибутива, если они согласованы с выбранным CNI.)

5. Сразу поставить **CNI**. Без него поды не станут Running.
6. Storage: local-path-provisioner / hostPath. Это не прод-CSI.
7. PSA: на dev-namespace профиль `baseline` уже полезен; `privileged` только системным аддонам.

`kubectl` — из `/etc/kubernetes/admin.conf` (на Dev control-plane часто ещё и worker: снять taint control-plane, иначе поды приложений не сядут).

### Конфигурирование Dev

| Параметр | На тесте | Почему можно |
|---|---|---|
| Один etcd | да | Некому строить кворум |
| Без ControlPlaneEndpoint-LB | да | Один apiserver |
| Encryption at rest | можно отложить | Изолированный стенд |
| LoadBalancer / MetalLB | опционально | NodePort хватит |

Чего **не** упрощать в манифестах приложений: `resources.requests`; запрет `privileged`/hostNetwork без нужды; секреты не в Git; StatefulSet + PVC, если в проде будет диск.

### Проверка Dev

- `kubectl get nodes` — Ready.
- CoreDNS Running.
- Тестовый Deployment встаёт и резолвит DNS внутри кластера.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, дешёво | Падение единственной ноды = всё лежит |
| Совпадает с практикой проекта (однонодовый dev допустим) | Не ловит выборы etcd, CSI WaitForFirstConsumer, истечение сертификатов на трёх API |
| | Успешный kind **не** есть готовность прода |

Препрод: **3 stacked control-plane в одном ЦОДе**, даже без боевой нагрузки.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

### Почему не stretch

Запись в etcd идёт через Raft на большинство. При высоком/плавающем RTT между ЦОДами начинаются выборы лидера, API таймаутится, **весь** кластер нестабилен. Официальная позиция etcd — по возможности один ЦОД.

### Топология

В **каждом** ЦОДе свой Kubernetes:

```
ЦОД-1: kubeadm HA (3 stacked CP или 3 CP + 3 external etcd) + workers
ЦОД-2: такой же независимый кластер (другие pod CIDR / service CIDR / PKI)
ЦОД-3: то же, если площадок три
```

Падение ЦОДа убивает **только** его API и его поды. Оставшиеся кластеры живы. Единого `kubectl` на все площадки нет — нужен GitOps (или эквивалент) и схема трафика приложений (см. документы Kafka, PostgreSQL: у них свои async/DR, не «scheduler сам унесёт PVC в чужой ЦОД»).

Четвёртый антипаттерн: etcd только в ЦОД-1, воркеры везде. Смерть ЦОД-1 = нельзя управлять кластером; рестарт подов на живых площадках не произойдёт. Это **не** HA.

### Предпосылки на каждой площадке

- Полная IP-связность **нод этого** кластера между собой (требование HA-гайда kubeadm). МежЦОДовая связность — для приложений/репликации, не для etcd.
- **Балансировщик TCP :6443** перед несколькими kube-apiserver **этого** ЦОДа (keepalived/HAProxy на двух машинах той же площадки). Health check — TCP до 6443. Один VIP в чужом ЦОДе = скрытый SPOF **этого** кластера.
- DNS-имя = `ControlPlaneEndpoint`.
- Диск etcd: локальный SSD/NVMe. **Не NFS.**
- NTP/chrony. Рассинхрон ломает TLS и таймауты Raft.
- Образы 1.36.4 с зеркала.

### Установка HA kubeadm (повторить в каждом ЦОДе)

Официальный порядок: сначала LB, потом первый control-plane, потом join остальных **по одному**.

1. Поднять LB :6443 (см. `HAProxy.install.md`, если берёте HAProxy).
2. На нодах: containerd, kubeadm/kubelet/kubectl **1.36.4**.
3. Первый control-plane:

```bash
kubeadm init \
  --kubernetes-version=v1.36.4 \
  --control-plane-endpoint "<dns-lb-этого-ЦОДа>:6443" \
  --upload-certs
```

4. Join второго и третьего control-plane (токен + certificate key из вывода init; не использовать один и тот же join-токен вечно).
5. Join workers.
6. **CNI** с NetworkPolicy, затем CSI, проверить CoreDNS.
7. Labels:

```text
topology.kubernetes.io/zone=<id этого ЦОДа>
```

Внутри одного ЦОДа можно разметить залы как подзоны, если они реально разные домены отказа. Без label scheduler не знает площадку.

8. Taint-пулы: `control-plane` (kubeadm так делает); отдельно `platform` (Kafka, БД), `general` (микросервисы), при необходимости `ingress` / `integration-dmz`.

Stacked 3 — допустимый старт (дефолт kubeadm, документированный минимум HA). External etcd (3+3) — когда готовы платить машинами за развязку IO API и etcd.

CIDR подов/сервисов **не пересекать** между кластерами ЦОДов: иначе потом не склеить маршрутизацию для MM2/репликации без NAT-ада.

### Конфигурирование безопасности (до боевых namespace)

Без этого кластер не считается настроенным. Все пункты — на **каждом** из 2–3 кластеров.

1. API :6443 только с jump/VPN, не в интернет.
2. TLS kubeadm не ослаблять (`insecure-skip-tls-verify` — не прод).
3. Люди — OIDC к IdP (или клиентские сертификаты). Файл `admin.conf` — не процесс на 50 человек.
4. RBAC: сервису свой ServiceAccount, только свой namespace. Не `cluster-admin` приложениям.
5. PSA: прикладные namespace `restricted` или минимум `baseline`. `privileged` — kube-system и оговорённые CSI/CNI.
6. NetworkPolicy default-deny, затем DNS + свои зависимости. Интеграционный контур к госсети — отдельно.
7. **Encryption at rest** для Secrets (`EncryptionConfiguration`) **до** боевых секретов, затем `kubectl replace` существующих Secret, чтобы перешифровать. Дефолт — plaintext в etcd.
   - Есть KMS/HSM → **KMSv2**.
   - Нет KMS → `aesgcm` локальным ключом (ключ не в Git; нужна ротация).
   - KMSv1 не использовать (deprecated с 1.28, выключен по умолчанию с 1.29).
8. Audit log apiserver.
9. Образы только со своего registry; не `:latest`.

Сертификаты kubeadm по умолчанию живут **1 год**. Процедура ротации — часть установки, не «фаза 2».

### Масштабирование (когда появятся цифры)

1. Requests ≈ факт, иначе BestEffort выкинут первыми.
2. HPA на stateless. Для Kafka/БД «просто добавь под» не создаёт реплики данных.
3. Добавлять workers **внутри каждого** ЦОДа по потребности этого кластера.
4. Следить за размером etcd (дефолт квоты **2 ГиБ**, типичный потолок обычной среды **8 ГиБ**) и latency API. В etcd только метаданные кластера, не озеро.

### GitOps между ЦОДами

Один и тот же desired state на 2–3 кластера **не** означает одни и те же PVC и одни и те же лидеры Kafka. В values/overlay: endpoint БД, bootstrap Kafka, replica vs primary. Без этого конфиги разъедутся за месяц — это слабое место схемы «три острова».

### Проверка прода

На **каждом** кластере:

1. Отказ одной CP-ноды: API через LB жив, `kubectl` работает.
2. Drain одной worker-ноды: поды с PDB уезжают, etcd не на NFS.
3. Снимок etcd восстановлен на стенде хотя бы раз.
4. NetworkPolicy реально режет (не только объект в API).
5. Secret в etcd не plaintext (после включения encryption).

МежЦОДовый прогон: выключить **целый** кластер ЦОД-1 на препроде и проверить, что приложения в ЦОД-2 делают то, что вы им обещали (чтение, DR, простой записи) — это уже контракт приложений, не kube-scheduler.

### Сильные / слабые стороны (остров на ЦОД)

| Сильное | Слабое |
|---|---|
| etcd локальный, предсказуемый | 2–3 upgrade, 2–3 PKI, 2–3 набора аддонов |
| Blast radius ограничен ЦОДом | Нет единого API; приложение само активно на нескольких площадках |
| Согласовано с гайдом etcd и с запретом stretch | Без GitOps конфиги разъедутся |
| Локальный LB apiserver проще | Stretch-Kafka **поверх** трёх кластеров всё равно нельзя: у Kafka свой MM2 (см. его install) |

**Не готов к проду**, если: один etcd «в уме на три ЦОДа», PLAIN Secrets, CNI без NetworkPolicy при обещании изоляции, все реплики приложений без zone labels внутри ЦОДа, cluster-admin у приложений, NFS под etcd, нет процедуры сертификатов.

---

## Источники

- Релиз 1.36.4: https://github.com/kubernetes/kubernetes/releases/tag/v1.36.4
- HA kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- Skew, запрет skip minor: https://kubernetes.io/docs/releases/version-skew-policy/
- etcd hardware / «один ЦОД когда возможно»: https://etcd.io/docs/v3.6/op-guide/hardware/
- Encryption Secrets: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- Правила и пробелы: `Kubernetes.md`

Порога RTT «для здорового etcd» в документации **проекта Kubernetes нет**. Дефолты heartbeat/election etcd — в доке etcd; в эту инструкцию они не подставляются как замер вашей сети.
