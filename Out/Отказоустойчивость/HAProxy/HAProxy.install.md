# HAProxy 3.4.3 — установка (учебный контур)

**Допущение:** закрытая сеть, одна Linux-машина (или Docker Desktop с Linux-образом), community **3.4.3**, один процесс, конфиг на хосте. Это не бой, не пара с Keepalived, не Ingress-контроллер.

**HAProxy** — балансировщик TCP и HTTP: принимает соединение и раскидывает на живые серверы. **Docker** — программа, которая запускает **образ** (упакованная программа с зависимостями) как контейнер. Образ: `haproxy:3.4.3` (не `latest`, не `3.5-dev`). Линия 3.4 — LTS до **Q2 2031**; патч **3.4.3** — 29 июля 2026.

Официальный образ **не** кладёт конфиг «из коробки»: без вашего `haproxy.cfg` слушать нечего.

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна машина рядом с учебным Kubernetes / тестовым HTTP-origin. Основной путь — Docker, конфиг каталогом на хосте (как на Docker Hub). Пакет линии 3.4.3 — тот же бинарь, другой способ поставки.

Не ставим в этом файле: Ingress `haproxytech/kubernetes-ingress` **3.2.13** (движок **3.2**, не 3.4.3), Keepalived, Windows как хост процесса (официальный образ — Linux; Docker Desktop запускает Linux-образ).

Живой процесс **не** растягивать на 2–3 дата-центра: у HAProxy нет голосования как у etcd. На стенде — один процесс.

| Зачем цифра | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Ориентир вендора «процесс живёт» | **одного ядра хватает >99% установок** | минимума «сколько ГБ, чтобы встал» **нет** | исполняемый файл + конфиг; кэш — RAM, не диск | [intro](https://docs.haproxy.org/3.4/intro.html) |
| Учебный контур | 1 ядро, **не** душить «долей ядра» гипервизора, **не swap** | `maxconn 256` — из **примера** мануала, не норма боя | локальный каталог конфига | intro + [пример 2.10](https://docs.haproxy.org/3.4/configuration.html) |
| Бой / ваша нагрузка | не эта таблица | не эта таблица | журналы + ротация | нагрузки в запросе платформы нет |

**Сильная сторона:** минуты, официальный образ, тот же `haproxy -c` и reload, что потом в бою.  
**Слабая сторона:** падение этой машины = нет входа, если DNS/kubeconfig смотрят только сюда.

**Критично:** порты stats и 6443 **не** в интернет; не `haproxy:latest`; один процесс ≠ кластер; учебный `stats auth` в бой не копировать.

## Установка для новичка

Официальные страницы шагов: образ — https://hub.docker.com/_/haproxy · проверка и reload — https://docs.haproxy.org/3.4/management.html · ключевые слова — https://docs.haproxy.org/3.4/configuration.html

**Frontend** — вход (IP:порт, HTTP или TCP). **Backend** — ферма серверов и проверка живости. **Listen** — frontend и backend в одном блоке (удобно для TCP 6443). **Stats socket** — unix-сокет пульта (не веб-админка). **Stats page** — встроенная HTML-статистика.

### Что должно быть до установки

Есть:

- Docker. Для портов **ниже 1024** не от root образ 2.4+ требует ядро хоста **≥ 4.11** и `--sysctl net.ipv4.ip_unprivileged_port_start=0`. На этом стенде слушаем **8080 / 8404 / 6443** — все ≥ 1024, sysctl не нужен.
- Ядро **≥ 3.9**, если хотите нормальный мягкий reload (`SO_REUSEPORT`).
- Закрытая сеть; с хоста свободны **8080** (учебный HTTP), **8404** (stats, порт **наш**, заводского нет), **6443** (API Kubernetes, если будете проксировать).
- Куда ходить с бэкенда: HTTP-origin и/или адрес api-сервера, **достижимый из контейнера**. `127.0.0.1` внутри контейнера — это сам HAProxy, не хост.

Нет (и не нужно на этом стенде): Keepalived, VIP, TLS, копирование таблиц (peers), Ingress-контроллер, публикация stats в сеть.

### Этапы

**1. Каталог и файл конфигурации**

**Что делаем:** создаём каталог и `haproxy.cfg`. Образ монтирует **каталог** в `/usr/local/etc/haproxy`; внутри должен быть файл именно `haproxy.cfg`.

Подставьте адреса origin и api-сервера. Нет api-сервера — блок `listen k8s-api` удалите, иначе проверка живости будет красной (процесс при этом **встанет**).

```text
global
    log stdout format raw local0
    maxconn 256
    master-worker
    stats socket /tmp/haproxy.sock mode 600 level admin
    stats timeout 2m

defaults
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend http-in
    bind *:8080
    mode http
    option httplog
    default_backend web

backend web
    mode http
    option httpchk
    http-check send meth GET uri /
    server s1 <адрес-origin>:80 check

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats hide-version
    stats realm HAProxy\ Statistics
    stats auth stand:stand-only

listen k8s-api
    bind *:6443
    mode tcp
    option tcplog
    option tcp-check
    server cp1 <адрес-apiserver>:6443 check
```

`maxconn 256` и таймауты 5s / 50s — из примера в мануале конфигурации, не смета боя. Пароль stats — **только закрытый стенд**, в конфиге он **открытым текстом** (так устроен HTTP Basic; вендор прямо пишет не считать его секретом «как у банка»). `stats admin` **не** включаем: кнопки снять сервер с фермы со страницы не нужны.

Успех: файл на диске, тег образа ещё не `latest`.

**2. Проверка синтаксиса до старта**

**Что делаем:** гоняем `haproxy -c` тем же образом, без долгоживущего контейнера. Так же проверяют перед каждым reload.

```bash
docker run --rm -v /путь/к/каталогу/haproxy:/usr/local/etc/haproxy:ro \
  haproxy:3.4.3 haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Успех: конфиг принят, процесс сразу выходит. Ошибка — контейнер для трафика не поднимаем.

**3. Запуск контейнера**

**Что делаем:** монтируем каталог **только чтение**, публикуем порты **на loopback хоста**.

```bash
docker run -d --name haproxy-dev \
  -v /путь/к/каталогу/haproxy:/usr/local/etc/haproxy:ro \
  -p 127.0.0.1:8080:8080 \
  -p 127.0.0.1:8404:8404 \
  -p 127.0.0.1:6443:6443 \
  haproxy:3.4.3
```

Привязка к `127.0.0.1` обязательна. Внутри контейнера `bind *:8404` — иначе проброс порта хоста не попадёт в listener (петля контейнера ≠ петля хоста). Образ 2.4+ стартует как пользователь **`haproxy`**, не root. Сокет пульта — `/tmp/haproxy.sock`: путь должен быть доступен этому пользователю; `/var/run/haproxy.sock` из примера management на не-root часто не создаётся.

Успех: `docker ps` — контейнер `Up`.

**4. Версия, сокет, страница stats**

```bash
docker exec haproxy-dev haproxy -v
docker exec haproxy-dev ls -l /tmp/haproxy.sock
curl -sI http://127.0.0.1:8404/
curl -sI -u 'stand:stand-only' http://127.0.0.1:8404/
```

Успех: в версии **3.4.3**; файл сокета есть; без учётки **401**; с учёткой **200** и HTML. Вендор для команд на сокете показывает `socat`; в официальном образе **socat не описан** — «стенд живой» закрываем страницей stats и `haproxy -c`.

HTTP через балансировщик (origin должен отвечать на check):

```bash
curl -sI http://127.0.0.1:8080/
```

Успех: ответ origin, не таймаут. На stats ферма `web` — зелёная. Origin выключен → check вычёркивает (на одном сервере вход умрёт — ожидаемо).

API Kubernetes: `kubectl` / `curl -k https://127.0.0.1:6443` — ответ api-сервера. Один api-сервер за входом — **не** запас.

**5. Reload после правки конфига**

**Что делаем:** снова `haproxy -c` (этап 2), затем мягкий reload. Docker Hub: `SIGHUP` → wrapper образа → `-sf`.

```bash
docker kill -s HUP haproxy-dev
```

Успех: контейнер жив, stats снова 200 с учёткой. Reload **не** «ноль оборванных»: при высокой нагрузке часть **новых** соединений может отвалиться (ориентир порядка 1 обрыв на 10000 новых/с на reload — не гарантия на вашем ядре). На этом стенде нагрузки нет — окна не поймаете.

Не `docker kill` без `-s HUP` и не `kill -9`.

### Чего этот стенд не доказывает

Отказ машины или зала, VIP/Keepalived, два держателя адреса, копирование stick-table, CPU TLS, окна reload под нагрузкой, выборы лидера (их в продукте нет), что один процесс = две–три площадки. Ingress-контроллер 3.2 сюда не ставили.

## Первый запуск — URL, порт, учётка, смена пароля

Встроенной админки «из коробки» и пароля завода **нет**. Слушает только то, что вы написали в `bind`.

| Что | На этом стенде | Проверка bind |
|---|---|---|
| Учебный HTTP | **http://127.0.0.1:8080/** — frontend `http-in` | `curl -sI http://127.0.0.1:8080/` |
| Stats page | **http://127.0.0.1:8404/** — listen `stats`, `stats uri /` | без учётки 401; с учётка 200 |
| API Kubernetes | **https://127.0.0.1:6443** — TCP-тоннель, HAProxy TLS **не** снимает | kubeconfig / `curl -k`; 6443 не с мира |
| Пульт | unix `/tmp/haproxy.sock` внутри контейнера | `ls` в контейнере; TCP stats на 9999 **не** открываем |

Заводского порта stats нет. **8404** — наш bind. Если в конфиге другой порт — URL другой; сверяйте `bind`, не память.

**Учётка stats:** пользователь `stand`, пароль `stand-only` (строка `stats auth`). Пользователей `admin`/`admin` продукт не создаёт. Без `stats auth` страница по умолчанию **без пароля** — на стенде так не оставляем.

**Смена пароля:** правите `stats auth` → `haproxy -c` → `docker kill -s HUP haproxy-dev`. Отдельной команды «passwd» нет. Новый секрет — не в git; в бою — свой секрет / Vault, не эта строка. Примеры мануала (`admin1:AdMiN123`) **не** копировать.

Страницу stats в интернет не выставлять: это карта ферм. `stats admin` на HTML не включали — и не включайте «чтобы потыкать».

## Подключение к своей системе

Клиенты бьют в **адрес этого HAProxy** (на стенде loopback), не в обход на origin / api-сервер.

| | |
|---|---|
| Протокол | **HTTP** на 8080 (заголовки, URL, cookies). **TCP** на 6443 — байтовый тоннель к kube-apiserver, не `mode http`. TLS на 443 в этом контуре нет |
| Кто клиент | Браузер / HTTP-клиент приложения; **kubectl** и контроллеры — на 6443 **этой** площадки (`Kubernetes.install.md`). Микросервисы внутри кластера ходят через Service/kube-proxy, не через этот вход |
| Секреты (не git) | `stats auth`; позже — сертификаты TLS и ключи ticket между узлами пары. Сокет пульта — unix + права, не TCP с мира |
| Что класть в git | `haproxy.cfg` **без** боевого пароля stats и без ключей |

HAProxy **не** единственный `:9092` перед всеми брокерами Kafka (ломает advertised listeners). Origin в бою принимает трафик только с IP балансировщика (и WAF, если он следующий hop).

**Чем продукт не является**

| Сосед / ожидание | Чем отличается |
|---|---|
| SafeLine WAF | Фильтр содержимого HTTP, не VIP и не раскладка TCP 6443 |
| Ingress-контроллер 3.2.13 | Другой репозиторий, движок **3.2**; объекты Ingress бинарь 3.4.3 не читает |
| kube-proxy | Трафик Service **внутри** кластера |
| Keepalived | Таскает VIP; HAProxy адрес между машинами **не** двигает |
| Шина Kafka / Camunda | Не брокер и не оркестратор; они могут быть origin за HTTP, не наоборот |

## Ссылки на материал

| Факт | URL |
|---|---|
| Релиз **3.4.3** (29 июля 2026), LTS 3.4 до **Q2 2031** | https://www.haproxy.org/ |
| Чем HAProxy является и не является; 1 ядро >99%; не swap; не душить CPU; файл+бинарь; Keepalived — другое ПО | https://docs.haproxy.org/3.4/intro.html |
| `haproxy -c`, reload `-sf` / SIGUSR2, окна обрыва, ядро 3.9 / `SO_REUSEPORT`, stats socket, socat | https://docs.haproxy.org/3.4/management.html |
| Пример: `maxconn 256`, timeout 5s/50s/50s, frontend/backend | https://docs.haproxy.org/3.4/configuration.html |
| `stats socket` (global); TCP stats «never done by default because this is dangerous» | https://docs.haproxy.org/3.4/management.html (раздел 9.3) |
| `stats enable`: дефолт URI `/haproxy?stats`, realm «HAProxy Statistics», **без** auth | https://docs.haproxy.org/3.4/configuration.html#4-stats%20enable |
| `stats auth` — HTTP Basic, пароль в конфиге открытым текстом | https://docs.haproxy.org/3.4/configuration.html#4-stats%20auth |
| `stats uri`; на отдельном listen удобен префикс `/` | https://docs.haproxy.org/3.4/configuration.html#4-stats%20uri |
| Образ `haproxy:3.4.3`; нет дефолт-конфига; путь `/usr/local/etc/haproxy/haproxy.cfg`; `-c`; `USER haproxy`; ядро ≥ 4.11 для портов &lt; 1024; `docker kill -s HUP` | https://hub.docker.com/_/haproxy |
| Ingress-контроллер **3.2.13** ≠ HAProxy **3.4.3** | https://github.com/haproxytech/kubernetes-ingress/releases |
| Правила, порты, железо | `HAProxy.md` |
| Словарь | `HAProxy.info.md` |
| Стыковка с платформой | `HAProxy.shema.md` |
| Роль консультанта | `HAProxy.consultant.md` |
| Control plane за TCP 6443 | `../Платформенная инфра/Kubernetes.install.md` |

**В доке вендора нет (не угадывать):** минимум RAM «чтобы контейнер просто встал»; заводской порт и пароль stats; порог RTT для peers/Anycast «на город»; «хватит N ядер на наш периметр»; Windows как хост балансировщика; socat в официальном Docker-образе; stretch-кластер HAProxy на 2–3 ЦОДа.
