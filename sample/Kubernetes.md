
Ресурсы:
+ Linux-VM с Debian/Ubuntu/RHEL-подобная ОС
    + https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
    + Kubernetes, `kubeadm`, `kubelet` и `kubectl` — **1.36.4**; CRI — `containerd`, до запуска нужен совместимый CNI: 
        + https://github.com/kubernetes/kubernetes/releases/tag/v1.36.4 
        + https://v1-36.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/
    + Официальный минимум control plane: **2 CPU**, **2 Gb RAM**. Ориентир **SSD** ~ **30 Gb**, не NFS
        + https://etcd.io/docs/v3.6/op-guide/hardware/
+ Порты
    + https://v1-36.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/

Установка:
+ Официальный quickstart kubeadm: установить CRI и пакеты, выполнить `kubeadm init --kubernetes-version=v1.36.4`, затем сразу установить CNI; без CNI нода не станет `Ready`:       
    + https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
    + https://v1-36.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Подключение:
+ -

