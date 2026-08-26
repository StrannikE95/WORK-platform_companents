# OpenStack Swift 2.37.3 — установка и конфигурирование

Связанный документ (глоссарий, кольца, S3, безопасность, почему так): `OpenStack Swift.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch **одного** кольца на несколько ЦОДов (один region с zone=ЦОД **или** Global Cluster с несколькими region) **не делаем**: это синхронный PUT/rsync по городской сети; порога RTT у проекта **нет**, а ping между площадками stretch не позволяет.

Версия: **Swift 2.37.3**, серия OpenStack **2026.1 (Gazpacho)**. Не 2.38.x (unreleased). Не **2.29.2** из инсталлятора GeoData — склеивать платформенный S3 с GeoData «потому что оба Swift» без письма вендора нельзя. Лицензия исходников — **Apache 2.0**. Официального оператора «как у CNPG» нет; Kolla-Ansible **убрал** роль Swift с 2025.1. OpenStack-Helm на 2026.1 заявляет Kubernetes **≥ 1.33 и ≤ 1.35** — у вас **1.36.4**, это **не** доказанная комбинация.

Документация: https://docs.openstack.org/swift/2026.1/  
Deployment Guide: https://docs.openstack.org/swift/2026.1/deployment_guide.html  
S3 (`s3api`): https://docs.openstack.org/swift/2026.1/middleware.html

Swift — объектный слой (вложения, снимки, архив интеграций), не SoT клиентских карточек и не Kafka.

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **Stretch колец запрещён.** Один кластер Swift (account/container/object rings, replica=3, rsync между storage-нодами) живёт **внутри одного ЦОДа**. Между ЦОДами — **независимые** кластеры (штатный мост — **container sync**) **или** один кластер + бэкап объектов/`.builder` как DR. Это варианты **C и D** из `OpenStack Swift.md`, не A/B (A = один region на 3 ЦОДа, B = Global Cluster — оба stretch).
2. Прод-identity — **Keystone** (отдельный HA-сервис). tempauth — только стенд.
3. Storage-ноды — машины с **локальным XFS** и стабильным IP в кольце, не `emptyDir` и не NFS как диск object-server. Proxy может быть ближе к Kubernetes.
4. Dev — изолированная сеть; учётки SAIO не секрет.
5. Нагрузки нет — нет цифры «N дисков и M ТБ». **Не** подставляем part_power «с потолка»: в гайде part_power считают от прогноза максимума дисков (≥ 100 партиций на диск, число партиций = 2^part_power). Здесь только протестированные константы гайда: **replica=3**, пример `min_part_hours=24`.
6. Для 2 ЦОДов: кластер в ЦОД-1, в ЦОД-2 — независимый Swift + container sync **или** только бэкап. Для 3 ЦОДов: то же + третья площадка. Третий ЦОД **не** добавляет «ещё одну копию в то же кольцо».
7. S3 Lifecycle / bucket policy / tagging в матрице Swift **нет** — в архитектуру не закладываем.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** PUT/GET объекта, AWS SDK/boto, ловить «lifecycle не поддерживается» на берегу. **Не** цель: отказ ЦОДа и терабайты.

### Предпосылки

- Одна Linux-машина/VM (официальный SAIO: ориентир гайда **≥ 2 ГиБ RAM и ≥ 40 ГиБ** места — чтобы **вообще завелось**, не прод).
- Порт 8080 свободен на localhost.
- Сеть стенда не торчит в интернет.

### Установка (SAIO — официальный путь Dev)

Полный плейбук не копируем: https://docs.openstack.org/swift/2026.1/development_saio.html — ветка/серия **2026.1**, пакеты и исходники **2.37.3**. Ниже — то, без чего стенд не считается тем же продуктом.

1. Зависимости (пример Ubuntu из SAIO): `memcached`, `rsync`, `sqlite3`, `xfsprogs`, Python 3, `liberasurecode-dev` и остальное по гайду **этой** серии.
2. XFS. Loopback как в SAIO:

```bash
sudo mkdir -p /srv
sudo truncate -s 1GB /srv/swift-disk
sudo mkfs.xfs /srv/swift-disk
sudo mkdir -p /mnt/sdb1
# fstab: /srv/swift-disk /mnt/sdb1 xfs loop,noatime 0 0
sudo mount -a
```

3. Каталоги `/srv/{1..4}` как в SAIO (лаборатория «несколько процессов на localhost», не прод-порты 6200).
4. rsyncd, memcached, кольца SAIO из гайда, затем proxy **только на loopback**:

```text
bind_ip = 127.0.0.1
bind_port = 8080
```

tempauth в `proxy-server.conf`. `s3api` в pipeline **перед** tempauth, как в sample; для более полной совместимости SAIO включает `s3_acl` / `check_bucket_owner`.

5. Версия процессов — **2.37.3**, не «какой Swift есть в дистрибутиве GeoData».

Проверка Swift API — `curl` на `/auth/v1.0` и `/v1/...` как в SAIO. Проверка S3: endpoint `http://127.0.0.1:8080`, access key вида `account:user`, secret = пароль tempauth.

Альтернатива «чуть ближе к бою»: 3 VM **в одном ЦОДе**, replica=3, несколько zone **внутри** площадки — уже видны replicator и кольца. Это всё ещё не модель двух городов.

На Kubernetes для теста: один под с SAIO в изолированном namespace, порт не публиковать. **Не** этот манифест в прод.

### Конфигурирование Dev

| Параметр | На тесте | Почему можно |
|---|---|---|
| Одна машина (SAIO) | да | Нет требования пережить выкат |
| Auth | tempauth | Иначе команда утонет в Keystone раньше PUT |
| TLS | HTTP на 127.0.0.1 | Стенд изолирован |
| Encryption at-rest | выкл. | Сначала протокол |
| Zone/region | одна | Не цель — отказ площадки |
| S3 Lifecycle | отсутствует как в проде | Пусть клиент упадёт на тесте |

Чего **не** упрощать: версия **2.37.3**; лимит **5 ГБ** на один PUT и multipart/SLO; явный ключ объекта (идемпотентный PUT).

### Проверка Dev

1. PUT/GET по Swift API и по S3 (разные URL и подписи — осознанно).
2. Вызов Lifecycle/bucket policy — ожидаемый отказ матрицы, не «потом в проде заработает».

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный SAIO | Нет отказа ЦОДа, нет Keystone, нет rsync между площадками |
| Дешево показывает враньё AWS SDK vs `s3_compat` | Успешный boto на localhost **не** доказывает прод |
| | tempauth приучает хранить секрет в конфиге |

Препрод: несколько дисков, **разные zone внутри одного ЦОДа**, Keystone, TLS, s3api, recon — даже без боевого трафика.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** пережить отказ **диска/ноды/proxy внутри ЦОДа**; пережить отказ **целого ЦОДа** ценой RPO>0 (container sync eventual **или** restore с бэкапа). Цифр ТБ нет.

### Почему не stretch

Дефолтная модель Swift — **один region с низкой задержкой**. Три ЦОДа как три zone одного кольца (вариант A) или три region Global Cluster с `write_affinity` (вариант B) заставляют PUT и/или rsync ходить через город. При неприемлемом ping это таймауты proxy, рост handoff и «копии ещё не там». Global Cluster **есть** форма stretch колец — здесь не используем. Цифры миллисекунд документация **не даёт**.

### Топология

**Внутри активного ЦОДа (ЦОД-1)** — один кластер (вариант D: «мозг в одном ЦОДе»):

- replica **3** — единственное **протестированное** значение Deployment Guide; меньше 3 на проде = вы сами испытательный стенд;
- zone = залы/стойки/питание **этого** ЦОДа (изоляция копий внутри площадки). Не region=чужой ЦОД;
- ≥2 proxy за локальным LB (HAProxy этого ЦОДа, см. `HAProxy.install.md` если берёте его). Один proxy = SPOF API при живых дисках;
- object/container/account на локальном XFS, `/srv/node/…` от **root** (`root:root 755`) — иначе при размонтировании rsync может писать в корневой том;
- rsync **внутренняя** сеть; клиентам только 443 → proxy (8080). Порты **6200/6201/6202** и **873** не публичны;
- одинаковые `*.ring.gz` на всех нодах; **`.builder` в бэкапе вне** единственной машины;
- memcached: полный список в `memcache_servers` на каждом proxy;
- Keystone + pipeline s3api **до** боевых бакетов.

`write_affinity` **не** включаем: документация — для workload «записал и не читаешь сразу из другого региона» (бэкапы). У микросервисов типично GET сразу после PUT.

**Между ЦОДами:**

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | ЦОД-1: единственное кольцо (HA внутри). ЦОД-2: **независимый** Swift (свои кольца, свой Keystone-проект) + **container sync** как DR **или** только бэкап объектов и `.builder` | API ЦОД-1 мёртв. Независимый кластер жив со своим (eventual) набором объектов. Без второго кластера — restore с бэкапа, RTO замерить |
| **3 ЦОДа** | ЦОД-1: активный. ЦОД-2: независимый + sync. ЦОД-3: третий независимый **или** только бэкапы | То же; три кольца = три операционки, не «один бакет на страну» |

Container sync — отдельная глава проекта (не «один endpoint, три VIP»). Это **не** догон партиций одного кольца. RPO > 0, конфликты last-write-wins. Не изобретаем расписание sync — сверять https://docs.openstack.org/swift/2026.1/overview_container_sync.html

Клиенты ЦОД-2/3 в штате: либо S3/Swift ЦОД-1 по городу (TLS), либо **локальный** независимый endpoint (другой бакет/аккаунт, пока sync не догнал).

### Предпосылки прода

- Выделенные storage-ноды, XFS, xattr, отдельные сети: клиентская / storage (620x) / replication (rsync).
- Keystone HA. Без него нет прода S3 (`s3token`).
- Балансировщик TCP/TLS перед proxy **этого** ЦОДа. VIP proxy в чужом ЦОДе = скрытый SPOF **этого** кластера.
- Kubernetes 1.36.4: proxy как Deployment за Service — нормально (stateless). Object-server в кольце по IP пода **без** hostNetwork/стабильного IP — нет. Helm OpenStack на 1.36 **не** считать сертифицированным путём.

### Установка (ЦОД-1, по Deployment Guide 2026.1)

Порядок официальный, не «скопируй ring-builder из блога 2016»:

1. Пакеты/образы **Swift 2.37.3** (дистрибутив 2026.1 или своя сборка). Не смешивать серии внутри одного кольца.
2. Диски XFS, точки `/srv/node/<device>`, rsyncd с модулями object/container/account.
3. Собрать **три** кольца (плюс object-кольцо на каждую extra policy). Отправная точка гайда:

```bash
swift-ring-builder account.builder create <part_power> 3 24
swift-ring-builder container.builder create <part_power> 3 24
swift-ring-builder object.builder create <part_power> 3 24
```

`3` — replica, `24` — пример `min_part_hours`. `<part_power>` — **не** выдуман в этой инструкции: по гайду от максимума дисков × ≥100 партиций на диск. Устройства добавлять с zone **внутри ЦОД-1**, вес ориентир гайда **100.0 × ТБ**. Затем один `rebalance`, раздать `.ring.gz` **всем** proxy и storage. Builder — в защищённый бэкап.

4. Запустить object/container/account, затем **несколько** proxy, memcached.
5. Keystone: `keystoneauth` + для S3 `s3token`. Pipeline прода (смысл, порядок сверять с `proxy-server.conf-sample` **2.37.3**): catch_errors / gatekeeper / healthcheck / cache / **authtoken** (`delay_auth_decision = True`) / **s3api** / **s3token** / **keystoneauth** / квоты / **bulk** / **slo** / proxy-server.
6. TLS на LB. Path-style S3 на первый контракт.
7. recon: заполнение дисков, async pending, время репликации, 5xx proxy.

Второй кластер в ЦОД-2 — **с нуля те же шаги**, другие кольца, другие IP. Затем container sync между контейнерами, которые решили зеркалировать. Не копировать `.ring.gz` ЦОД-1 на ноды ЦОД-2: это и есть попытка stretch.

### Конфигурирование

| Параметр | Прод | Зачем |
|---|---|---|
| replica | 3 | Единственное протестированное |
| RAID 5/6 под объект | нет | Вендор не рекомендует; надёжность — replica + auditor |
| tempauth | нет | Пароли в конфиге — не identity |
| `s3_acl` | решить сознательно | Дефолт проекта исторически `False`; часть CVE-2026-71191 завязана на него |
| Квоты | `account_quotas` / container quotas | Иначе озеро съест диски |
| At-rest encryption | по решению ИБ; секрет ≥44 символа base64, одинаковый на всех proxy; лучше Barbican | Имена объектов **не** шифруются |
| GeoData 2.29.2 | отдельный контур | Не этот кластер |

Один PUT по умолчанию до **5 ГБ**; больше — SLO/multipart.

### Масштабирование (когда появятся цифры)

1. Больше гигабайт → диски в object-кольцо, вес по ТБ, `rebalance`. Не «увеличить PVC одного пода».
2. Больше QPS → больше **proxy** (CPU/сеть). TLS часто снимают на LB.
3. Мелкие файлы / тяжёлый listing → шардинг контейнеров (отдельная дисциплина), не «как диск Postgres».
4. Холодный архив дешевле → **вторая** storage policy с EC на **новый** контейнер, не ломая default 3×.
5. Не забивать диски вплотную — некуда двигать партиции при rebalance.

### Проверка прода (пока это не пройдено — это не прод)

1. Версия proxy/storage = **2.37.3**.
2. PUT/GET Swift и S3 с ключами сервиса; без ключа — отказ. 6200 с клиентской сети не открыт.
3. Убить один диск и одну storage-ноду **внутри ЦОД-1**: GET жив, replicator сходится.
4. Падение одного proxy: LB отдаёт на живой.
5. Restore `.builder` + прогон GET на стенде. Если container sync — замерить лаг (это ваш RPO).
6. Учение «ЦОД-1 мёртв»: клиенты ЦОД-2 идут в локальный кластер или ждут restore — то, что обещали, не «как получится».

### Сильные / слабые стороны (кластер в одном ЦОДе + независимый/бэкап DR)

| Сильное | Слабое |
|---|---|
| Кольцо и rsync не ходят межплощадочно на пути PUT | Падение ЦОД-1 = нет **этого** S3, пока sync/restore |
| Согласовано с запретом stretch и с вариантами C/D исходного файла | Container sync eventual; три кластера = тройные ключи |
| replica=3 внутри площадки — штатный HA диска/ноды | Ошибка rebalance бьёт весь кластер ЦОД-1 |
| | OpenStack-Helm ≠ ваш Kubernetes 1.36.4 «из коробки» |

**Не готов к проду**, если: SAIO/tempauth; один proxy; replica≠3; NFS/`emptyDir` как диск object-server; кольца разъехались; нет бэкапа `.builder`; 6200 открыт клиентам; один ring на три ЦОДа (A/B); приложение требует S3 Lifecycle/bucket policy; кластер общий с GeoData 2.29.2; Swift назначен SoT карточек; Dev-конфиг скопирован в бой.

---

## Источники

- Релиз-ноты 2.37.3 / 2026.1: https://docs.openstack.org/releasenotes/swift/2026.1.html
- Deployment Guide: https://docs.openstack.org/swift/2026.1/deployment_guide.html
- SAIO: https://docs.openstack.org/swift/2026.1/development_saio.html
- Global Clusters (почему это stretch — и почему не берём): https://docs.openstack.org/swift/2026.1/overview_global_cluster.html
- Container sync: https://docs.openstack.org/swift/2026.1/overview_container_sync.html
- s3api / `s3_compat` / encryption / 5 ГБ: см. список в `OpenStack Swift.md`

Порога RTT для stretch колец в документации Swift **нет** — поэтому stretch в этой инструкции не предлагается.
