# Wazuh 4.14.7 — установка (учебный контур)

**Допущение:** одна 64-bit Linux-машина в закрытой сети; all-in-one (Wazuh server + indexer + dashboard на одном хосте). Бой и растяжка кластера на несколько дата-центров в этот файл не входят.

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Выделенная **Linux**-машина (x86_64 или ARM64) в **одном** дата-центре. Windows как хост server/indexer/dashboard в списке ОС вендора **нет**. Мозг SIEM (server + indexer + dashboard) на этой машине; наблюдаемые хосты — агентами сюда. Три полных Wazuh без склейки = три SIEM.

Этот путь — **пакеты Linux**, не контейнеры. **Docker** (программа, которая запускает **образ** — упакованную программу с зависимостями — как контейнер) и **kubectl** (программа, которая применяет описание подов к кластеру Kubernetes) здесь **не нужны**. Официальные пути Docker (`wazuh-docker` тег `v4.14.7`) и Kubernetes (`wazuh-kubernetes` тег `v4.14.7`, Kustomize) существуют; в этот учебный контур не входят.

**Сколько железа.** Для all-in-one вендор даёт **рекомендацию** (обычно до ~100 точек и 90 дней индекса), не отдельную строку «минимум, чтобы процесс поднялся». Цифры «хватит N ядер под ваш Kafka» в мануале **нет**.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Учебный all-in-one, 1–25 агентов, 90 дней индекса | **4 vCPU** | **8 ГиБ** | **50 ГБ** | Quickstart, recommended |
| То же, 25–50 агентов | 8 vCPU | 8 ГиБ | 100 ГБ | Quickstart |
| То же, 50–100 агентов | 8 vCPU | 8 ГиБ | 200 ГБ | Quickstart |
| Минимум all-in-one «чтобы процесс встал» | в доке вендора **нет** | нет | нет | — |
| Kubernetes «поды поднялись» (другой путь) | 2 CPU | 3 ГиБ | 2 ГиБ | kubernetes-conf; **не** смета all-in-one |
| Агент на наблюдаемой машине | — | **~35 МБ** среднее | — | страница агента |

Распределённая установка **на один узел** (не этот стенд): server минимум **2 ГиБ / 2 CPU**, рекомендовано **4 ГиБ / 8 CPU**; indexer минимум **4 ГиБ / 2 CPU**, рекомендовано **16 ГиБ / 8 CPU**. Диск indexer за 90 дней (таблица APS вендора, не ваш auditd): сервер **3.7 ГБ**, станция **1.5 ГБ**, сетевое устройство **7.4 ГБ**. Нагрузки платформы нет — сметы «терабайты влезут» нет.

Куча Java indexer: `Xms = Xmx`, ориентир **половина RAM** узла (пример: 8 ГБ → 4g). Это страница tuning, не таблица all-in-one.

**Сильная сторона:** совпадает с официальным Quickstart, одна VM, цепочка «агент → алерт → индекс → экран» за часы.  
**Слабая сторона:** одна машина = одна точка отказа всего сразу; нет копий шардов, нет worker-кластера, нет балансировщика. Успешный алерт **не** доказывает отказ зала и поток с брокерной ноды.

**Критично, даже если не спрашивали:**

- Порты **1514** (события агентов), **1515** (регистрация, `authd`), **443** (dashboard), **55000** (API server), **9200** (REST indexer) **не в интернет**. Между нодами indexer — **9300–9400**; master↔worker — **1516**. Syslog **514** по умолчанию выключен.
- Скрипт линии **4.14**, не тег `latest`, не прыжок через minor. После установки проверить пакеты **4.14.7**.
- Живой кластер indexer/server **не растягивать** на 2–3 дата-центра: порога RTT в миллисекундах у вендора **нет**.
- Учебный пароль и самоподписанный сертификат — **только закрытый стенд**.

ОС центральных ролей (вендор): Amazon Linux 2/2023, CentOS Stream 10, RHEL 7–10, Ubuntu 16.04–24.04.

## Установка для новичка

Команды — **на Linux-машине стенда**, с правами root. Официальная страница шагов: https://documentation.wazuh.com/current/quickstart.html

Выбран **один** путь: all-in-one. Kubernetes `envs/local-env` и Docker Compose в этих этапах нет.

### Что должно быть до установки

Есть:

- 64-bit Linux x86_64 или ARM64 из списка выше; интернет до `packages.wazuh.com` (закрытый контур — другой гайд вендора, offline).
- Закрытая сеть; вход с jump-хоста или VPN.
- Ориентир **4 vCPU / 8 ГиБ / 50 ГБ** свободных (таблица 1–25 агентов).
- Свободны на хосте **443**, **1514**, **1515**, **55000**, **9200**.

Нет (и не должно появиться):

- Публикация 1514/1515/443/55000/9200 в интернет.
- Второй master «для устойчивости» (штатно нельзя).
- Смесь пакетов 4.13 и 4.14.
- Тег образа `latest`.

### Этап 1. Машина подходит

**Что делаем:** проверяем архитектуру и запас RAM/диска.

```bash
uname -m
free -h
df -h
```

Успех: `x86_64` или `aarch64`; RAM и диск не меньше строки «1–25 агентов», если учитесь на этой таблице.

### Этап 2. Ставим центральные роли скриптом вендора

**Что делаем:** скачиваем installation assistant линии **4.14** и ставим server + indexer + dashboard на **этот** хост (`-a` = all-in-one). Скрипт ставит пакеты из репозитория 4.14; это **не** pin `4.14.7` в URL. Сразу после — сверить версию.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Успех: в конце есть блок Summary: URL `https://<WAZUH_DASHBOARD_IP_ADDRESS>`, пользователь `admin`, пароль `<ADMIN_PASSWORD>`. Сохраните пароль: он сгенерирован, не строка `admin`/`admin`.

Все пароли indexer и API:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Архив лежит в каталоге, откуда запускали скрипт. Файл **не** в git.

Снять установку (если нужно): `sudo bash ./wazuh-install.sh -u`.

### Этап 3. Версия 4.14.7 и службы

**Что делаем:** подтверждаем пакеты и что процессы живы.

Debian/Ubuntu:

```bash
dpkg -l wazuh-manager wazuh-indexer wazuh-dashboard
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

RHEL/семейство yum:

```bash
rpm -q wazuh-manager wazuh-indexer wazuh-dashboard
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

Indexer отвечает на REST **9200** (подставьте пароль `admin` из Summary):

```bash
curl -k -u admin:'<ADMIN_PASSWORD>' https://127.0.0.1:9200
```

Успех: в пакетах **4.14.7**; три службы `active`; `curl` возвращает JSON кластера, не ошибку TLS/401 как тупик. Если скрипт поставил другую 4.14.x — в Quickstart **нет** флага «поставить ровно 4.14.7»; не продолжать «как есть» и не подставлять `latest`.

На каждом server в поставке есть **Filebeat**: читает JSON-алерты с диска manager и отдаёт в indexer **9200**. Без него dashboard алертов не увидит.

### Этап 4. Выключить репозиторий пакетов

**Что делаем:** случайный `yum update` / `apt upgrade` не должен разъехать версии. Совет Quickstart.

RPM:

```bash
sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo
```

Debian/Ubuntu (тот же приём, страница агента — тот же файл `wazuh.list`):

```bash
sed -i "s/^deb /#deb/" /etc/apt/sources.list.d/wazuh.list
apt-get update
```

Успех: репозиторий не отдаёт пакеты при обычном обновлении системы.

### Стенд живой — до браузера

1. Пакеты manager, indexer, dashboard = **4.14.7**.
2. `curl` на `https://127.0.0.1:9200` от `admin` проходит.
3. Рестарт единственного indexer: поиск дырявится. На одном узле без копий шардов так и должно быть (завод шаблона: 3 primary, **0 replicas**).

Чего этот стенд **ещё не доказывает:** отказ дата-центра, выборы лидера indexer при потере площадки (на одной машине нечему голосовать), нагрузку и `events_dropped` / `discarded_count`, балансировщик на 1514, второго worker, копии шардов, сценарий смерти единственного master.

## Первый запуск — URL, порт, учётка, смена пароля

**Только закрытый стенд.** Эти учётки в бой не копируют.

**URL и порт.** Браузер: `https://<IP-машины-стенда>/`  
Порт **443** (таблица архитектуры; в URL Quickstart порт не пишут). Открывайте **с jump-хоста или VPN**, не выставляйте 443 в интернет. Сертификат самоподписанный: браузер покажет предупреждение — на учебном стенде это ожидаемо (так пишет Quickstart).

**Учётка dashboard.** Пользователь **`admin`**. Пароль — из Summary установщика, не фиксированная строка. Повторно:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

`admin` — администратор **indexer**; им входят в dashboard. API server — отдельные пользователи `wazuh` и `wazuh-wui` (пароли в том же файле). Смена `wazuh-wui` без правки конфига dashboard **ломает** связь UI с API.

Не путать с другими поставками (этот файл их не ставит): Docker и overlay Kubernetes v4.14.7 публикуют `admin` / `SecretPassword`; overlay ещё `kibanaserver` / `kibanaserver`, API `wazuh-wui` / `MyS3cr37P450r.*-`, пароль регистрации агентов `password`, ключ кластера `123a45bc67def891gh23i45jk67l8mn9`. Это известный GitHub, не пароль all-in-one.

**Смена пароля.** Инструмент вендора (на all-in-one сам обновляет связанные конфиги). Пароль: 8–64 символа, есть заглавная, строчная, цифра и символ из набора `.*+?-`.

Скачать копию скрипта линии 4.14 или вызвать встроенный:

```bash
curl -so wazuh-passwords-tool.sh https://packages.wazuh.com/4.14/wazuh-passwords-tool.sh
sudo bash wazuh-passwords-tool.sh -u admin -p '<НОВЫЙ_ПАРОЛЬ>'
```

Сменить все учётки indexer и API (подставьте текущий пароль API-админа `wazuh` из `wazuh-passwords.txt`):

```bash
sudo bash wazuh-passwords-tool.sh -a -A -au wazuh -ap '<ТЕКУЩИЙ_ПАРОЛЬ_wazuh>'
```

Новый пароль — в сейф / Vault, не в git и не в чат. Учебный пароль из архива установщика в бой не переносят. В бою — свои секреты; единый вход (SSO) в 4.14 — отдельная настройка dashboard/indexer, не шаг этого стенда.

## Подключение к своей системе

Приложения **не** ходят в Wazuh как в HTTP-API своего бэкенда и **не** пишут в indexer (порт **9200**) как в платформенный OpenSearch. Клиент — **агент** на наблюдаемой машине (или syslog на **514**, коллектор по умолчанию **выключен**).

Официальная установка агента: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html  
Мастер в UI: **Agents management → Summary → Deploy new agent**.

### Протокол

| Что | Как |
|---|---|
| События уже зарегистрированного агента | **TCP 1514**, шифрование **AES** (блок 128, ключ 256 бит). UDP 1514 по умолчанию выключен. Это не TLS |
| Первичная регистрация (`authd`) | **TCP 1515**, живёт на **master**. На all-in-one master = эта же машина |
| API управления | HTTPS **55000** — dashboard и операторы, не мир |
| Поиск алертов | indexer REST **9200** — Filebeat и dashboard, не приложения платформы |

В бою перед 1514 (и обычно 1515) вендор рекомендует **TCP-балансировщик**; агент знает один VIP. На одном учебном хосте балансировщик не нужен.

### Кто клиент

- **Агент 4.14.7** на Linux/Windows/macOS (пакет) или DaemonSet в Kubernetes (имя = нода, иначе ноды сливаются). Менеджер должен быть **не старше агента наоборот**: совместимость, когда версия manager ≥ версии агента.
- Тот же учебный хост может нести агента — так быстрее увидеть цепочку.
- Официальный Docker-агент — syslog-коллектор; **без** каталогов хоста (`/var/log`, `/proc`) хост не мониторит.
- Kafka, Camunda, озеро, интеграционное API — **источники логов и поверхность FIM**, не клиенты indexer. Без агента (или syslog) SIEM на этих машинах слепой.
- Приложения **не** подключаются к Wazuh indexer вместо платформенного OpenSearch.

Пример Linux (APT), IP — адрес машины all-in-one. На all-in-one заводской `<use_password>no</use_password>`: пароль регистрации **не** обязателен, пока вы сами не включите `authd`.

```bash
apt-get install gnupg apt-transport-https
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list
apt-get update
WAZUH_MANAGER="<IP_ALL_IN_ONE>" apt-get install wazuh-agent
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
sed -i "s/^deb /#deb/" /etc/apt/sources.list.d/wazuh.list
apt-get update
```

Успех: агент **Active** в **Agents management → Summary**. Потом измените файл из FIM (или пришлите известный лог) — документ в индексе `wazuh-alerts-*` и строка на экране.

Если включите пароль регистрации: файл `/var/ossec/etc/authd.pass` на manager, `chmod 640`, владелец `root:wazuh`, в `ossec.conf` `<use_password>yes</use_password>`, переменная агента `WAZUH_REGISTRATION_PASSWORD`. 1515 не с мира. В overlay Kubernetes пароль по умолчанию — известная строка `password`; на all-in-one её нет, пока не зададите сами.

### Что класть в секрет (не в git)

| Секрет | Где на этом стенде | Куда не класть |
|---|---|---|
| Пароль `admin` (и остальные из `wazuh-passwords.txt`) | архив установщика + сейф после смены | git, чат, образ |
| Пароль регистрации агентов (если включите) | `/var/ossec/etc/authd.pass` | git |
| Ключ кластера server (ровно 32 символа) | нужен при нескольких узлах server; на одном all-in-one кластера нет | публичный манифест GitHub |
| Сертификаты | самоподписанные скрипта — только стенд | привычка игнорировать предупреждение браузера в бою |

В git — процедура без паролей. Каталог manager `/var/ossec` (ключи агентов) — состояние на диске; потеря = заново регистрировать флот.

### Чем Wazuh не является

Не WAF, не Falco, не NetworkPolicy. Не шина событий и не Camunda. Не платформенный OpenSearch. Не два активных master. Active Response на интеграционном контуре **выключен** (завод в эталоне — выкл). Поиск уязвимостей через CTI вендора на закрытом контуре можно не включать: без исходящего доступа модуль как в гайде не работает.

## Ссылки на материал

Факты ниже — с **указанной** страницы, не «из документации вообще».

| Факт | Страница |
|---|---|
| Релиз **4.14.7** (29 июля 2026) | https://documentation.wazuh.com/current/release-notes/release-4-14-7.html |
| Порты 1514/1515/1516/514/55000/9200/9300–9400/443; AES; Filebeat→9200 | https://documentation.wazuh.com/current/getting-started/architecture.html |
| All-in-one команда `-a`, таблица 4/8/50 … 8/8/200, ОС, учётка `admin`, `wazuh-passwords.txt`, выключить yum-репозиторий, самоподписанный сертификат | https://documentation.wazuh.com/current/quickstart.html |
| Server: min/rec CPU·RAM, диск APS, `events_dropped` / `discarded_count` | https://documentation.wazuh.com/current/installation-guide/wazuh-server/index.html |
| Indexer: min/rec CPU·RAM, диск 3.7/1.5/7.4 ГБ за 90 дней | https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/index.html |
| Один master; `ossec.conf` на worker сам не едет | https://documentation.wazuh.com/current/user-manual/wazuh-server-cluster/types-of-nodes.html |
| Агенты: список адресов vs балансировщик | https://documentation.wazuh.com/current/user-manual/wazuh-server-cluster/agent-connections.html |
| Ключ кластера 32 символа; в `<nodes>` один master | https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/cluster.html |
| Шарды/копии; завод 3 primary / 0 replicas; куча `Xms=Xmx` | https://documentation.wazuh.com/current/user-manual/wazuh-indexer/wazuh-indexer-tuning.html |
| Смена паролей: `admin`, `kibanaserver`, `wazuh` / `wazuh-wui`; инструмент 4.14; правила сложности | https://documentation.wazuh.com/current/user-manual/user-administration/password-management.html |
| Смена API; смена `wazuh-wui` ломает dashboard | https://documentation.wazuh.com/current/user-manual/api/securing-api.html |
| Агент: ОС, ~35 МБ RAM, мастер в UI, manager ≥ agent | https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html |
| Пакет Linux, `WAZUH_MANAGER`, выключить APT-репозиторий | https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html |
| `WAZUH_REGISTRATION_PASSWORD` | https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/deployment-variables/deployment-variables-linux.html |
| `authd.pass`, `<use_password>yes` | https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/security-options/using-password-authentication.html |
| Kubernetes: clone `-b v4.14.7`, пароль агентов `password` | https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-deployment.html |
| K8s минимум 2 CPU / 3 ГиБ / 2 ГиБ; StatefulSet vs Deployment | https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-conf.html |
| Репозиторий манифестов v4.14.7; секреты overlay | https://github.com/wazuh/wazuh-kubernetes/tree/v4.14.7 |
| `local-env`: 1 indexer, 1 worker; port-forward dashboard `8443:443` | https://github.com/wazuh/wazuh-kubernetes/blob/v4.14.7/local-environment.md |
| Docker: `vm.max_map_count=262144`; single-node ≥ 4 CPU / 8 ГБ / 50 ГБ; dashboard `admin`/`SecretPassword`; агент в контейнере хост не видит | https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html |
| Образы `wazuh/*:4.14.7` в overlay | https://raw.githubusercontent.com/wazuh/wazuh-kubernetes/v4.14.7/wazuh/indexer_stack/wazuh-indexer/cluster/indexer-sts.yaml |
| Зачем продукт, порты, роль в платформе | `Wazuh.md` |
| Словарь | `Wazuh.info.md` |
| Стыковка с площадками | `Wazuh.shema.md` |
| Роль консультанта | `Wazuh.consultant.md` |

Порога задержки для растяжки indexer/server на несколько дата-центров в документации Wazuh **нет** — поэтому растяжка здесь не предлагается. Отдельной цифры «минимум CPU/RAM, чтобы all-in-one только стартанул» нет. Сметы диска под ваш APS нет.
