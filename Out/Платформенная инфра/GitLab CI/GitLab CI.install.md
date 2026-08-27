# GitLab 19.3.0 — установка (учебный контур)

Отдельного пакета «только CI» нет. Ставите **координатор** (сам GitLab: репозитории, очередь работ, UI/API) и отдельно **GitLab Runner** (агент, который спрашивает «есть работа?» и исполняет её). Это **self-managed**, не GitLab.com.

Версии: **GitLab 19.3.0**, **GitLab Runner 19.3.0**. Пакет Linux (Omnibus) `gitlab-ee=19.3.0-ee.0` (без лицензии это функции Free). Если в репозитории уже есть security-патч **19.3.x** — берите его, не `latest` и не прыжок на 19.4.

**Допущение:** закрытая сеть, **одна** Linux-машина, встроенные PostgreSQL и Redis. Учебный запуск в промышленный контур не копировать. Дефолтный Helm `gitlab/gitlab` сюда не входит (вендор: *not intended for production*).

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна виртуальная машина **Linux x86_64**. Официальный список ОС пакета: в том числе **Ubuntu 24.04 / 22.04**, Debian 12/13, RHEL 8/9/10. В этой инструкции команды — **Ubuntu 24.04** ([Install on Ubuntu](https://docs.gitlab.com/install/package/ubuntu/), [supported platforms](https://docs.gitlab.com/install/package/)). Windows как хост GitLab/Gitaly не предполагается.

На этой же машине (или соседней) — Runner. Страница регистрации Runner просит **отдельный** сервер от GitLab; на учёбе один хост допустим, если хватает RAM. Kubernetes на этом стенде **не** нужен.

```mermaid
flowchart LR
  DEV["git / браузер"] -->|"443 / 22"| GL["GitLab 19.3\nкоординатор"]
  RN["Runner 19.3"] -->|"опрос API 443"| GL
  GL -->|"job"| RN
```

**Сколько.** Цифр вашей нагрузки нет — нет «хватит N ядер на терабайты». Путать «чтобы процесс встал» и смету промышленного контура нельзя. Swap в референсах **не рекомендуется**.

| Зачем | CPU | RAM | Диск | Откуда |
|---|---|---|---|---|
| Single-node, базовая линия | **8 vCPU** | **16 ГБ** | сумма строк ниже | [Installation requirements](https://docs.gitlab.com/install/requirements/) |
| «Память тесна», тот же single-node | — | **≥ 8 ГБ** | — | та же страница, memory-constrained |
| Каталог приложения (пакет ~2,5 ГБ + ОС, логи) | — | — | **40 ГБ** | та же страница |
| Репозитории (Gitaly) | — | — | не меньше, чем сами репо; SSD, не burstable, не NFS | та же страница |
| Встроенная PostgreSQL | — | — | **5–12 ГБ** | та же страница |

Для учёбы берите **8 vCPU / 16 ГБ / ≥ 40 ГБ** локального диска под `/var/opt/gitlab`. Отдельной таблицы «минимум ядер на один echo-job Runner» у вендора нет — не выдумывать.

**Сильная сторона:** официальный Linux package, один хост, встроенные Postgres/Redis. **Слабая:** падение этой VM = нет Git, нет очереди работ, нет UI.

**Критично:** **443 / 80 / 22** — с jump-хоста или VPN, не в интернет без TLS и своего пароля. **8075** (Gitaly), **5432**, **6379** снаружи не публиковать (на Omnibus Postgres/Redis по умолчанию через сокет). Не `latest`. Let’s Encrypt на закрытой сети **не** заработает (нужен вход с серверов LE на 80/443). Один Omnibus — не кластер и не Geo.

---

## Установка для новичка

Команды — **на Linux-машине стенда**, не в PowerShell. Нужен `sudo`.

Страницы шагов: [Ubuntu package](https://docs.gitlab.com/install/package/ubuntu/), [пин версии](https://docs.gitlab.com/update/package/), [HTTPS вручную](https://docs.gitlab.com/omnibus/settings/ssl/), [Runner из репозитория](https://docs.gitlab.com/runner/install/linux-repository/), [регистрация `glrt-`](https://docs.gitlab.com/runner/register/).

**Docker** — программа, которая запускает **образ** (упакованная программа с зависимостями) как **контейнер**. Он нужен только для executor `docker`, не для самого пакета GitLab.

### Что должно быть до установки

Есть:

- Ubuntu 24.04 (или другая ОС из таблицы пакета), x86_64, NTP.
- DNS или `/etc/hosts`: имя, которое станет `external_url` (ниже `gitlab.example.local`), указывает на эту VM.
- Доступ к `https://packages.gitlab.com/*` и `https://storage.googleapis.com/packages-ops/*` **или** заранее скачанные `.deb` тех же версий ([firewall prefixes](https://docs.gitlab.com/install/package/ubuntu/), [offline](https://docs.gitlab.com/topics/offline/quick_start_guide/)).
- Свободны на хосте **80**, **443**, **22**.
- Свой пароль `root` (≥ 8 символов). Не `5iveL!fe`.

Нет (и не должно появиться на этой VM):

- Публикация 443/22 в интернет.
- Let’s Encrypt (закрытая сеть).
- Legacy registration token Runner.
- `privileged: true` / Docker-in-Docker.
- Второго Ingress/nginx на 80/443.

### Этап 1. Проверяем машину

**Что делаем:** убеждаемся, что ОС и место подходят.

```bash
uname -m
lsb_release -a
nproc
free -h
df -h
```

Успех: `x86_64`; Ubuntu 24.04 (или другая ОС из [таблицы пакета](https://docs.gitlab.com/install/package/)); RAM ≥ 8 ГБ (лучше 16); свободно ≥ 40 ГБ.

### Этап 2. SSH и порты

**Что делаем:** включаем SSH и открываем 22/80/443 **в сеть стенда**, не в мир.

```bash
sudo systemctl enable --now ssh
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Успех: `sudo ufw status` показывает 22, 80, 443. Команды — с [Install on Ubuntu](https://docs.gitlab.com/install/package/ubuntu/).

### Этап 3. Репозиторий пакетов GitLab

**Что делаем:** добавляем официальный репозиторий **координатора** (это ещё не Runner). Скрипт можно сначала открыть в браузере.

```bash
sudo apt update
sudo apt install -y curl
curl --location "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh" | sudo bash
apt-cache madison gitlab-ee | head
```

Успех: в `madison` видна строка **19.3.0**. Если сети до packages.gitlab.com нет — ставьте скачанный `gitlab-ee_19.3.0-ee.0_amd64.deb` через `dpkg -i` ([downloaded package](https://docs.gitlab.com/update/package/)).

Только CE: репозиторий `gitlab/gitlab-ce`, пакет `gitlab-ce=19.3.0-ce.0`.

### Этап 4. Ставим координатор 19.3.0

**Что делаем:** ставим **конкретную** версию. `EXTERNAL_URL` пишется в `/etc/gitlab/gitlab.rb`. На закрытой сети сначала **http**, иначе пакет сам запросит Let’s Encrypt и упадёт.

```bash
sudo GITLAB_ROOT_PASSWORD='ВАШ_ПАРОЛЬ_≥8' \
  EXTERNAL_URL="http://gitlab.example.local" \
  apt install gitlab-ee=19.3.0-ee.0
```

Пин версии: `gitlab-ee=<version>-ee.0` ([Upgrade with the Linux package](https://docs.gitlab.com/update/package/)). Первая установка идёт минуты: пакет поднимает встроенные PostgreSQL, Redis, Gitaly, Puma, Sidekiq, NGINX.

Если hostname пакет не принял и `reconfigure` не запустился — тот же пароль передайте в **первый** `sudo GITLAB_ROOT_PASSWORD='…' gitlab-ctl reconfigure`. Повторный reconfigure пароль из переменной **не** меняет.

Успех:

```bash
sudo gitlab-ctl status
sudo gitlab-rake gitlab:env:info | grep -E 'GitLab.*19\.3'
```

`gitlab-ctl status` — все основные сервисы `run` ([maintenance](https://docs.gitlab.com/omnibus/maintenance/)). В `env:info` — линия **19.3**.

### Этап 5. HTTPS своим сертификатом

**Что делаем:** Let’s Encrypt на закрытой сети не используем. Кладём сертификат внутреннего CA (или самоподписанный) под **то же DNS-имя**, что в `external_url`. Имена файлов = hostname.

```bash
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 755 /etc/gitlab/ssl
# скопируйте цепочку: сервер → промежуточные → root CA
sudo cp gitlab.example.local.crt gitlab.example.local.key /etc/gitlab/ssl/
sudo chmod 644 /etc/gitlab/ssl/gitlab.example.local.crt
sudo chmod 600 /etc/gitlab/ssl/gitlab.example.local.key
```

В `/etc/gitlab/gitlab.rb`:

```ruby
external_url "https://gitlab.example.local"
letsencrypt['enable'] = false
gitlab_rails['nginx']['redirect_http_to_https'] = true
```

Ключ `gitlab_rails['nginx']` — для GitLab **19.2+** ([SSL](https://docs.gitlab.com/omnibus/settings/ssl/)). Самоподписанный ключ/crt официально генерируют так ([offline](https://docs.gitlab.com/topics/offline/quick_start_guide/)):

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/gitlab/ssl/gitlab.example.local.key \
  -out /etc/gitlab/ssl/gitlab.example.local.crt
```

```bash
sudo gitlab-ctl reconfigure
curl -kI https://gitlab.example.local | head -1
```

Успех: HTTP 302 или 200 на `https://gitlab.example.local`. Браузер/git/Runner должны **доверять** этому CA, иначе clone и регистрация Runner сломаются на TLS.

### Этап 6. Docker на хосте Runner

**Что делаем:** для executor `docker` ставим Docker Engine (свой пакет/зеркало контура; не цель этой страницы — версия Engine). Если образы с Docker Hub не тянутся — этот этап пропустите и на этапе 8 берите executor **`shell`**.

```bash
docker version
```

Успех: клиент и демон отвечают. `privileged` **не** включаем.

### Этап 7. Ставим GitLab Runner 19.3.0

**Что делаем:** это **другой** пакет и **другой** репозиторий, не `gitlab-ee`. Пин **той же** minor 19.3. С 17.7.1 вместе с конкретной версией ставят `gitlab-runner-helper-images` той же строки.

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" -o script.deb.sh
less script.deb.sh
sudo bash script.deb.sh
apt-cache madison gitlab-runner | head
sudo apt install gitlab-runner=19.3.0-1 gitlab-runner-helper-images=19.3.0-1
gitlab-runner --version
```

Точную строку пакета смотрите в `madison` (шаблон вендора: `17.7.1-1` → для 19.3.0 ожидайте `19.3.0-1`). Успех: `gitlab-runner --version` печатает **19.3.x**. Страница: [linux-repository](https://docs.gitlab.com/runner/install/linux-repository/).

### Этап 8. Создаём Runner в UI и регистрируем `glrt-`

**Что делаем:** токен **`glrt-`** выдаёт GitLab при создании Runner в UI. Legacy registration token не включать (снятие в **20.0**).

1. Войти как `root` (следующий раздел).
2. **Admin → CI/CD → Runners → Create instance runner**.
3. ОС Linux; теги или **Run untagged**; **Create runner**.
4. Скопировать authentication token (`glrt-…`). Он на экране недолго; после регистрации живёт в `/etc/gitlab-runner/config.toml`.

На машине Runner:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.local/" \
  --token "glrt-…" \
  --executor "docker" \
  --docker-image "alpine:3.21" \
  --docker-privileged=false \
  --description "stand-docker"
```

Если нет Docker/образов — `--executor "shell"` без `--docker-*`. Shell: работы видят друг друга на одной ОС — только стенд.

Kubernetes-executor — только если кластер уже есть; чарт `gitlab/gitlab-runner` пинить по **APP VERSION = 19.3.0**, не по последнему тегу чарта.

Успех: Admin → Runners показывает зелёный Runner **19.3.x**; в `config.toml` `token = "glrt-…"` и `privileged = false`. Страницы: [Create instance runner](https://docs.gitlab.com/ci/runners/runners_scope/), [register](https://docs.gitlab.com/runner/register/), [new workflow](https://docs.gitlab.com/ci/runners/new_creation_workflow/).

### Этап 9. Один pipeline

**Что делаем:** создаём проект, кладём `.gitlab-ci.yml`, push. Работа клонирует репозиторий **сама** (через координатор/Gitaly), руками `git clone` внутри YAML для своего же проекта не нужен.

```yaml
stages: [test]
hello:
  stage: test
  image: alpine:3.21
  script:
    - echo "stand-ok"
```

Для `shell` строку `image:` уберите.

Успех: pipeline зелёный; лог виден после завершения. Остановите Runner (`sudo gitlab-runner stop`) — UI жив, новые работы в **`pending`**. Снова `sudo gitlab-runner start`.

Чего этот стенд **ещё не доказывает:** отказ зала, Praefect/Geo, Hybrid Helm, внешние Postgres/Redis/бакет, нагрузку RPS, clone монорепозитория, BuildKit, выборы лидера, переживание падения координатора живым Runner.

---

## Первый запуск — URL, порт, учётка, смена пароля

**URL и порт.** `https://gitlab.example.local/` — порт **443** (из `external_url`). Git SSH — **22**. Открывайте с jump/VPN.

Страница входа: [Initial sign-in](https://docs.gitlab.com/install/package/ubuntu/#initial-sign-in).

**Учётка.** Пользователь **`root`**. Пароль:

1. Если задали `GITLAB_ROOT_PASSWORD` при `apt install` — этот пароль (минимум 8 символов).
2. Иначе случайный пароль пакета:

```bash
sudo cat /etc/gitlab/initial_root_password
```

Файл **`/etc/gitlab/initial_root_password`** — штатный путь **Linux package (Omnibus)** на этой установке. Вендор хранит его **не меньше 24 часов**; после 24 часов первый `gitlab-ctl reconfigure` **удаляет** файл. Не в git и не в чат.

В `gitlab.rb` поле `gitlab_rails['initial_root_password']` вендор **не** рекомендует: пароль открытым текстом.

**Смена пароля.** Сразу после входа:

1. Аватар → **Edit profile**.
2. **Access → Password and authentication** → **Change password**.
3. Текущий пароль → новый (≥ 8, не слабый словарьный) → **Save password**.

Страница: [Change your password](https://docs.gitlab.com/user/profile/user_passwords/). Учебный пароль в промышленный контур не копируют. Забыли — сброс с хоста: [Reset user passwords](https://docs.gitlab.com/security/reset_user_password/) (`gitlab-rake` / Rails, пользователь `root`).

Смените и **email** у `root` (при установке без `GITLAB_ROOT_EMAIL` почта случайная).

---

## Подключение к своей системе

Репозитории и CI — **этот** GitLab. Сервисы получают код и пайплайны отсюда. Runner ходит на **URL координатора** (443). SonarQube/сканеры — job в `.gitlab-ci.yml` с токенами. Backstage читает catalog из git. Исходящие к ведомствам — через **ваше** интеграционное API, не из GitLab напрямую.

### Протокол

| Кто | Как |
|---|---|
| Разработчик, clone/push | **HTTPS 443**: `git clone https://gitlab.example.local/<group>/<project>.git`. Пароль git = **personal access token** (scope `read_repository` / `write_repository`), не пароль UI. Имя пользователя — любая непустая строка. Либо **SSH 22**. |
| Job CI, clone своего/чужого проекта | HTTPS: пользователь `gitlab-ci-token`, пароль **`$CI_JOB_TOKEN`**. Живёт только пока идёт работа; в логах маскируется. Чужой проект — только если он в **allowlist** job token. |
| Runner manager | HTTPS **443** к API GitLab (`--url`). Токен **`glrt-`**. Не порт Gitaly 8075. |

```bash
git clone https://<username>@gitlab.example.local/<group>/<project>.git
# пароль = PAT, не пароль root

# внутри job (обычно делает helper сам):
# git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.local/<group>/<project>
```

Страницы: [PAT + Git over HTTPS](https://docs.gitlab.com/user/profile/personal_access_tokens/), [CI_JOB_TOKEN clone](https://docs.gitlab.com/ci/jobs/ci_job_token/).

### Что класть в секрет (не в git)

| Секрет | Где | Куда не класть |
|---|---|---|
| Пароль `root` | сейф / Vault | git, чат, `gitlab.rb` |
| `/etc/gitlab/initial_root_password` | только диск VM, 24 ч | git; файл исчезнет |
| PAT разработчика | сейф пользователя | `.gitlab-ci.yml`, логи |
| `glrt-` | `/etc/gitlab-runner/config.toml` (права root) | git, Confluence, образ |
| `$CI_JOB_TOKEN` | выдаёт GitLab на время job | переменные проекта «навсегда», чужой лог |

`CI_JOB_TOKEN` **не** заводят руками и **не** кладут в CI/CD variables. Allowlist: **Settings → CI/CD → Job token permissions**.

### Чем продукт не является

| Ожидание | Чем отличается |
|---|---|
| GitLab.com | Чужой SaaS; у вас свой URL и свои Runner |
| «Только CI» / Jenkins | CI встроен в GitLab; без координатора Runner не к чему регистрироваться |
| Legacy registration token | Кто украл — регистрирует любой Runner; план снятия **20.0**. Только `glrt-` |
| `CI_JOB_TOKEN` = PAT | Job-токен короткоживущий и урезанный; PAT — у человека |
| Kafka / Camunda / озеро | GitLab их runtime не держит. Падение GitLab не должно ронять уже работающие сервисы |
| Один Gitaly на 2–3 ЦОДа | Порт **8075** между площадками как «кластер» вендор так не описывает (HA **< 5 мс**, Praefect — *single location*) |
| Kaniko как стратегия сборки | Репозиторий Google архивирован; на стенде сборку образа не учим через privileged DinD |

На учебном стенде прямой HTTP ещё возможен до этапа 5 — после HTTPS клиенты без доверия к CA не клонируют. В промышленном контуре HTTP как норма не оставляют.

---

## Ссылки на материал

Факты в этом файле — со **страниц** ниже, не «из документации вообще».

| Факт | URL |
|---|---|
| Релиз GitLab 19.3 / Runner 19.3 (20 августа 2026) | https://docs.gitlab.com/releases/19/gitlab-19-3-released/ |
| Пакет Ubuntu: репозиторий, `EXTERNAL_URL`, `GITLAB_ROOT_PASSWORD`, вход `root`, файл `/etc/gitlab/initial_root_password` (24 ч) | https://docs.gitlab.com/install/package/ubuntu/ |
| Поддерживаемые ОС пакета | https://docs.gitlab.com/install/package/ |
| 8 vCPU / 16 ГБ baseline, ≥ 8 ГБ constrained, диск 40 ГБ, PG 17.x для GitLab 19.x, swap не рекомендуется, HA **< 5 мс**, не cross-region | https://docs.gitlab.com/install/requirements/ |
| Пин `gitlab-ee=<version>-ee.0` | https://docs.gitlab.com/update/package/ |
| HTTPS вручную, `letsencrypt['enable'] = false`, файлы `/etc/gitlab/ssl/<hostname>.crt\|key`, ключ `gitlab_rails['nginx']` с 19.2 | https://docs.gitlab.com/omnibus/settings/ssl/ |
| Offline: сначала `http` `EXTERNAL_URL`, затем свой crt | https://docs.gitlab.com/topics/offline/quick_start_guide/ |
| `gitlab-ctl status`, `gitlab-rake` | https://docs.gitlab.com/omnibus/maintenance/ |
| `gitlab-rake gitlab:env:info` | https://docs.gitlab.com/administration/raketasks/maintenance/ |
| Смена своего пароля в UI | https://docs.gitlab.com/user/profile/user_passwords/ |
| Сброс пароля `root` с хоста | https://docs.gitlab.com/security/reset_user_password/ |
| Порты пакета (443/80, 22, 8075, 5432, 6379, …) | https://docs.gitlab.com/administration/package_information/defaults/ |
| Runner: репозиторий, пин версии + `gitlab-runner-helper-images` | https://docs.gitlab.com/runner/install/linux-repository/ |
| `gitlab-runner register --token`, префикс `glrt-` | https://docs.gitlab.com/runner/register/ |
| Создание instance runner в Admin, `glrt-` | https://docs.gitlab.com/ci/runners/runners_scope/ |
| Новый workflow, снятие registration token в 20.0 | https://docs.gitlab.com/ci/runners/new_creation_workflow/ |
| `CI_JOB_TOKEN`, clone `gitlab-ci-token`, allowlist | https://docs.gitlab.com/ci/jobs/ci_job_token/ |
| Git HTTPS + PAT | https://docs.gitlab.com/user/profile/personal_access_tokens/ |
| `privileged = false` у Docker executor (дефолт) | https://docs.gitlab.com/runner/configuration/advanced-configuration/ |
| Privileged = снять изоляцию контейнера | https://docs.gitlab.com/runner/security/ |
| Дефолтный Helm ≠ прод | https://docs.gitlab.com/charts/ |
| Чарт 10.3.0 → GitLab 19.3.0 | https://docs.gitlab.com/charts/installation/version_mappings/ |
| Референс-архитектуры | https://docs.gitlab.com/administration/reference_architectures/ |
| Правила, порты, роль в платформе | `GitLab CI.md` |
| Словарь | `GitLab CI.info.md` |
| Схемы стыковки | `GitLab CI.shema.md` |
| Роль консультанта | `GitLab CI.consultant.md` |

В доке вендора **нет**: ядер на один учебный Runner; порога RTT «между вашими тремя ЦОДами»; обещания, что Runner переживёт мёртвый GitLab. Поэтому в этом файле их нет.

Строка пакета Runner `19.3.0-1` — шаблон с [linux-repository](https://docs.gitlab.com/runner/install/linux-repository/); фактический суффикс проверяйте `apt-cache madison gitlab-runner` в день установки. Тег образа job `alpine:3.21` — не версия GitLab; на закрытом контуре замените образом из **вашего** registry.
