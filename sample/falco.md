

Ресурсы:
+ Kubernetes ≥ 1.29, Linux-ноды (x86_64 или ARM64), закрытая сеть
+ На машине админа:
	+ `kubectl` installed and configured
	- `helm 3` installed and configured
	- Cluster admin privileges

Установка:
+ Falco DaemonSet: по одному поду на каждую ноду кластера (DaemonSet - контроллер Kubernetes) — потому что смотреть надо ядро этой машины.
+ https://falco.org/docs/setup/kubernetes/

Подключение:
+ 
