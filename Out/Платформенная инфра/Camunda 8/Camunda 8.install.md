# Camunda 8.9 — установка (учебный контур)

Связанные документы: правила — `Camunda 8.md`; словарь — `Camunda 8.info.md`; схема платформы — `Camunda 8.shema.md`.

Ставите **свою** установку Self-Managed, не облако Camunda Cloud и не линейку 7. Артефакт: **Camunda 8 Run 8.9.17**. Поставщик: это локальная сборка для разработки и прототипа, **не для production**. Вторичное хранилище по умолчанию — **H2** (встроенная файловая СУБД).

**Docker** — программа, которая запускает готовый **образ** (упакованная программа с зависимостями). **Helm** на Kubernetes в этот учебный файл не входит.

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** Одна машина в **закрытой** сети: ваш ноутбук или учебная VM. C8 Run — процесс на этой машине: Orchestration Cluster (движок Zeebe + Operate + Tasklist + Admin) и Connectors. Windows и macOS как хост — только учёба. На Ubuntu вендор просит **22.04 или новее**.

Клиенты ходят на **gateway**: REST **8080**, gRPC **26500**. Порты **26501** (команды gateway→брокер) и **26502** (Gossip + Raft) наружу не публиковать. **9600** — метрики, не в интернет. **8086** — Connectors API.

**Сколько железа.** Цифры «хватит N ядер для C8 Run» в мануале **нет** — не подставлять. Troubleshooting C8 Run: ориентир **8 ГБ RAM**; при нехватке памяти — `JAVA_OPTS=-Xmx4g`. Диск — **локальный** (H2 пишет файлы рядом с установкой). Порог **≥ 1000 IOPS** со страницы поддерживаемых сред относится к диску **брокера** в нормальной установке, не к ноутбуку с H2.

| Зачем цифра | Что взять | Откуда |
|---|---|---|
| Учебный C8 Run | 8 ГБ RAM как ориентир; локальный диск | troubleshooting C8 Run |
| Ядра / смета боя | нет | в доке вендора для C8 Run этого нет |
| Диск брокера (≥1000 IOPS, SSD) | не этот стенд | [supported environments](https://docs.camunda.io/docs/reference/supported-environments/) |

**Сильная сторона:** совпадает с официальным C8 Run, поднимается за минуты, учит воркера **снаружи** JVM движка.  
**Слабая сторона:** один процесс, H2, нет Raft между машинами, нет экспорта в OpenSearch 3.8.0.

**Критично:** `8080`/`26500` в интернет не открывать. Учётка `demo`/`demo` — только закрытый стенд. Не `latest`. Один C8 Run ≠ кластер брокеров. H2 и C8 Run в production не копировать.

---

## Установка для новичка

Официальные шаги: [Install and start Camunda 8 Run](https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/install-start/). Патч **8.9.17** берём с [GitHub Release 8.9.17](https://github.com/camunda/camunda/releases/tag/8.9.17), не «latest» со страницы, где имя файла без патча.

В архиве есть встроенная Java. Системный JDK не обязателен; если встроенный runtime сломан — fallback **OpenJDK 21–25**.

### Что должно быть до установки

Есть:

- Закрытая сеть; 8080, 26500, 8086, 9600 свободны на машине.
- [Desktop Modeler](https://docs.camunda.io/docs/components/modeler/desktop-modeler/install-the-modeler/) **5.46+** (рисовать и деплоить BPMN).
- Для воркера на Java: JDK **17+** (Spring Boot Starter) или **8+** (Java client) — это **ваш** микросервис, не брокер.

Нет (и не должно появиться):

- Публикация 8080/26500/26501/26502/9600 в интернет.
- Привычка «C8 Run = почти production».
- Образ/тег `latest`.
- Схемы с XML-досье в переменных процесса.

### Этап 1. Скачать C8 Run 8.9.17

**Что делаем:** качаем архив под вашу ОС и распаковываем. Переходим в каталог `c8run`, как в гайде вендора.

Linux x86_64:

```bash
curl -LO https://github.com/camunda/camunda/releases/download/8.9.17/camunda8-run-8.9.17-linux-x86_64.tar.gz
tar -xzf camunda8-run-8.9.17-linux-x86_64.tar.gz
cd c8run
```

Windows x86_64: скачать `camunda8-run-8.9.17-windows-x86_64.zip` с того же [релиза 8.9.17](https://github.com/camunda/camunda/releases/tag/8.9.17), распаковать, открыть каталог `c8run`.

macOS: `camunda8-run-8.9.17-darwin-aarch64.zip` (Apple Silicon) или `camunda8-run-8.9.17-darwin-x86_64.zip`.

Успех: в каталоге есть `c8run` / `c8run.exe` и `start.sh` (Linux/macOS).

### Этап 2. Запустить

**Что делаем:** стартуем Orchestration Cluster и Connectors. Если старт сорвался — сначала остановка, потом снова старт (так вендор).

Linux / macOS:

```bash
./c8run start
# иначе:
./start.sh
```

Windows:

```powershell
.\c8run.exe start
```

Остановка: `./shutdown.sh` или `./c8run stop` (Linux/macOS); `.\c8run.exe stop` (Windows).

Успех: браузер сам открывает Operate **или** открывается http://localhost:8080/operate. Логи: `c8run/log/camunda.log`.

### Этап 3. Проверить, что движок отвечает

**Что делаем:** спрашиваем топологию кластера по REST. По умолчанию C8 Run **не** требует пароль на API.

```bash
curl http://localhost:8080/v2/topology
```

Успех: JSON с брокерами, не ошибка соединения. Версия линейки **8.9.x**.

Если 8080 занят: `./c8run start --port 8081` (тогда Operate — `http://localhost:8081/operate`).

### Этап 4. Процесс с одной сервисной задачей и воркер снаружи

**Что делаем:** в Desktop Modeler рисуем BPMN с одной **сервисной задачей** (тип работы, например `fetch-client-from-lake`). Деплоим на локальный C8 Run (готовое соединение `c8run (local)` в стартовом пакете; иначе REST `http://localhost:8080` / gRPC `localhost:26500`). Стартуем экземпляр. В переменную — идентификатор, не досье (потолок вендора порядка **~3 МБ** на экземпляр).

Воркер — **отдельная** программа (не метод внутри JVM движка). Для Java на этом стенде API по умолчанию открыт:

```yaml
camunda:
  client:
    mode: self-managed
    auth:
      method: none
    grpc-address: http://localhost:26500
    rest-address: http://localhost:8080
```

Адреса — абсолютный URI (`http://…`). Стартер: `io.camunda:camunda-spring-boot-starter` **8.9.x**, JDK 17+, Spring Boot 3.5.x. Метод с `@JobWorker(type = "fetch-client-from-lake")` забирает работу, ходит в **ваши** сервисы, отвечает `complete` / `fail`.

Готовый пример: [getting started](https://docs.camunda.io/docs/guides/getting-started-example/) (`mvn spring-boot:run` в каталоге Java-воркеров).

Успех: экземпляр в Operate доходит до конца. Пауза секунды–минуты, пока экспортёр догонит экран — нормально.

### Docker Compose — второй официальный путь учёбы

Если на машине уже Docker и нужен стек контейнеров, а не C8 Run: архив [docker-compose-8.9](https://github.com/camunda/camunda-distributions/releases/tag/docker-compose-8.9) (`CAMUNDA_VERSION=8.9.17`). Нужны Docker **≥ 20.10.16** и Compose v2 **≥ 2.24.0** (`docker compose version`, два слова). Распаковать целиком (включая `.env`) и из каталога:

```bash
docker compose up -d
docker compose ps
```

Это **lightweight** `docker-compose.yaml`: Orchestration Cluster + Connectors, H2. Те же URL **8080** / **26500**, учётка UI `demo`/`demo`, API по умолчанию открыт. Поставщик: Compose-файлы не для production. Полный `docker-compose-full.yaml` (Keycloak, Management Identity, Optimize) для первого знакомства с воркером не нужен.

### Чего этот стенд не доказывает

- Отказ машины, зала, выборы лидера Raft, кворум 3/3/3.
- Экспорт в OpenSearch 3.8.0 и backpressure, когда поиск лежит.
- Нагрузку, ёмкость партиций, согласованный backup API.
- OIDC, лицензию production (`CAMUNDA_LICENSE_KEY` с 8.6), TLS на входе.

---

## Первый запуск — URL, порт, учётка, смена пароля

C8 Run и lightweight Compose: Operate, Tasklist и **Admin** (с 8.9 так называется бывший Identity Orchestration Cluster) сидят на одном **8080**. Отдельный Management Identity (**8084**, Web Modeler / Console / Optimize) в C8 Run **нет**.

| Что | URL / порт | Кто ходит |
|---|---|---|
| Operate (где застрял процесс) | http://localhost:8080/operate | браузер |
| Tasklist (человеческие задачи) | http://localhost:8080/tasklist | браузер |
| Admin (пользователи, права кластера) | http://localhost:8080/admin | браузер |
| REST API | http://localhost:8080/v2 | клиенты, деплой |
| gRPC (Zeebe) | localhost:26500 | job workers |
| Connectors | 8086 | служебный |
| Метрики | http://localhost:9600/actuator/prometheus | не в интернет |
| 26501 / 26502 | внутренние | не публиковать |

**Учётка UI (только закрытый стенд):** `demo` / `demo`. Так в troubleshooting C8 Run и в гайде Compose.

**API C8 Run / lightweight Compose** по умолчанию **без пароля**, проверки прав выключены. Это удобно отлаживать воркер и опасно, если 8080 виден с чужой машины.

Сменить пароль `demo` (Basic, не OIDC):

1. Войти в Admin → Users → карандаш у пользователя → поле **Password** → Save. Страница: [Users](https://docs.camunda.io/docs/components/admin/user/).
2. Либо при старте задать первого пользователя: `./c8run start --username … --password …` (Windows: `.\c8run.exe start --username … --password …`).
3. Либо YAML `camunda.security.initialization.users` и `./start.sh --config application.yaml`. Чтобы API тоже требовал пароль: `unprotected-api: false`, `authorizations.enabled: true` — см. [configure C8 Run](https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/configuration/). Тогда `curl` так: `curl -u demo:demo http://localhost:8080/v2/topology`.

В бою эту учётку не оставляют: там корпоративный IdP, не `demo`.

---

## Подключение к своей системе

Протокол к движку: **gRPC 26500** и **REST 8080** на **gateway**. Клиенты не ходят на брокеры как на балансировщик Kafka и не открывают 26501/26502. Балансировщик на 26500 должен понимать **долгие** потоки воркеров (обычный HTTP L7 рвёт стримы) — на localhost C8 Run это не проявляется.

В этой платформе воркер — ваш микросервис (часто Spring). Он:

- подписывается на **job type** из BPMN;
- ходит в **Kafka** (шина фактов), озеро / PostgreSQL (эталон карточки), **интеграционное API** (ведомства);
- возвращает в процесс id и статус, не XML досье.

BPMN — в git. Секреты (пароль UI, ключи коннекторов, потом лицензия и OIDC) — не в git и не в схеме процесса. На C8 Run секреты коннекторов — переменные окружения; в Compose — файл `connector-secrets.txt`, не YAML приложения.

Клиент воркера совместим с **8.9** (`camunda-spring-boot-starter` / `camunda-client-java` 8.9.x). Для C8 Run по умолчанию `auth.method: none`. Если включили Basic — username/password в секрет приложения, не в репозиторий.

Camunda **не**:

| Сосед | Чем отличается |
|---|---|
| Kafka | Шина событий; Camunda хранит *состояние долгого процесса* |
| PostgreSQL / озеро | Эталон карточки; в переменных процесса — id/статус |
| Интеграционное API | Единственный выход в ведомства; коннекторы Camunda туда не плодить |
| GeoData workflow | Чужой движок и своя Kafka; роли не смешивать |
| Camunda 7 | Движок в той же JVM (`JavaDelegate`); в 8 движок всегда удалённый |

---

## Ссылки на материал

| Факт | URL |
|---|---|
| C8 Run не production; состав (кластер + Connectors + H2) | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/ |
| Скачать, `./c8run start`, Operate http://localhost:8080/operate | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/install-start/ |
| `--username` / `--password` / `--port`; закрыть API; метрики :9600 | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/configuration/ |
| Порты 8080, 26500, 8086, 9600; `demo`/`demo`; 8 ГБ RAM | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run-troubleshooting/ |
| Compose не production; `docker compose up -d`; Docker ≥ 20.10.16, Compose ≥ 2.24.0 | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/docker-compose/ и https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/docker-compose/install-start/ |
| URL Operate/Tasklist/Admin 8080; gRPC 26500; `demo`/`demo`; API lightweight открыт | https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/docker-compose/configuration/ |
| Архив Compose, `CAMUNDA_VERSION=8.9.17` | https://github.com/camunda/camunda-distributions/releases/tag/docker-compose-8.9 |
| C8 Run 8.9.17 (архивы linux/windows/darwin) | https://github.com/camunda/camunda/releases/tag/8.9.17 |
| Чарт 14.8.5, образ `camunda/camunda:8.9.17` | https://helm.camunda.io/camunda-platform/version-matrix/camunda-8.9/ |
| OpenJDK брокера 21–25; OpenSearch 3.6+; диск брокера ≥1000 IOPS; Desktop Modeler 5.46+ | https://docs.camunda.io/docs/reference/supported-environments/ |
| REST 8080, gRPC 26500, внутренние 26501/26502, 9600 | https://docs.camunda.io/docs/self-managed/components/orchestration-cluster/zeebe/operations/network-ports/ |
| Смена пароля пользователя в Admin (Basic) | https://docs.camunda.io/docs/components/admin/user/ |
| Admin = бывший Identity кластера (пути `/admin`) | https://docs.camunda.io/docs/reference/announcements-release-notes/890/whats-new-in-89/ |
| Воркер Spring, `mode: self-managed`, gRPC/REST | https://docs.camunda.io/docs/apis-tools/camunda-spring-boot-starter/getting-started/ |
| Java client: `http://localhost:26500` и `http://localhost:8080` | https://docs.camunda.io/docs/apis-tools/java-client/getting-started/ |
| Первый процесс + воркер снаружи | https://docs.camunda.io/docs/guides/getting-started-example/ |
| Docker-образы vs Compose (Compose = quick start) | https://docs.camunda.io/docs/self-managed/deployment/docker/docker/ |
| Зачем продукт, порты, железо | `Camunda 8.md` |
| Словарь (gateway, job worker, H2, Admin ≠ Management Identity) | `Camunda 8.info.md` |
| Стыковка с Kafka, озером, интеграционным API | `Camunda 8.shema.md` |
