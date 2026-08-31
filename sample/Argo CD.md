
Ресурсы:
+ Один **Kubernetes**-кластер. 
	+ ПО: kubectl, kubeconfig, CoreDNS; образ **`quay.io/argoproj/argocd:v3.5.2`**, не `latest`. 
		+ https://argo-cd.readthedocs.io/en/stable/getting_started/ · https://github.com/argoproj/argo-cd/releases/tag/v3.5.2
	+ Официального **CPU/RAM/HDD нет**. Ориентир: **+2 vCPU, +4 Gb RAM, +10 Gb SSD**.
	+ Снаружи свободен **443/TCP** (UI/API/gRPC). Внутри кластера: **8080** (`argocd-server`), **8081** (repo-server), **6379** (Redis-кэш).

Установка:
+ https://argo-cd.readthedocs.io/en/stable/getting_started/

Подключение:
+ https://argo-cd.readthedocs.io/en/stable/user-guide/ci_automation/
