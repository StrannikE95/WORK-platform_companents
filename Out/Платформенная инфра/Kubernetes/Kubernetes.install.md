# Kubernetes 1.36.4 — установка (учебный контур)

Словарь: `Kubernetes.info.md`. Схема: `Kubernetes.shema.md`. Зачем и из чего состоит: `Kubernetes.md`.

**Допущение:** закрытая сеть, одна Linux-машина, vanilla Kubernetes **1.36.4**, путь **kubeadm**, CRI **containerd**. Это не бой и не kind. Один живой кластер на дата-центр; stretch etcd на 2–3 площадки здесь нет.

Релиз (20 августа 2026): https://github.com/kubernetes/kubernetes/releases/tag/v1.36.4  
Установка kubeadm 1.36: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
Создание кластера: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Живая страница kubernetes.io/docs сейчас ведёт на **1.37** и репозиторий `pkgs.k8s.io/.../v1.37/`. Команды брать со снимка **v1.36**.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна Linux-машина (VM или железо) с **glibc** (Debian/Ubuntu/RHEL и аналоги; Alpine из коробки kubeadm не рассчитан). На ней будут: **containerd** (рантайм контейнеров, сокет `/run/containerd/containerd.sock`), **kubelet** (агент ноды), **kubeadm** (сборка кластера), **kubectl** (клиент API). Windows-ноды эта инструкция не ставит.

Если цель только API и поды на ноутбуке — **kind / k3d / minikube**. Это другой продукт и другой плейбук; успех kind **не** готовность боя. **k3s / RKE2 / OpenShift** сюда не подставлять.

Платформа: **по одному независимому кластеру на дата-центр**. Учебный стенд — одна площадка, один control plane. Порога RTT для etcd в документации Kubernetes **нет** — один кластер на три ЦОДа не собираем.

**Сколько железа.** Вендор разделяет «чтобы завелось» и смету боя. Путать нельзя.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум kubeadm (control plane) | **2 CPU** | **2 ГиБ** | локальный диск; уникальные hostname / MAC / `product_uuid` | [install-kubeadm](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), [create-cluster](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) |
| Учебный контур этой инструкции | те же 2 CPU / 2 ГиБ | те же | etcd kubeadm кладёт в `/var/lib/etcd` на **локальный** диск, не NFS | то же; NFS для etcd запрещает [гайд etcd](https://etcd.io/docs/v3.6/op-guide/hardware/) |

Цифр «хватит N worker на терабайты» в доке **нет**. Терабайты — PVC / Kafka / БД, не etcd.

**Сильная сторона:** официальный однонодовый путь kubeadm, те же пакеты 1.36.4.  
**Слабая сторона:** один etcd = падение машины = кластер с нуля. Стенд не ловит выборы лидера и отказ зала.

**Критично:** порт **6443** (kube-apiserver, TLS) не с интернета. CRI — **containerd**, не `dockerd`. Пакеты пинить **1.36.4**, не `latest` и не «какой скачается из репы 1.36». Ядро с **cgroup v2** (`stat -fc %T /sys/fs/cgroup/` → `cgroup2fs`): kubelet 1.36 по умолчанию не стартует на cgroup v1 ([cgroups](https://kubernetes.io/docs/concepts/architecture/cgroups/)).

---

## Установка для новичка

Команды — **на Linux-машине стенда**, права root/`sudo`. Не PowerShell. Официальный порядок: CRI **до** `kubeadm init`; CNI **сразу после**.

Страницы шагов: [install-kubeadm](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), [container-runtimes](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/), [create-cluster](https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

### Что должно быть до установки

Есть:

- Linux с cgroup v2, NTP/chrony, выход на `registry.k8s.io` **или** зеркало образов 1.36.4.
- Уникальные `hostname`, MAC, `product_uuid` (`ip link`; `sudo cat /sys/class/dmi/id/product_uuid`).
- Свободен **6443/TCP**. На одной машине также служебные **2379–2380**, **10250**, **10257**, **10259** ([порты](https://v1-36.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/)).
- Не меньше 2 CPU и 2 ГиБ RAM.

Нет:

- `dockerd` как CRI кластера (Docker Engine CRI не реализует; нужен отдельный cri-dockerd — **не** путь этой платформы).
- Kubernetes Dashboard в поставке kubeadm (вне scope инструмента).
- Готового веб-пароля.

### Этап 1. Машина

**Что делаем:** проверяем CPU, RAM, cgroup, уникальность.

```bash
nproc
free -h
stat -fc %T /sys/fs/cgroup/
hostname
ip link
sudo cat /sys/class/dmi/id/product_uuid
```

Успех: ≥ 2 CPU, ≥ 2 ГиБ, `cgroup2fs`, hostname/MAC/uuid не пустые и не клоны соседней VM.

### Этап 2. Swap и sysctl

**Что делаем:** kubelet по умолчанию **не стартует**, если видит swap. На учёбе выключаем. Включаем IPv4 forwarding — без этого сеть подов не заработает.

```bash
sudo swapoff -a
# чтобы пережило reboot: убрать swap из /etc/fstab

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

Успех: `net.ipv4.ip_forward = 1`. Swap не виден в `swapon --show`.

### Этап 3. containerd (CRI)

**Что делаем:** ставим рантайм, которым kubelet создаёт контейнеры. **containerd** — демон; сокет CRI — `unix:///var/run/containerd/containerd.sock`. Не `dockerd`.

Постановка бинарника: [getting-started containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) (пакет `containerd.io` от Docker **или** официальный tar). Патч containerd эта инструкция **не** пинит — в карточке Kubernetes его нет.

После установки — конфиг, который требует страница Kubernetes. Пакет часто кладёт `cri` в `disabled_plugins` — CRI должен быть **включён**. На systemd — драйвер cgroup **systemd** (kubeadm с 1.22 так и ставит kubelet).

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

В `/etc/containerd/config.toml`:

- `cri` **нет** в `disabled_plugins`.
- Для containerd **1.x**: `SystemdCgroup = true` в `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`.
- Для containerd **2.x**: то же в `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]`.

```bash
sudo systemctl restart containerd
sudo systemctl enable --now containerd
sudo systemctl is-active containerd
ls /run/containerd/containerd.sock
```

Успех: сервис `active`, сокет на месте. Два CRI на одной машине — тогда в `kubeadm init` нужен `--cri-socket`; на этом стенде держим только containerd.

### Этап 4. Пакеты kubelet, kubeadm, kubectl **1.36.4**

**Что делаем:** репозиторий **линии 1.36** (`pkgs.k8s.io`), затем патч **1.36.4**, затем hold — иначе `apt upgrade` снимет другой патч. Старые `apt.kubernetes.io` / `yum.kubernetes.io` заморожены с 13 сентября 2023.

Debian/Ubuntu (на Debian < 12 и Ubuntu < 22.04 сначала `sudo mkdir -p -m 755 /etc/apt/keyrings`):

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-cache madison kubeadm
sudo apt-get install -y kubelet='1.36.4-*' kubeadm='1.36.4-*' kubectl='1.36.4-*'
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

Шаблон `1.36.4-*` — из [гайда апгрейда kubeadm](https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/): суффикс пакета смотреть в `madison`, не выдумывать `-1.1`.

RHEL/Fedora: репозиторий `https://pkgs.k8s.io/core:/stable:/v1.36/rpm/`, установка `kubelet-'1.36.4-*' kubeadm-'1.36.4-*' kubectl-'1.36.4-*'` — та же страница install-kubeadm.

Успех:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

Во всех трёх — **1.36.4**. kubelet до `kubeadm init` будет в crashloop — так пишет вендор, это нормально.

### Этап 5. Список образов control plane

**Что делаем:** фиксируем, какие образы `registry.k8s.io` уйдут в init (в том числе etcd **3.6.x**). Без сети до реестра или зеркала init не встанет.

```bash
kubeadm config images list --kubernetes-version=v1.36.4
```

Успех: список без ошибки. Закрытый контур: пролить эти образы в зеркало, init с `--image-repository=<зеркало>`.

### Этап 6. `kubeadm init`

**Что делаем:** поднимаем один control plane. Флаг `--kubernetes-version` обязателен: дефолт kubeadm — `stable-1`, это **не** 1.36.4. `--control-plane-endpoint` на одном API **не** ставим (иначе потом без LB не «дорастёте» до HA этим же кластером — так в create-cluster).

```bash
sudo kubeadm init --kubernetes-version=v1.36.4
```

Если выбранный CNI требует свой pod CIDR — добавить `--pod-network-cidr=<cidr>` **здесь**, не после. Service CIDR дефолт `10.96.0.0/12`, DNS-домен `cluster.local`. На нескольких площадках платформы CIDR **разных** кластеров не пересекают — на одном учебном хосте дефолтов достаточно, если они не бьются с CNI.

Успех: строка `Your Kubernetes control-plane has initialized successfully!` Сохранить `kubeadm join ...` из вывода (токен — секрет, TTL дефолт **24 ч**).

### Этап 7. kubeconfig для обычного пользователя

**Что делаем:** kubectl ходит в API по файлу kubeconfig. Файл с правами cluster-admin kubeadm пишет в `/etc/kubernetes/admin.conf`. Без копии в `$HOME` следующие `kubectl` не сработают.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

От root достаточно `export KUBECONFIG=/etc/kubernetes/admin.conf`.

Успех: `kubectl get nodes` отвечает (нода ещё может быть `NotReady` — нет CNI).

### Этап 8. CNI

**Что делаем:** сеть подов. Без CNI поды **не** станут Running, CoreDNS не поднимется. Kubernetes сеть «из коробки» не даёт. Ставить **один** CNI на кластер.

```bash
kubectl apply -f <add-on.yaml>
```

`<add-on.yaml>` — манифест выбранного CNI с [страницы аддонов](https://v1-36.docs.kubernetes.io/docs/concepts/cluster-administration/addons/). Конкретный продукт в карточке **не** зафиксирован; **обязан** реализовывать NetworkPolicy (иначе объект в API — декорация). Примеры из той же страницы: Calico, Cilium. Flannel NetworkPolicy не обещает — для этой платформы не брать.

Успех:

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

Нода `Ready`. CoreDNS `Running`.

### Этап 9. Снять taint control-plane

**Что делаем:** на одной машине control plane = единственная нода. По умолчанию taint `node-role.kubernetes.io/control-plane:NoSchedule` — прикладные поды не сядут.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Успех: `untainted` (или taint уже не было).

### Этап 10. Проверка «стенд живой»

**Что делаем:** Deployment + DNS внутри кластера.

```bash
kubectl create namespace dev
kubectl label namespace dev pod-security.kubernetes.io/enforce=baseline
kubectl -n dev create deployment dns-check --image=registry.k8s.io/pause:3.10 --replicas=1
kubectl -n dev rollout status deployment/dns-check
kubectl -n dev run dns-query --rm -it --restart=Never --image=busybox:1.28 -- \
  nslookup kubernetes.default.svc.cluster.local
```

Успех: под Deployment Running; `nslookup` резолвит Service `kubernetes` в namespace `default`. Версия API:

```bash
kubectl version
```

Сервер — **v1.36.4**.

Storage на учёбе: **hostPath** или local-path-provisioner. Это не боевой CSI. Готового YAML local-path в документации kubeadm **нет**.

Снесли машину — кластер пропал: состояние было в etcd на одном диске. Так и должно быть.

**Чего этот стенд не доказывает:** отказ зала, кворум etcd, ControlPlaneEndpoint/LB :6443, выборы лидера, CSI WaitForFirstConsumer, истечение сертификатов на трёх API, нагрузку, stretch. Успех kind/k3d/minikube — не этот стенд и не бой.

---

## Первый запуск — URL, порт, учётка, смена пароля

Веб-консоли kubeadm **не** ставит. Заводского пароля админки **нет**. Dashboard в scope kubeadm не входит ([kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)); проект Dashboard к тому же archived — в этот контур не ставим.

| Что | Значение |
|---|---|
| URL API | `https://<ip-машины>:6443` |
| Порт | **6443/TCP**, TLS. Health check «жив ли API» — TCP до 6443 |
| Протокол | HTTPS Kubernetes API, не HTTP-форма логина |
| Учётка | клиентский сертификат в kubeconfig, не login/password |

Файл `/etc/kubernetes/admin.conf`: сертификат `CN = kubernetes-admin`, группа `kubeadm:cluster-admins` → ClusterRole `cluster-admin`. **Не раздавать.** Рядом kubeadm пишет `super-admin.conf` (`O = system:masters`) — обходит RBAC; убрать в сейф, в повседневную работу не брать.

Проверка с ноды:

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get --raw=/readyz
```

Успех: `ok`. С чужой машины — скопировать `admin.conf` (scp), не публиковать 6443 в интернет:

```bash
scp user@<control-plane>:/etc/kubernetes/admin.conf ./admin.conf
kubectl --kubeconfig ./admin.conf get nodes
```

Сертификат в `admin.conf` должен совпадать с IP/DNS, которым вы стучитесь (SAN). Если ходите по другому имени — это не «сменить пароль», а перевыпустить сертификат API (`--apiserver-cert-extra-sans` на init или процедура certs).

**«Смена пароля».** Пароля нет. Отозвать доступ = не шарить `admin.conf` / `super-admin.conf` и выпустить **другой** kubeconfig:

```bash
kubeadm kubeconfig user --client-name <имя>
```

Дальше RoleBinding/ClusterRoleBinding. Не cluster-admin «на отдел». Сертификаты kubeadm по умолчанию живут **1 год**.

Опционально, только localhost: `kubectl proxy` → HTTP `http://127.0.0.1:8001/` (это прокси с вашей машины, не открытый API без TLS).

---

## Подключение к своей системе

Остальные продукты платформы деплоятся **в этот** Kubernetes (исключения — в их карточках: SafeLine Compose, пакеты Luxms, storage-ноды Swift, Gitaly на VM). Клиенты: **kubectl**, Helm, GitLab Agent. Приложения сами «не перевезут PVC в чужой дата-центр».

**С ноутбука / CI:** `kubectl` + kubeconfig на `https://<ip>:6443`. В git — манифесты и Helm values **без** секретов.

**Из пода:** DNS кластера — CoreDNS, домен дефолт `cluster.local`.

| Имя | Куда резолвится |
|---|---|
| `kubernetes.default.svc.cluster.local` | ClusterIP API (изнутри кластера) |
| `<svc>.<namespace>.svc.cluster.local` | Service в чужом namespace |
| `<svc>` | Service **в том же** namespace (search в `/etc/resolv.conf` пода) |

Короткое имя `data` из namespace `test` **не** найдёт Service `data` в `prod` — нужно `data.prod` или FQDN ([DNS](https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/)).

Интеграции с ведомствами — через **своё интеграционное API**, не напрямую из kube-apiserver.

### Секреты (не в git)

| Секрет | Где живёт | Куда не класть |
|---|---|---|
| `admin.conf` / копия kubeconfig | `/etc/kubernetes/admin.conf`, `$HOME/.kube/config` | git, чат, образ |
| `super-admin.conf` | `/etc/kubernetes/super-admin.conf` | повседневный kubectl, git |
| bootstrap-токен `kubeadm join` | вывод init; `kubeadm token` | git; TTL 24 ч |
| токен ServiceAccount в поде | `/var/run/secrets/kubernetes.io/serviceaccount/token` (проекция TokenRequest, живёт пока жив под) | долгоживущий Secret «навечно», если нет явной нужды |
| `kubectl create token <sa>` | stdout, TTL задаёте вы | git |

Поду — **свой** ServiceAccount и RBAC своего namespace, не `cluster-admin`. Долгоживущий SA-токен через Secret — устаревший путь; в 1.36 штатно — TokenRequest.

### Чем продукт не является

| Сосед | Чем отличается |
|---|---|
| Docker Engine 24 | Builder образов. Не CRI этого кластера |
| kind / k3d / minikube | Учебный API на ноутбуке, не этот kubeadm и не бой |
| OpenShift / RKE2 / k3s | Другой дистрибутив, другой плейбук |
| Kafka | Шина событий; RF и диски — в инструкции Kafka |
| PostgreSQL / озеро | Эталон карточек; не etcd |
| Grafana / Prometheus | Наблюдение; Kubernetes их не заменяет |
| Ingress-контроллер / MetalLB | Ядро не ставит исполнителя HTTP-входа и не выдаёт внешний IP типу LoadBalancer на железе |
| Реестр образов | Нужен свой Harbor/зеркало; `registry.k8s.io` может быть недоступен |
| GitOps (Argo CD / Flux) | Синхронизация трёх control plane сама не появляется |

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз **1.36.4** (20 августа 2026) | https://github.com/kubernetes/kubernetes/releases/tag/v1.36.4 |
| Минимум 2 CPU / 2 ГиБ, уникальные hostname/MAC/`product_uuid`, swap, репозиторий `pkgs.k8s.io` **v1.36**, hold пакетов, сокет containerd | https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ |
| `kubeadm init`, admin.conf / super-admin.conf, CNI до CoreDNS Running, taint control-plane, scp kubeconfig, один etcd = нет переживания ноды | https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/ |
| `--kubernetes-version` (дефолт `stable-1`), порт 6443, `cluster.local`, service CIDR `10.96.0.0/12` | https://v1-36.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/ |
| CRI, `net.ipv4.ip_forward`, systemd cgroup, **не** dockershim; Docker Engine ≠ CRI без cri-dockerd | https://v1-36.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/ |
| Постановка containerd | https://github.com/containerd/containerd/blob/main/docs/getting-started.md |
| Пин патча `1.36.4-*` через `apt-cache madison` | https://v1-36.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/ |
| Запрет skip minor | https://kubernetes.io/docs/releases/version-skew-policy/ |
| Порты 6443, 2379–2380, 10250, 10257, 10259 | https://v1-36.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/ |
| DNS Service/Pod, `*.svc.cluster.local` | https://v1-36.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/ |
| ServiceAccount, путь токена, TokenRequest, `kubectl create token` | https://v1-36.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/ |
| PSA, label `pod-security.kubernetes.io/enforce` | https://v1-36.docs.kubernetes.io/docs/concepts/security/pod-security-admission/ |
| Список CNI; Calico/Cilium умеют NetworkPolicy | https://v1-36.docs.kubernetes.io/docs/concepts/cluster-administration/addons/ |
| kubeadm не ставит Dashboard и прочие аддоны «для удобства» | https://kubernetes.io/docs/reference/setup-tools/kubeadm/ |
| cgroup v2; kubelet по умолчанию не стартует на cgroup v1 | https://kubernetes.io/docs/concepts/architecture/cgroups/ |
| HA kubeadm (не этот стенд; один endpoint на площадку) | https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/ |
| etcd: один ЦОД когда возможно, SSD, не NFS | https://etcd.io/docs/v3.6/op-guide/hardware/ |
| Secrets в etcd по умолчанию plaintext | https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/ |
| kind (только ноутбук) | https://kind.sigs.k8s.io/docs/user/quick-start/ |
| Зачем продукт, порты, железо | `Kubernetes.md` |
| Словарь | `Kubernetes.info.md` |
| Схема стыковки с платформой | `Kubernetes.shema.md` |
| Роль консультанта | `Kubernetes.consultant.md` |

**В доке вендора нет (и здесь не выдумано):** порог RTT «для здорового etcd»; смета worker-нод под терабайты; конкретный CNI и его YAML; патч-версия containerd; заводской веб-пароль; готовый URL Dashboard в kubeadm; NFS как диск etcd; один кластер на три дата-центра.
