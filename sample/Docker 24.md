
Ресурсы:
+ Одна выделенная **64-битная Linux-VM** в закрытой сети; для зафиксированного набора пакетов — **Ubuntu 22.04 LTS (Jammy), amd64**. Ubuntu 22.04 и amd64 поддерживаются официальной инструкцией: https://docs.docker.com/engine/install/ubuntu/
+ Docker Inc. не задаёт минимальные CPU, RAM и HDD для Engine. Стартовый ориентир для небольшого учебного стенда: **2 vCPU, 4 ГиБ RAM, 30 ГБ локального SSD**; это рекомендация, не минимум вендора. Размер диска уточнять по частоте сборок и размеру образов; данные `/var/lib/docker` не разделять между демонами: https://docs.docker.com/engine/daemon/
+ Необходимое ПО для воспроизводимой установки: `docker-ce` **24.0.9**, `docker-ce-cli` **24.0.9**, `containerd.io` **1.6.28**, Buildx **0.11.2**, Compose **2.21.0** из официального Jammy pool: https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/
+ Линия **24.0** больше не сопровождается; в 24.0.9 также не закрыты известные CVE BuildKit, поэтому использовать её только для совместимого учебного стенда/сборочной VM, а не как безопасный runtime боевого Kubernetes: https://docs.docker.com/engine/release-notes/24.0/ и https://github.com/moby/moby/blob/master/project/BRANCHES-AND-TAGS.md

Открытые порты:
+ Для локального Docker Engine входящие TCP-порты **не требуются**: CLI работает через Unix-сокет `/var/run/docker.sock`; доступ к нему и членство в группе `docker` дают привилегии уровня root: https://docs.docker.com/engine/security/
+ **2375/TCP не открывать**. Если удалённый API действительно нужен, использовать SSH либо mTLS на **2376/TCP**; ключи mTLS защищать как root-пароль: https://docs.docker.com/engine/security/protect-access/
+ Порты Swarm **2377/TCP, 7946/TCP+UDP и 4789/UDP** не открывать, поскольку Swarm для этого стенда не включается: https://docs.docker.com/engine/swarm/swarm-tutorial/
+ Порты публикуемых контейнеров определяются самими приложениями; Docker предупреждает, что опубликованные порты могут обходить правила `ufw`/`firewalld`: https://docs.docker.com/engine/network/packet-filtering-firewalls/

Установка:
+ Официальный quickstart для Ubuntu: установка конкретной версии из репозитория либо `.deb`-пакетов и проверка образом `hello-world`: https://docs.docker.com/engine/install/ubuntu/
+ Базовый вводный quickstart Docker: https://docs.docker.com/get-started/
+ Для версии **24.0.9** не использовать convenience script `get.docker.com`, потому что по умолчанию он ставит актуальный stable-релиз; брать зафиксированные `.deb` из официального Jammy pool: https://docs.docker.com/engine/install/ubuntu/#install-from-a-package

Подключение:
+ Локально — Docker CLI/SDK через `/var/run/docker.sock`; удалённо — предпочтительно через SSH-контекст, без публикации API без TLS: https://docs.docker.com/engine/security/protect-access/
+ Docker Engine не является CRI для Kubernetes после удаления dockershim; боевые Kubernetes-ноды должны использовать containerd/другой CRI-runtime, а не Docker Engine 24: https://kubernetes.io/blog/2022/02/17/dockershim-faq/
