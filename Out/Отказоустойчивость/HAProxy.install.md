# HAProxy 3.4.3 — установка и конфигурирование

Связанный документ (глоссарий, frontend/backend, VRRP, почему так): `HAProxy.md`.

Этот файл — **как поставить и настроить**. Stretch одного HAProxy (или одного Keepalived) на несколько ЦОДов **не делаем**: у процесса нет кворума как у etcd; межЦОДовый ping не склеивает VIP. HA — **пара внутри ЦОДа**; между площадками — независимые экземпляры + DNS/край.

Версия: **HAProxy 3.4.3** (LTS линии 3.4, поддержка на haproxy.org **до Q2 2031**). Образ: `haproxy:3.4.3` (не `latest`, не `3.5-dev`).  
Документация: Starter https://docs.haproxy.org/3.4/intro.html · Management https://docs.haproxy.org/3.4/management.html · Configuration https://docs.haproxy.org/3.4/configuration.html

Роли на платформе: **B** — listen TCP **:6443** перед `kube-apiserver` (ControlPlaneEndpoint, см. `Kubernetes.install.md`); при необходимости **A** — вход TCP/HTTP периметра ЦОДа. Это не WAF, не Ingress-контроллер и не шина Kafka.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch запрещён.** Пара HAProxy + Keepalived живёт **внутри одного ЦОДа**. 2 ЦОДа = две пары. 3 ЦОДа = три пары. Общего VIP на город у HAProxy нет.
2. Берём **community 3.4.3**, не Enterprise. Ingress Controller `haproxytech/kubernetes-ingress` **3.2.13** собран с движком **3.2**, не 3.4.3 — это другой продукт; в этом файле его не ставим как «тот же инстанс».
3. Прод control plane — kubeadm HA в каждом ЦОДе (`Kubernetes.install.md`). LB :6443 — на двух машинах **той же** площадки.
4. Dev — изолированная сеть. Нагрузки нет — **нет** числа ядер и `maxconn` «хватит для прода».
5. Keepalived — **другое ПО**. Ставить только если VIP реален (L2 или unicast VRRP + маршрутизация). Нет VIP — пара всё равно два процесса, снаружи их склеивает что-то ещё (Service/DNS), не «магия HAProxy».
6. Anycast/BGP на три ЦОДа — **решение сети**, не кластеризация HAProxy. В `HAProxy.md` это вариант B: мощно, если сеть умеет; stick-table/peers между ЦОДами обычно **не** включают. Слабое место: отказ площадки обрабатывает маршрутизация, процесс об этом не договаривается.
7. HAProxy **не** ставим единственным `:9092` перед всеми брокерами Kafka.

Критические пробелы: как клиент выбирает ЦОД при аварии (DNS/ADC/Anycast); есть ли L2 для VIP; карта портов (только 6443 или ещё HTTP); таймауты Integration API; порядок относительно WAF.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** понять frontend/backend, `haproxy -c`, health check; чтобы kubeconfig или тестовый HTTP ходили в одно место. **Не** цель: отказ ЦОДа и TLS-ёмкость.

### Предпосылки

- Docker Engine **или** Linux-пакет линии 3.4.3.
- Для роли B: где слушает единственный kube-apiserver (часто :6443 на control-plane). Для роли A: тестовый HTTP origin.
- Сеть стенда не торчит в интернет.

### Установка (Docker — основной путь Dev)

Конфиг на хосте (пример **роли B**, бэкенд подставьте свой; на Dev часто один api-сервер — это не HA):

```
global
    log stdout format raw local0
    maxconn 256

defaults
    mode tcp
    timeout connect 5s
    timeout client  1m
    timeout server  1m
    option tcplog

listen k8s-api
    bind *:6443
    mode tcp
    option tcp-check
    server cp1 <адрес-apiserver-с-контейнера>:6443 check
```

Адрес `cp1` — тот, куда контейнер **реально** достучится до apiserver. Слепой `127.0.0.1` внутри контейнера — это сам HAProxy, не хост. Host-gateway (Docker Desktop `host.docker.internal`, Linux extra_hosts и т.п.) зависит от среды — подставьте свой, не копируйте чужой IP.

```bash
docker run -d --name haproxy-dev \
  -v ${PWD}/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro \
  -p 127.0.0.1:6443:6443 \
  haproxy:3.4.3
```

Привязка к `127.0.0.1` обязательна: `-p 6443:6443` без адреса часто слушает все интерфейсы. Stats (`:8404`) на Dev — только loopback, лучше вообще не bind'ить наружу.

Проверка:

```bash
docker exec haproxy-dev haproxy -v
docker exec haproxy-dev haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

В строке версии — **3.4.3**. `haproxy -vv`: есть ли PROMEX, какая OpenSSL. Затем `kubectl` / `curl -k https://127.0.0.1:6443` — ответ api-сервера, не таймаут.

HTTP-роль A на Dev — отдельный `mode http` frontend `:80` на тестовый origin, тот же образ, тот же bind на localhost (другой хост-порт, если 80 занят).

### Конфигурирование Dev

| Параметр | Значение | Зачем |
|---|---|---|
| Один процесс | да | Некому делить VIP |
| TLS | нет | Иначе PKI раньше health check |
| Keepalived / peers | нет | Нет второго узла |
| `master-worker` | лучше сразу | Тот же reload, что в проде |
| stats HTML | можно на loopback | Учиться смотреть ферму |

Чего **не** упрощать: всегда `haproxy -c` перед reload; хотя бы tcp-check (для HTTP — **HTTP check**, не только TCP :443); клиенты ходят **на адрес HAProxy**, не в обход; таймауты под долгие интеграции, если этот стенд их имитирует; образ **3.4.3**, не `haproxy:latest`.

### Проверка Dev

1. `haproxy -v` = 3.4.3.
2. Origin down → check вычёркивает (на одном сервере вход умрёт — это ожидаемо).
3. Рестарт контейнера: конфиг с volume на месте.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Минуты, официальный образ | Падение процесса = нет входа |
| Совпадает со Starter Guide | Не ловит окна reload, VRRP, CPU TLS |
| | Успех на одном процессе ≠ две–три площадки |

Препрод: **пара + Keepalived (или два пода + Service) + TLS** в **одном** ЦОДе, даже без боевого RPS.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **одной VM/пода HAProxy внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой того, что городской слой перестанет слать на эту площадку. Цифр Gbps нет.

### Почему не stretch

HAProxy **stateless по задумке** (Starter Guide): несколько процессов — несколько независимых прокси. Keepalived делит VIP в одном L2/маршрутном домене. Растянуть один VRRP на три зала — скрытый SPOF и split-brain, не HA. Peers stick-table через город обычно не делают: другой смысл лимита, партиция таблицы.

### Топология

**Внутри каждого ЦОДа** — пара:

- 2 процесса **3.4.3** (VM или hostNetwork-поды на разных нодах);
- VIP через **Keepalived** (track: HAProxy жив, иначе VIP на мёртвом прокси — классический простой) **или** Kubernetes Service, если это не control-plane;
- роль B: `listen` TCP **:6443** → kube-apiserver **этого** кластера (health — TCP до 6443, как в `Kubernetes.install.md`);
- роль A (если нужна): frontend HTTP(S) → WAF/Ingress/**локальный** origin;
- `ControlPlaneEndpoint` = DNS этого VIP. Один VIP «в чужом ЦОДе» = SPOF **этого** Kubernetes.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | Пара+VIP в ЦОД-1, такая же независимая пара в ЦОД-2. Клиенты/kubectl смотрят в endpoint **своего** кластера | API и вход ЦОД-1 мертвы. ЦОД-2 жив. Общего `kubectl` нет |
| **3 ЦОДа** | Третья пара | То же |

Городской HTTP-вход (браузеры): DNS/ADC **или** Anycast, если сеть так решила. Anycast **не** делает из трёх HAProxy один кластер: нет общего состояния, нет единого reload, peers между ЦОДами не появляются сами. Сложность split-horizon — на сетевиках (`HAProxy.md`, вариант B).

Origin — **этот** ЦОД. Не тащить HTTP через город «на всякий случай». Kafka advertised listeners — мимо этого LB.

### Предпосылки прода

- Linux, ядро ≥ 3.9 (`SO_REUSEPORT` для reload). **Не swap** (Starter Guide).
- `ulimit` / `LimitNOFILE` согласованы с `maxconn`.
- NTP. Syslog или stdout + ротация.
- PKI для :443, если терминируете TLS. :6443 — только админские сети, не «как у сайта».
- Конфиг в Git; применение: `haproxy -c` затем reload (`SIGUSR2` / `-sf`), не `kill -9`.

### Установка пары (повторить в каждом ЦОДе)

Официальный порядок для kubeadm: **сначала LB**, потом первый control-plane (`Kubernetes.install.md`).

1. Две машины площадки, пакет или образ **3.4.3**, пользователь не root после bind (официальный образ 2.4+ уже `USER haproxy`).
2. `haproxy.cfg` роли B (и отдельно frontend A, если нужен — **разные bind**, ACL на 6443).
3. Keepalived: VIP внутри ЦОДа; `track_script` на процесс/порт HAProxy; **не** один VRRP на три ЦОДа.
4. DNS `ControlPlaneEndpoint` → VIP. Проверить `kubeadm init --control-plane-endpoint`.
5. PROMEX / метрики с localhost; stats page не с мира.

Смысл listen :6443 (не полный боевой файл):

```
defaults
    mode tcp
    timeout connect 10s
    timeout client  1m
    timeout server  1m

listen k8s-api
    bind <vip>:6443
    mode tcp
    option tcplog
    option tcp-check
    server cp1 <ip-cp1>:6443 check
    server cp2 <ip-cp2>:6443 check
    server cp3 <ip-cp3>:6443 check
```

Таймауты HTTP-роли под Integration API — **из замера** лагов ведомств, не дефолт «секунды»: иначе HAProxy рвёт легитимный long-poll. `timeout tunnel` — для websocket.

Проверка конфига в CI и в unit-файле: `haproxy -c -f /etc/haproxy/haproxy.cfg`.

### Конфигурирование прода

| Параметр | Прод | Зачем |
|---|---|---|
| Версия | pinned **3.4.3** на всех узлах пары | Иначе ACL/TLS разъедутся |
| `master-worker` | да | Штатный reload |
| `hard-stop-after` | явно, по длине сессий | Не копить старые worker |
| Health check | TCP+проверка api / HTTP к origin | Голый TCP :443 врёт для HTTP |
| Peers | если stick-table (липкость, rate-limit) **внутри ЦОДа** | Reload иначе обнуляет таблицу |
| Peers между ЦОДами | обычно нет | Не городской stateful-кластер |
| `nbthread` | начать с ядер **сокета**, не «все 128» без замера | Starter Guide: межсокетный трафик — тормоз |
| `maxconn` | по RAM **после замера** | Цифры sizing из гайдов 1.6 — не ваш план |

Официальный ориентир Starter Guide: **одного ядра хватает >99% установок**; много потоков — в основном TLS и очень широкие каналы. Это не замер вашего периметра. Бенчмарки 2010-х / чужое железо в `HAProxy.md` **не** копировать в закупку.

Reload не zero-drop: Management Guide описывает короткие окна потери **новых** коннектов. Катить **все** площадки сразу нельзя. Апгрейд: снять узел с VIP → reload → вернуть.

### Масштабирование (когда появятся цифры)

1. Замерить новые TLS/с, RPS keep-alive, concurrent, `maxconn` usage, CPU vs softirq.
2. Упёрлись handshake — keep-alive, tickets (ключи **синхронизировать** между узлами пары), не слепой `nbthread=128`.
3. Упёрлись FD — лимиты и keep-alive, потом ещё узлы **в этой** паре/пуле ЦОДа.
4. Горизонталь между ЦОДами — новые независимые пары, не «replicas без VIP».
5. HPA «50 подов» без VIP и без peers = 50 лимитеров.

Кэш RAM и compression — точечно; Starter Guide: это не замена Varnish.

### Безопасность (без этого периметр не настроен)

1. На мир — только нужные listen (80/443). Не stats, не runtime socket, не peers, не **6443**.
2. Runtime API — unix socket, группа админов. Stats `:8404` с мира = карта бэкендов и drain.
3. `ssl-min-ver TLSv1.2` (лучше 1.3 где клиенты умеют). Ключи не в Git. `ssl verify none` к origin в проде не настройка.
4. Реальный IP: PROXY Protocol **или** XFF только от доверенных hop. `forwardfor` слепо с мира = подделка.
5. Origin принимает трафик **только** с IP HAProxy (и WAF). Иначе LB обходят.
6. GitOps конфига; ручной `vi` на одном ЦОДе = дрейф входа.

### Проверка прода (пока это не пройдено — это не прод)

Внутри ЦОДа:

1. `haproxy -v` = 3.4.3 на обоих узлах.
2. Выключить один HAProxy: VIP уехал (Keepalived track сработал), `kubectl` / HTTP живы.
3. Выключить один kube-apiserver: check снял его, :6443 отвечает.
4. Reload под синтетикой; есть `hard-stop-after`; доля обрывов зафиксирована (ориентир Management Guide **~1/10000** новых коннектов на reload — не гарантия на вашем ядре).

Между ЦОДами: выключить вход ЦОД-1 — городской слой (если он есть) не шлёт сюда; kubectl ЦОД-2 смотрит в **свой** endpoint. Anycast, если включили, проверяют сетевики: это не «кластер HAProxy встал».

### Сильные / слабые стороны (пара в ЦОДе + независимые площадки)

| Сильное | Слабое |
|---|---|
| Отказ VM закрывается VIP внутри зала | Keepalived — свой split-brain |
| Согласовано с kubeadm HA и запретом stretch | 2–3 копии `haproxy.cfg`; без GitOps разъедутся |
| Stateless, понятный reload | Городской failover — DNS/ADC/Anycast, не бинарь |
| | Anycast ≠ кластер HAProxy: нет общего состояния |

**Не готов к проду**, если: один процесс на три ЦОДа; `haproxy:latest`; stats с мира; нет health check; Kafka через один TCP frontend на все брокеры; Keepalived без track; VRRP растянут на ЦОДы; peers нет, а rate-limit «есть»; reload без `-c`; VIP control-plane в чужом ЦОДе; Ingress Controller 3.2 выдают за 3.4.3.

---

## Источники

- Релизы и LTS 3.4 (Q2 2031), пакет 3.4.3: https://www.haproxy.org/
- Starter (VRRP/keepalived, «HAProxy is not», sizing): https://docs.haproxy.org/3.4/intro.html
- Management (reload, `SO_REUSEPORT`, master-worker): https://docs.haproxy.org/3.4/management.html
- Configuration: https://docs.haproxy.org/3.4/configuration.html
- Образ: https://hub.docker.com/_/haproxy
- Ingress Controller 3.2.13 ≠ 3.4.3: https://github.com/haproxytech/kubernetes-ingress/releases
- Правила и пробелы: `HAProxy.md`

Утверждений «Keepalived склеит три ЦОДа» в документации **нет**. Stretch в этой инструкции не предлагается.
