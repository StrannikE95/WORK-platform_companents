# Spring Cloud Gateway 5.0.3 — установка (учебный контур)

**Допущение:** закрытый учебный стенд, один процесс на машине разработчика, HTTP без TLS. Это **библиотека** в вашем Spring Boot-приложении, не готовый образ и не Ingress. Настройки отсюда в бой не копировать.

Ставите: стартер `spring-cloud-starter-gateway-server-webflux` из поезда **Spring Cloud 2025.1.3** (Oakwood) на **Spring Boot 4.0.8**. В дереве зависимостей должна оказаться Gateway **5.0.3** — в ней закрыт **CVE-2026-47879**. Версию стартера не пишите сами: её даёт BOM. **5.0.2 и ниже этой линии без патча не брать.**

Официального образа Gateway на Docker Hub нет. **Docker** — программа, которая запускает **образ** (готовая упаковка программы с зависимостями). Вендор такой упаковки не публикует: собираете свой JAR (и свой образ, если понадобится).

---

## Куда ставить и сколько ресурсов (учебный контур)

**Куда.** На машину разработчика (Windows / Linux / macOS), где есть JDK и Maven. Слушает только `127.0.0.1:8080`. В Kubernetes для учёбы не обязательно: это обычное Boot-приложение, не Helm вендора и не Ingress-контроллер.

**Сколько железа.** Цифр CPU/RAM/реплик «хватит на стенд» у проекта **нет**. Своего склада данных нет. Нужны: JDK, Maven, свободный порт **8080**, место под кэш Maven. Не путать «процесс поднялся» со сметой боя.

**Сильная сторона:** совпадает с моделью продукта (Boot + YAML 5.x), за часы ловит кривой префикс маршрутов.  
**Слабая сторона:** падение процесса = нет входа; стенд **не** доказывает отказ зала, HPA и общий rate limit.

**Критично:** порт **8080** и Actuator в интернет не публиковать. Не ставить оба стартера WebFlux и WebMVC в один процесс. Не `latest` Boot и не диапазон BOM.

---

## Установка для новичка

Официальный каркас: родитель Boot и BOM поезда — https://spring.io/projects/spring-cloud · системные требования Boot **4.0.8** — https://docs.spring.io/spring-boot/4.0.8/system-requirements.html · YAML 5.x — https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html

На странице поездов пример XML всё ещё может показывать BOM **2025.1.2**. Пините **2025.1.3**, как в анонсе от 20.08.2026.

### Что должно быть до установки

Есть:

- **JDK** (комплект Java: компилятор и среда запуска программ). Boot 4.0.8: минимум **Java 17**, совместимость до **Java 26**. Для этого контура — **21**.
- **Maven ≥ 3.6.3** — программа, которая скачивает библиотеки и собирает проект. Gradle 8.14+ / 9.x допустим; шаги ниже — Maven.
- Свободен порт **8080** на localhost. Доступ к Maven Central или вашему зеркалу.
- Закрытая сеть.

Нет (и не должно появиться на стенде):

- Публикация 8080 наружу.
- Стартер `spring-cloud-starter-gateway` без `-server-webflux` (гайды 4.x).
- Включённый discovery locator («все Service наружу»).
- Фильтр `RequestHeaderToRequestUri`.
- `management.endpoint.gateway.access=unrestricted` с любой сети кроме localhost.

### Этап 1. Проверяем Java и Maven

**Что делаем:** убеждаемся, что JDK и Maven есть и не младше минимума Boot 4.0.8.

```bash
java -version
mvn -version
```

Успех: Java **21** (допустимо 17–26); Maven **3.6.3** или новее. Команда `java -version` — из инструкции установки Boot: https://docs.spring.io/spring-boot/4.0.8/installing.html

### Этап 2. Каркас проекта

**Что делаем:** создаём каталог приложения и три файла: `pom.xml` (список библиотек), класс запуска, YAML маршрутов.

Каталоги: `http-gateway/src/main/java/example/` и `http-gateway/src/main/resources/`.

`pom.xml` — родитель **4.0.8**, BOM **2025.1.3**, стартер без своей `<version>`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.8</version>
    <relativePath/>
  </parent>
  <groupId>example</groupId>
  <artifactId>http-gateway</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>21</java.version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>2025.1.3</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

Actuator — отдельный стартер Boot: без него нет `/actuator/gateway`.

`src/main/java/example/GatewayApplication.java`:

```java
package example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {
  public static void main(String[] args) {
    SpringApplication.run(GatewayApplication.class, args);
  }
}
```

`src/main/resources/application.yml` — префикс **5.x WebFlux**. Старый `spring.cloud.gateway.routes` **молча не биндится** (маршрутов нет). Путь в `uri:` официально **игнорируется**.

```yaml
server:
  address: 127.0.0.1
  port: 8080
spring:
  cloud:
    gateway:
      server:
        webflux:
          discovery:
            locator:
              enabled: false
          routes:
            - id: demo
              uri: http://127.0.0.1:8081
              predicates:
                - Path=/api/**
              filters:
                - StripPrefix=1
management:
  endpoints:
    web:
      exposure:
        include: health,gateway
  endpoint:
    gateway:
      access: read-only
```

`server.address=127.0.0.1` — иначе Boot часто слушает все интерфейсы. `StripPrefix=1` срезает первый сегмент пути (`/api/foo` → origin `/foo`): https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/stripprefix-factory.html

Успех: три файла на месте, в `pom.xml` ровно **4.0.8** и **2025.1.3**.

### Этап 3. Проверяем, что подтянулась Gateway 5.0.3

**Что делаем:** Maven скачивает зависимости и печатает дерево. Запускать из каталога с `pom.xml`.

```bash
mvn -q dependency:tree "-Dincludes=org.springframework.cloud"
```

Успех: в дереве **5.0.3**, не 5.0.2. Если 5.0.2 — BOM не 2025.1.3.

### Этап 4. Запуск

**Что делаем:** Maven компилирует проект и стартует встроенный **Netty** (HTTP-сервер Boot для WebFlux). Команда плагина Boot 4.0.8:

```bash
mvn spring-boot:run
```

Успех: в логе порт **8080**, нет ошибки биндинга маршрутов. Остановка — `Ctrl+C`. Документация run: https://docs.spring.io/spring-boot/4.0.8/maven-plugin/run.html

Стенд **ещё не доказывает** отказ процесса, нагрузку и маршруты в Kubernetes.

---

## Первый запуск — URL, порт, учётка, смена пароля

Встроенной админки и **учётки по умолчанию нет**. Это библиотека, не прибор с логином `admin`. Пароля менять нечего. Если позже добавите Spring Security — это уже ваши пользователи, не «завод Gateway».

| Что | Значение на этом стенде |
|---|---|
| HTTP входа | `http://127.0.0.1:8080` |
| Порт приложения | **8080** (`server.port`, завод Boot) |
| Actuator | тот же порт: `management.server.port` не задан |
| Префикс Actuator | `/actuator` |
| Жив ли процесс | `GET http://127.0.0.1:8080/actuator/health` |
| Список маршрутов | `GET http://127.0.0.1:8080/actuator/gateway/routes` |

Проверка маршрутов (только с этой машины):

```bash
curl -s http://127.0.0.1:8080/actuator/gateway/routes
```

Успех: в JSON есть `route_id` / `demo`. Пустой список при живом YAML — почти всегда префикс 4.x.

Запись маршрутов через Actuator на стенде **не используем**. Завод `/gateway` выключен; мы открыли **read-only**. `unrestricted` позволяет создавать и удалять маршруты (вектор SSRF). YAML-маршрут через `DELETE` не удаляется (**404**) — Actuator не источник истины: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html

Прокси: нужен любой HTTP на `127.0.0.1:8081`. Тогда `GET http://127.0.0.1:8080/api/...` уходит на origin без префикса `/api`. Нет origin — проверка Actuator достаточна для «YAML 5.x подхватился»; 502/connection refused на `/api` в этом случае ожидаемы.

`/env` и `/heapdump` не экспонировать.

---

## Подключение к своей системе

**Протокол.** HTTP на стенде. В платформе вход людей — HTTPS на **крае** (HAProxy / SafeLine WAF / Ingress), дальше HTTP до подов шлюза. Шлюз сам открывает исходящий HTTP(S) к origin (Netty HttpClient). Это не прокси для Kafka, Postgres или gRPC Camunda.

**Кто вызывает.** Браузер портала, мобильное, партнёры → край площадки → **этот** Gateway → Service микросервиса **этой же** площадки или Integration API. Исходящие вызовы к госслужбам шлюз **не** делает.

**Секреты и Git.**

| Куда | Что |
|---|---|
| Git (ConfigMap / Helm values) | Маршруты YAML: `id`, `uri` (DNS Service, **без** path), предикаты, `StripPrefix`, таймауты |
| Secret / Vault, не Git | Сертификаты TLS (если снимаете не только на крае), пароль Redis **если** общий rate limit |
| Не класть в Git | Учебный `unrestricted` Actuator, `use-insecure-trust-manager: true` |

Перед краем задайте `spring.cloud.gateway.server.webflux.trusted-proxies` — Java-regex IP края, не `.*`. Без свойства фильтры Forwarded / X-Forwarded в 5.x **не включаются**: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/httpheadersfilters.html

**Чем этот продукт не является.** Не WAF (семантика атаки — SafeLine). Не Ingress-контроллер и не городской VIP (HAProxy / Ingress). Не шина событий (Kafka). Не BPMN (Camunda). Не интеграционное API в ведомства. Не общий сессионный кластер на три ЦОДа: реплики — независимые JVM, согласованность маршрутов — Git.

**Сильная сторона схемы «край → Gateway → локальный Service»:** origin не тащит RTT чужого зала.  
**Слабая:** ошибка GitOps = 404 или лишний путь сразу везде, куда выкатили тот же артефакт.

---

## Ссылки на материал

| Факт | URL |
|---|---|
| Поезд **2025.1.3**, Boot **4.0.8**, Gateway **5.0.3**, CVE-2026-47879 | https://spring.io/blog/2026/08/20/spring-cloud-2025-1-3-has-been-released |
| Текст CVE-2026-47879 (затронуты 5.0.0–5.0.2; фикс 5.0.3) | https://spring.io/security/cve-2026-47879 |
| Матрица поездов (Oakwood = Boot 4.0.x и 4.1.x с 2025.1.2); образец BOM | https://spring.io/projects/spring-cloud |
| Тег v5.0.3 | https://github.com/spring-cloud/spring-cloud-gateway/releases/tag/v5.0.3 |
| Обзор: Server vs Proxy Exchange, WebFlux vs MVC | https://docs.spring.io/spring-cloud-gateway/reference/ |
| YAML `server.webflux.routes` | https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/configuration.html |
| Appendix 5.0.3 (пул `elastic`, connect-timeout 30 с, `metrics.enabled=false`, trusted-proxies) | https://docs.spring.io/spring-cloud-gateway/reference/5.0.3/appendix.html |
| Actuator `/gateway` (read-only vs unrestricted; YAML не DELETE) | https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/actuator-api.html |
| Path в URI маршрута игнорируется | https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/how-it-works.html |
| StripPrefix | https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/stripprefix-factory.html |
| `trusted-proxies` для Forwarded / X-Forwarded | https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/httpheadersfilters.html |
| Boot 4.0.8: Java 17–26, Maven ≥ 3.6.3, Gradle 8.14+ / 9.x | https://docs.spring.io/spring-boot/4.0.8/system-requirements.html |
| `java -version`; Maven с Boot | https://docs.spring.io/spring-boot/4.0.8/installing.html |
| Parent `spring-boot-starter-parent` **4.0.8** | https://docs.spring.io/spring-boot/4.0.8/maven-plugin/using.html |
| `mvn spring-boot:run` | https://docs.spring.io/spring-boot/4.0.8/maven-plugin/run.html |
| Завод `server.port=8080`; встроенный reactive-сервер слушает 8080 | https://docs.spring.io/spring-boot/4.0.8/appendix/application-properties/index.html · https://docs.spring.io/spring-boot/4.0.8/reference/web/reactive.html |
| Actuator: префикс `/actuator`, по HTTP завод только `health` | https://docs.spring.io/spring-boot/4.0.8/reference/actuator/endpoints.html |
| Зачем продукт, порты, роль в платформе | `Spring Cloud Gateway.md` |
| Словарь | `Spring Cloud Gateway.info.md` |
| Схемы стыковки | `Spring Cloud Gateway.shema.md` |
| Kubernetes площадки | `Kubernetes.install.md` |
| Край HTTP | `HAProxy.install.md`, `SafeLine WAF.install.md` |

В документации вендора **нет**: ядер и гигабайт «хватит», порога RTT до origin в другом ЦОДе, готового образа Gateway, встроенной учётки администратора. Stretch сессий шлюза эта инструкция не предлагает.
