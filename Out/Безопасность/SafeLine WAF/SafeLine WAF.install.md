# SafeLine WAF 9.4.0 — установка (учебный контур)

SafeLine — обратный прокси: клиент стучится в WAF, тот проверяет HTTP/HTTPS и, если чисто, пересылает на **upstream** (Ingress этого зала). Это не Ingress, не NetworkPolicy и не библиотека в коде.

**Допущение:** закрытая сеть, одна Linux-машина **x86_64**, редакция **Personal**, версия **9.4.0**. Боевой запуск сюда не копировать.

Официальный путь: Linux + **Docker Compose** (Docker — программа, которая запускает **образ**, упакованную программу с зависимостями, как **контейнер**; Compose поднимает пачку контейнеров по `compose.yaml`). Docker ≥ 20.10.14, Compose v2 ≥ 2.0.0. Автоинсталлятор `manager.sh` и китайский `waf-ce.chaitin.cn` **не** используем: скрипт ставит `latest`. Ручной путь с [Deploy](https://docs.waf.chaitin.com/en/GetStarted/Deploy), образы `chaitin/safeline-*`, `IMAGE_TAG=9.4.0`, `REGION=-g`.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Выделенная Linux-машина **x86_64** **рядом** с учебным Kubernetes, не внутри него как Ingress. На этой VM — Docker Compose. **Tengine** (форк Nginx, контейнер `safeline-tengine`) в `network_mode: host`: слушает порты **самой машины**. На хосте свободны **80/443**. Второй процесс с теми же портами (Ingress, nginx) — конфликт.

macOS и Windows вендор не поддерживает ни в одной редакции. **ARM = Pro**; Personal на ARM не ставим. Каталог данных — **локальный диск**, не NFS (NFS как data dir в мануале вендора **нет**). Community Helm (`replicas: 1`) — не Install Guide, в этот стенд не входит.

```mermaid
flowchart LR
  U["Клиент / партнёр"] -->|"80/443"| WAF["Linux VM + Compose\nTengine host-сеть"]
  WAF -->|"чистый HTTP"| ING["Ingress этого зала"]
```

**Сколько.** Минимум «чтобы контейнеры встали» и учебный ориентир — разные цифры. Путать нельзя.

| Зачем | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Минимум, чтобы процесс поднялся | 1 ядро | 1 ГБ | 5 ГБ | [Deploy](https://docs.waf.chaitin.com/en/GetStarted/Deploy) |
| Учебный контур (&lt; 100 запросов/с) | **2 ядра** | **4 ГБ** | `/data/safeline` на локальном диске | [FAQ](https://docs.waf.chaitin.com/en/faq/home) |

Для учёбы берите **2 CPU / 4 ГБ / ≥ 5 ГБ** свободных. FAQ: цифры — справка, не гарантия. Сметы боя здесь нет: вашей HTTP-нагрузки в контексте нет.

**Сильная сторона:** совпадает с Install Guide, одна VM. **Слабая:** падение VM = нет WAF и, если DNS только сюда, нет HTTP-входа.

**Критично:** **9443** (консоль) и **5432** (Postgres узла) в интернет не публиковать. Не `IMAGE_TAG=latest`. Один Compose — не кластер: у узла свой PostgreSQL, общей БД на залы нет.

---

## Установка для новичка

Команды — **на Linux-машине стенда**, не в PowerShell. Права root (так в Install Guide). Страница шагов: https://docs.waf.chaitin.com/en/GetStarted/Deploy

### Что должно быть до установки

**Есть:**

- Linux x86_64, флаг CPU **SSSE3**
- закрытая сеть; вход с jump-хоста / VPN
- свободны **80**, **443**, **9443**, **65508** (health check, не меняется), **65443** (своя страница ошибки, не меняется)
- ≥ 5 ГБ на каталог данных

**Нет** (и не должно появиться на этой VM):

- Ingress / nginx на 80/443
- публикация 9443 и 5432 в интернет
- ARM без Pro
- `.env` в git

### Этап 1. Проверка машины

**Что делаем:** убеждаемся, что это x86_64 с SSSE3 и хватает места. Без SSSE3 официальный путь не тот.

```bash
uname -m
lscpu | grep ssse3
cat /proc/cpuinfo | grep "processor"
free -h
df -h
```

Успех: `uname -m` → `x86_64`; в `lscpu` есть `ssse3`; свободно ≥ 5 ГБ.

### Этап 2. Docker и Compose

**Что делаем:** ставим Docker и плагин Compose v2, если их ещё нет. Уже стоят — только версии.

```bash
curl -sSL "https://get.docker.com/" | bash
docker version
docker compose version
```

Успех: Docker **≥ 20.10.14**, Compose **≥ 2.0.0** (`docker compose`, два слова — v2, не `docker-compose`). `Cannot connect to the Docker daemon` — демон не запущен: FAQ предлагает `systemctl restart docker`. На закрытом контуре `get.docker.com` может быть недоступен — Docker из **вашего** зеркала пакетов, те же минимальные версии.

### Этап 3. Каталог `/data/safeline`

**Что делаем:** создаём каталог конфига, PostgreSQL и логов **этого** узла. Вендор по умолчанию `/data/safeline`, не меньше 5 ГБ. Не NFS.

```bash
mkdir -p /data/safeline
cd /data/safeline
```

Успех: `pwd` → `/data/safeline`.

### Этап 4. `compose.yaml`

**Что делаем:** скачиваем официальный файл стека. В нём tengine с `network_mode: host`.

```bash
cd /data/safeline
wget "https://waf.chaitin.com/release/latest/compose.yaml"
```

Успех: файл лежит в `/data/safeline`. URL у вендора — `latest`; образы ниже пиним **9.4.0**. Не смешивать compose разных тегов: changelog **9.3.11** перевёл бандл Postgres **15.2 → 15.18**. Публичный GitHub `compose.yaml` ветки `main` может отставать (там ещё тег `15.2`) — ориентир для имён сервисов и host-сети, не замена файла **того** релиза.

### Этап 5. `.env` — pin 9.4.0, `REGION=-g`

**Что делаем:** создаём скрытый файл переменных рядом с compose. **Не** в git: там пароль Postgres. Не копировать `IMAGE_TAG=latest` из примера вендора.

```bash
cd /data/safeline
touch .env
```

Содержимое `.env`:

```bash
SAFELINE_DIR=/data/safeline
IMAGE_TAG=9.4.0
MGT_PORT=9443
POSTGRES_PASSWORD=ЗАМЕНИТЕ_СВОИМ_ПАРОЛЕМ
SUBNET_PREFIX=172.22.222
IMAGE_PREFIX=chaitin
ARCH_SUFFIX=
RELEASE=
REGION=-g
MGT_PROXY=0
```

| Переменная | Зачем |
|---|---|
| `SAFELINE_DIR` | Каталог данных |
| `IMAGE_TAG` | **9.4.0**, не `latest` |
| `MGT_PORT` | Консоль на хосте (внутри контейнера **1443**) |
| `POSTGRES_PASSWORD` | Пароль встроенной PostgreSQL узла. Свой, длинный |
| `SUBNET_PREFIX` | Префикс Docker-сети стека. Конфликт адресов → FAQ `Pool overlaps` |
| `IMAGE_PREFIX` | Реестр: `chaitin` |
| `ARCH_SUFFIX` | Пусто на x86_64. На ARM было бы `-arm`, но ARM = Pro |
| `RELEASE` | Пусто. Не `-lts` (LTS не сопровождают с 9.1.0-LTS) |
| `REGION` | `-g` — международная линейка |
| `MGT_PROXY` | Сколько прокси перед консолью. На стенде **0** |

Успех: `grep IMAGE_TAG /data/safeline/.env` → `9.4.0`; есть `REGION=-g`; пароль не плейсхолдер.

### Этап 6. Подъём стека

**Что делаем:** Compose скачивает образы и запускает контейнеры. Может занять несколько минут (так пишет вендор).

```bash
cd /data/safeline
docker compose up -d
```

Успех: команда без ошибки.

### Этап 7. `docker ps`

**Что делаем:** проверяем, что стек жив, до браузера.

```bash
docker ps
docker logs --tail 50 safeline-mgt
docker logs --tail 50 safeline-tengine
```

Успех: контейнеры `Up` / healthy, в логах нет циклического рестарта. В колонке образа у сервисов с `IMAGE_TAG` — **9.4.0**. Ожидаются: `safeline-tengine`, `safeline-detector`, `safeline-mgt`, `safeline-pg`, плюс `safeline-luigi`, `safeline-chaos`, `safeline-fvm`.

Если tengine пишет `Address already in use` — занят порт хоста (часто 80/443/9443/65508/65443). Если `failed to create network safeline-ce` — FAQ: `systemctl restart docker` и повтор.

**Чего этот стенд не доказывает:** отказ зала, рассинхрон правил на двух узлах, апгрейд с обрывом трафика, CPU детектора на пике, реальный IP за городским балансировщиком, выборы лидера (их в продукте нет). Рестарт VM при DNS только сюда = нет входа — на одном узле это ожидаемо.

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL:** `https://<ip-машины>:9443/` — порт задаётся `MGT_PORT`. Открывать **с VPN / jump-хоста**, не из интернета: это админка щита. Сертификат консоли самоподписанный (`curl -k` в healthcheck compose). Предупреждение браузера на стенде ожидаемо.

Страница входа: https://docs.waf.chaitin.com/en/GetStarted/Deploy#use-web-ui

**Учётка.** Готового пароля «на экране из коробки» нет. Его печатает сброс на машине стека:

```bash
docker exec safeline-mgt resetadmin
```

Ожидаемый вывод ([Get Administrator Account](https://docs.waf.chaitin.com/en/GetStarted/Deploy#get-administrator-account)):

```text
[SafeLine] Initial username：admin
[SafeLine] Initial password：**********
[SafeLine] Done
```

Логин: **`admin`**, пароль — из этой печати. Вендор: «Please must remember this content».

**Смена пароля.** Сразу после входа сменить в консоли. Учебный пароль из `resetadmin` в бой не копировать. Новый — вне git (сейф / Vault). Забыли — снова `resetadmin` ([FAQ Login Issues](https://docs.waf.chaitin.com/en/faq/home)).

**TOTP (Personal).** Консоль может потребовать одноразовый код (живёт ~30 с). Часы сервера и телефона — NTP, один часовой пояс. Код «не тот»: FAQ — время; перепривязка: `docker exec safeline-mgt resettotp`.

Роль read-only есть с **9.3.10** — на стенде не обязательна.

---

## Подключение к своей системе

Приложения **не** ходят в SafeLine как в библиотеку (нет JDBC/gRPC-клиента). Люди и партнёры открывают **домен, который слушает WAF**; WAF проксирует на origin — Ingress **этого** зала. Модель и поля сайта: https://docs.waf.chaitin.com/en/GetStarted/AddApplication

| Что | Как |
|---|---|
| Вход пользователей | HTTP/HTTPS, listen сайта, обычно **80/443** на хосте WAF |
| Консоль | HTTPS **9443** — админы с VPN |
| До origin | HTTP или HTTPS, как зададите **Upstream**. На закрытом стенде допустим открытый текст до локального Ingress |
| Health (если появится LB) | порт **65508**, не меняется |
| Лицензия Pro | исходящий **:50052** на `safeline.stream.safepoint.cloud`. На учебном Personal **не нужен**. Offline в Install Guide **нет** |

Клиенты: браузер и HTTP-партнёр — на **домен WAF**, не в обход на Ingress. Машинные интеграции (колбэки, health ведомств) — на тот же домен, в allowlist **до** капчи на этих URL. Syslog в SIEM — **Pro**, не Personal. Kafka, Camunda, Postgres эталона в SafeLine не подключаются.

**Первый учебный сайт:** Applications → Add Application. **Domain** — имя, как его видит клиент. **Port** — что слушает SafeLine (80 или 443; для HTTPS — опция SSL и сертификат). **Upstream** — Ingress **этого** зала, не HTTP через город. DNS/hosts: домен → **IP машины SafeLine**, иначе щит обходят.

```bash
curl -v -H "Host: <домен>" http://<ip-safeline>:<listen-порт>
```

Успех по FAQ: ответ origin пришёл **и** счётчик «Today's visits» в консоли вырос на 1. `502 Bad Gateway tengine` — WAF не достучался до upstream.

Политика стенда: сначала **журнал**, потом блокировка. Капчу и динамическую защиту **не** включать на URL роботов и JSON API. Реальный IP за балансировщиком (`Applications → Advanced → Get Attack IP From`) — **до** банов по IP; на одном узле без LB WAF берёт IP из сокета.

### Секреты (не в git)

| Секрет | Где живёт | Куда не класть |
|---|---|---|
| `POSTGRES_PASSWORD` | `/data/safeline/.env` | git, чат, образ |
| Пароль `admin` после смены | сейф / Vault | git, `.env` в репозитории |
| Ключ лицензии Pro (если появится) | консоль + сейф | git |
| Сертификаты сайтов на tengine | консоль «SSL Cert» / PKI | публичный репозиторий ключа |

В git — процедура и безсекретовый список имён переменных. Сам `.env` с паролем — нет.

### Чем продукт не является

| Сосед | Чем отличается |
|---|---|
| Ingress / NetworkPolicy | WAF их не заменяет. Origin в Kubernetes в бою принимает HTTP **только** с IP WAF |
| HAProxy | Балансировка и VIP; WAF — фильтр содержимого HTTP |
| Spring Cloud Gateway | Ваш Java-шлюз **внутри** платформы; SafeLine — на входе HTTP снаружи |
| Защита Kafka / gRPC / Postgres | Не видит шину и не читает тело не-HTTP |
| Исходящие вызовы к ведомствам | Стоит на **входящем** HTTP |
| Кластер с общей БД | У каждого узла свой Postgres. Personal ≠ кластер |
| Библиотека в коде приложения | Клиенты ходят на домен WAF |

На учебном стенде прямой обход origin ещё возможен — для отладки. В бою это дыра.

---

## Ссылки на материал

| Факт | URL |
|---|---|
| ОС Linux, x86_64/arm64, Docker ≥ 20.10.14, Compose ≥ 2.0.0, минимум 1 CPU / 1 ГБ / 5 ГБ, SSSE3, ручная установка, `.env`, `REGION=-g`, ARM=Pro, отказ от LTS, консоль `:9443`, `resetadmin`, пользователь `admin` | https://docs.waf.chaitin.com/en/GetStarted/Deploy |
| Сайт: Domain, Port, Upstream; модель обратного прокси | https://docs.waf.chaitin.com/en/GetStarted/AddApplication |
| Апгрейд рвёт трафик, бэкап = `compose down` + копия каталога, миграция необратима | https://docs.waf.chaitin.com/en/GetStarted/Upgrade |
| Релиз **9.4.0** (17 августа 2026), Postgres 15.18 в 9.3.11, read-only с 9.3.10 | https://docs.waf.chaitin.com/en/Reference/Changelog |
| FAQ: 2/4 от QPS; порты 9443 / 65508 / 65443; не macOS/Windows; `resetadmin` / `resettotp`; `SUBNET_PREFIX`; диагностика `curl` и 502 | https://docs.waf.chaitin.com/en/faq/home |
| Лицензия Pro, хост `:50052` | https://docs.waf.chaitin.com/LicenseDisconnectionInstructions |
| Обзор, обратный прокси | https://docs.waf.chaitin.com/ |
| Compose: host-сеть tengine, `MGT_PORT`→1443, имена контейнеров | https://github.com/chaitin/SafeLine/blob/main/compose.yaml |
| Зачем продукт, порты, железо | `SafeLine WAF.md` |
| Словарь | `SafeLine WAF.info.md` |
| Схема стыковки с платформой | `SafeLine WAF.shema.md` |
| Роль консультанта | `SafeLine WAF.consultant.md` |

**В доке вендора нет (и здесь не выдумано):** порог RTT между залами; «N запросов/с на ядро»; NFS как каталог данных; URL `compose.yaml` именно тега 9.4.0 (есть только `/release/latest/`); offline-схема лицензии; процедура «выбрать нового master»; готовый пароль admin без `resetadmin`.
