
Ресурсы:
+ Один **Kubernetes**-кластер в закрытой сети. Argo CD — GitOps **CD**, не CI: https://argo-cd.readthedocs.io/en/stable/
	+ ПО: `kubectl`, kubeconfig, CoreDNS; образ **`quay.io/argoproj/argocd:v3.5.2`**, не `latest` и не ветка `stable`. CLI той же версии: https://argo-cd.readthedocs.io/en/stable/getting_started/ · https://github.com/argoproj/argo-cd/releases/tag/v3.5.2
	+ Официального минимума **CPU/RAM/HDD нет** (зависит от числа Applications). Ориентир non-HA стенда поверх Kubernetes: **+2 vCPU, +4 ГБ RAM, +10 ГБ SSD**. HA-манифест требует **≥ 3 узлов**: https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/
	+ Снаружи свободен **443/TCP** (UI/API/gRPC). Внутри кластера: **8080** (`argocd-server`), **8081** (repo-server), **6379** (Redis-кэш). В интернет не публиковать: https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/

Установка:
+ https://argo-cd.readthedocs.io/en/stable/getting_started/ — манифест пинить на `v3.5.2`: https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.2/manifests/install.yaml

Подключение:
+ CI обновляет Git; Argo CD синхронизирует тот же Git в Kubernetes. `kubectl apply` из CI на те же ресурсы не использовать: https://argo-cd.readthedocs.io/en/stable/user-guide/ci_automation/
