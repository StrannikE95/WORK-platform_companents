
Ресурсы:
+ Закрытая Linux-машина с `glibc` (Debian/Ubuntu/RHEL-подобная ОС); конкретные версии дистрибутивов официально не зафиксированы: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
+ Kubernetes, `kubeadm`, `kubelet` и `kubectl` — **1.36.4**; CRI — `containerd` (официальная точная версия не задана), до запуска нужен совместимый CNI: https://github.com/kubernetes/kubernetes/releases/tag/v1.36.4 и https://v1-36.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/
+ Официальный минимум control plane: **2 CPU**, **2 ГиБ RAM**. Минимальная ёмкость диска не установлена; практический ориентир закрытого однонодового стенда — **30 ГБ локального SSD**, не NFS: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ и https://etcd.io/docs/v3.6/op-guide/hardware/
+ Порты control plane: **6443/TCP**, **2379–2380/TCP**, **10250/TCP**, **10257/TCP**, **10259/TCP**; worker: **10250/TCP**, **10256/TCP**, при NodePort — **30000–32767/TCP/UDP**. Порты выбранного CNI добавляются отдельно; **6443** не публиковать в интернет: https://v1-36.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/

Установка:
+ Официальный quickstart kubeadm: установить CRI и пакеты, выполнить `kubeadm init --kubernetes-version=v1.36.4`, затем сразу установить CNI; без CNI нода не станет `Ready`: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ и https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
+ Для боя использовать HA-схему минимум с тремя control-plane и балансировщиком API, а не однонодовый стенд: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

Подключение:
+ Администраторы подключаются `kubectl` по HTTPS к **6443** через kubeconfig; веб-консоль и заводской логин/пароль kubeadm не устанавливает: https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
