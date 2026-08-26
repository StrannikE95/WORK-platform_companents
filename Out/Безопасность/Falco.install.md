# Falco 0.44.1 — установка и конфигурирование

Связанный документ (глоссарий, modern eBPF, дропы, Operator vs Helm, почему так): `Falco.md`.

Этот файл — **как поставить и настроить**. Не копируйте Dev-значения в прод. Stretch здесь **не применим**: Falco — **агент на ноде** (DaemonSet), не Raft и не общий кластер с кворумом. Два или три независимых Kubernetes = **Falco в каждом**. Глобального «кластера Falco на страну» нет и не нужно. Алерты уходят в SIEM (Wazuh) и/или в NATS через Falcosidekick — не в выдуманный federated Falco.

Версии: **Falco 0.44.1**, образ **`falcosecurity/falco:0.44.1`**. Рекомендуемый путь на Kubernetes **1.29+**: **Falco Operator v0.4.1**, Helm-чарт **`falcosecurity/falco-operator` 0.3.0** (с Operator v0.4.0 default Falco = 0.44.1). Альтернатива: Helm **`falcosecurity/falco` 9.1.0** (тоже 0.44.1) — если кластер старше 1.29 (native sidecars Artifact Operator).  
Документация: https://falco.org/docs/ · Operator: https://falco.org/docs/setup/operator/

---

## Допущения этой инструкции

Их не было в исходном контексте платформы; без них нельзя дать конкретные команды.

1. **По контуру Falco на Kubernetes-кластер.** 2 ЦОДа = 2 кластера = 2 установки. 3 ЦОДа = 3. Общего etcd/Falco-кворума нет.
2. Прод — **vanilla Kubernetes 1.36.x + kubeadm** в каждом ЦОДе (см. `Kubernetes.install.md`) → Operator v0.4.1 допустим (1.36 ≥ 1.29).
3. Драйвер прода — **`modern_ebpf`**. Ядра с BTF и BPF ring buffer (обычно Linux ≥ 5.8). kmod — только пул, где modern eBPF физически не встаёт (full privileges).
4. Dev — изолированная сеть; HTTP до Sidekick на тесте допустим.
5. Нагрузки нет — нет millicores «хватит агенту на брокер». Есть `scap.n_drops` и рычаги буферов/правил.
6. Приёмник алертов **будет**: SIEM и/или NATS (через Falcosidekick). Сам Falco 0.44 шлёт stdout/file/syslog/HTTP; встроенного NATS/Kafka producer **нет** (gRPC output в 0.44 удалён). Falcosidekick UI **не** замена SIEM.
7. Talon / kill pod в этом файле **не включается**.

---

## Dev: 1 ЦОД, без нагрузки

**Цель:** увидеть алерт штатных правил на ваших образах, отладить доставку. **Не** цель: доказать детект на Kafka-ноде под нагрузкой.

### Предпосылки

- Dev-Kubernetes, Linux-ноды (Windows Falco не покрывает).
- BTF/ringbuf на ядре стенда — иначе modern eBPF не встанет, и вы отладите не тот драйвер.
- Registry до `falcosecurity/falco:0.44.1` и OCI-правил (или зеркало).

Deployment с `replicas: 2` для syscall-мониторинга **не использовать**: ложное чувство HA и не покрывает ноды.

### Установка (Operator — основной путь Dev при K8s ≥ 1.29)

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
kubectl create namespace falco
helm install falco-operator falcosecurity/falco-operator \
  --version 0.3.0 -n falco
```

Затем CR `Falco`: DaemonSet, **`engine.kind: modern_ebpf`**, Plugin **container**, Rulesfile официальный. Образ агента явно **0.44.1**, не `latest`. На тесте тег ruleset можно ослабить; в прод — pin.

Альтернатива без Operator:

```bash
helm install falco falcosecurity/falco --version 9.1.0 -n falco \
  --set image.tag=0.44.1
```

Сверять values чарта 9.1.0: driver modern eBPF, `daemonset` не Deployment.

Quickstart-манифест оператора (`quickstart.yaml`) поднимает Falco + Sidekick + UI + Redis + metacollector. Для знакомства удобно. **Это не прод** (HTTP, `latest`, UI).

Опционально на тесте: Falcosidekick **1** реплика, HTTP только в закрытой сети. Хотя бы один выход кроме `kubectl logs`.

### Конфигурирование Dev

| Параметр | На тесте | Почему можно |
|---|---|---|
| TLS до Sidekick | нет, если сеть закрыта | Иначе PKI раньше правил |
| Реплики Sidekick | 1 | Нет требования пережить выкат |
| Свои exceptions | минимум | Сначала увидеть шум Java/Kafka |
| CPU limit | мягкий / без жёсткого throttle | Не получить дропы из cgroup |
| Control-plane | можно не сразу | На тесте часто одна нода |

Чего **не** упрощать: драйвер **`modern_ebpf`**; метрики дропов включены (`kernel_event_counters_enabled`, `syscall_event_drops`: `log` + `alert`, не `ignore`); container plugin **до** ruleset (иначе поля `container.*` мертвы).

### Проверка Dev

1. Поды DaemonSet Running, драйвер modern eBPF, версия **0.44.1**.
2. `kubectl exec` в тестовый под с `sh` → ожидаемый алерт дошёл до stdout **и** до Sidekick/лога.
3. Рестарт пода Falco: нода слепая, пока не встанет — ожидаемо.

### Сильные / слабые стороны Dev

| Сильное | Слабое |
|---|---|
| Часы, официальный quickstart Operator | Нет TLS, нет покрытия tainted-нод, нет ёмкости буферов |
| Шум правил на *ваших* образах дёшево | Детект exec на простой ноде ≠ брокер под нагрузкой |
| | UI приучает «смотреть в Falco», а не в SIEM |

Препрод: DaemonSet на все tainted-ноды, TLS, 2 Sidekick, pin 0.44.1, свои exceptions — в **одном** ЦОДе.

---

## Prod: 2 или 3 ЦОДа, без stretch, нагрузка возможна

**Цель:** на **каждой** живой Linux-ноде с нагрузкой есть агент; падение ЦОДа слепит **только** его ноды; алерты с живых площадок доходят в SIEM/NATS. Цифр events/s нет.

### Почему не stretch

Нечего растягивать. Syscalls снимаются с **этого** ядра. «Один Falco на три ЦОДа» означало бы один Kubernetes — его у вас нет. Сводить нужно **алерты**, не агенты.

### Топология

В **каждом** Kubernetes — свой контур:

```
ЦОД-1: Operator + DaemonSet Falco 0.44.1 + Falcosidekick (≥2)
ЦОД-2: то же независимо
ЦОД-3: то же, если площадок три
```

| Площадок | Что где | Отказ ЦОД-1 |
|---|---|---|
| **2 ЦОДа** | Falco в обоих кластерах; Sidekick каждого смотрит в **общий** SIEM и/или NATS | Детект и нагрузка ЦОД-1 мертвы вместе. ЦОД-2 шлёт алерты как раньше |
| **3 ЦОДа** | Третий такой же контур | То же; третий агент не «выбирает лидера» за мёртвый ЦОД |

Не ставить один Sidekick «в ЦОД-1 на всех»: смерть домашнего ЦОДа + межЦОДовый HTTP с нод — лишняя связность. Sidekick **локальный** кластеру; дальше SIEM/NATS, которые и так умеют принимать с нескольких источников.

Метки `k8s.cluster` / hostname ЦОДа в алерте закладывают **до** боя, иначе SOC увидит три потока без площадки.

### Предпосылки на каждой площадке

- Ядро: BTF, ringbuf, `CONFIG_BPF_JIT` / `net.core.bpf_jit_enable=1`. Снять **во всех пулах** до install.
- CRI-сокеты на ноде для container plugin (containerd).
- Зеркало `falcosecurity/falco:0.44.1` и OCI-артефактов правил.
- PKI: HTTPS Falco → Sidekick с **валидным** сертификатом. Документация Falco: self-signed / invalid для `http_output` **не поддерживаются**.
- Куда слать: Wazuh (SIEM; Falcosidekick в него целят отдельно, это не часть дистрибутива Wazuh) и/или NATS. Канал в git не класть как секрет URL+token в открытую.
- Taints: control-plane, dedicated Kafka, Camunda, интеграция — без tolerations это **слепые** самые важные ноды.

### Установка (повторить в каждом кластере)

1. Helm **falco-operator 0.3.0** → Operator **v0.4.1**. Deployment оператора **≥ 2** + PDB. Краткий простой оператора уже запущенный DaemonSet обычно переживает; выкат правил — нет.
2. CR `Falco`: только **DaemonSet**, `engine.kind: modern_ebpf`, образ **0.44.1** (userspace 0.44 **несовместим** с драйвером 0.43).
3. Порядок артефактов (дока Operator): Plugin **container** (pin тега) → **k8smeta** + Component metacollector → Rulesfile **зафиксированный** тег → свой overlay exceptions.
4. Не удалять `Falco` CR раньше Rulesfile/Plugin/Config — finalizer sidecar повиснет.
5. Метрики + `syscall_event_drops.actions`: `log` + `alert`. Не `ignore`. Не `exit` как норма (гарантированная слепота до рестарта).
6. Выходы **до** объявления «мы под мониторингом»:
   - stdout/syslog → сбор логов ноды (переживает смерть Sidekick);
   - `http_output` HTTPS → локальный Falcosidekick (`json_output: true`).
7. Falcosidekick **≥ 2**, PDB `minAvailable: 1`, anti-affinity в **этом** ЦОДе. `tlsserver.deploy=true`, лучше mTLS. Не LoadBalancer в интернет. Quickstart `http://sidekick:2801` в прод **не** копировать.
8. Маршруты Sidekick: SIEM (Wazuh) и/или NATS. Опционально отдельный топик Kafka — не бизнес-топики, TLS+ACL. UI Sidekick наружу не публиковать.
9. NetworkPolicy: поды Falco только на Sidekick; Sidekick только на SIEM/NATS.
10. GitOps на Rulesfile/Config. Один и тот же overlay на 2–3 кластера **плюс** идентификатор кластера в алерте.

Least privilege modern eBPF: `CAP_BPF`, `CAP_PERFMON`, `CAP_SYS_RESOURCE`, `CAP_SYS_PTRACE` (на ядрах без дробных cap — ещё `CAP_SYS_ADMIN`). Не `privileged: true` «на всякий случай». hostPath — минимум: `/proc`, runtime sockets.

`priorityClassName` высокий: не вытеснять агента как BestEffort. Rolling DaemonSet: `maxUnavailable` низкий (часто 1). Во время выката часть нод слепая.

Metacollector в quickstart — `replicas: 1`: SPOF **обогащения** k8s-полями, не съёма syscalls. Мониторить; детект ядра без него жив, алерты деградируют до hostname.

### Конфигурирование (смысл)

| Параметр | Прод | Зачем |
|---|---|---|
| Режим | DaemonSet на **всех** Linux-нодах с нагрузкой | Deployment не видит чужие ядра |
| Драйвер | modern_ebpf | Без сборки kmod; legacy eBPF в 0.44 удалён |
| Правила | pin OCI + overlay | `latest` разъедет 2–3 кластера |
| HTTP | https, валидный cert | Self-signed Falco не примет |
| Дропы | log+alert, не ignore | Слепая зона должна быть видна |
| CPU limit | не 50m «чтобы не мешал» | Throttle → дропы → ложь |

`output_timeout` подбирать замером: слишком маленький — потеря HTTP-алертов; слишком большой — backpressure → дропы. Пример оператора `2000` мс — иллюстрация, не закон.

### Масштабирование (когда появятся цифры)

Единица масштаба — **нода**. Добавили worker → DaemonSet сам поставил агента. Второй процесс Falco на той же ноде **удваивает** работу ядра, не делит её.

1. Смотреть `scap.n_drops`, `scap.evts_drop_rate_sec`, CPU пода.
2. Дропы: сначала сузить правила/`base_syscalls` (`repair: true` — официальный рычаг минимума для state engine), потом буферы.
3. Troubleshooting modern eBPF (с оговоркой авторов): эксперимент `buf_size_preset` 6–7, `cpus_for_each_buffer` 4–6 на крупных машинах. Больше буфер = больше RAM × число буферов. Не гарантия.
4. Ориентир той же страницы: порядка < ~1–1.5k events/s на CPU обычно терпимо, > ~3k/s часто тяжело — **не** смета вашего кластера.
5. Шумные пулы (Kafka) — отдельные замеры. Документация признаёт: на очень шумных машинах дропы могут быть неустранимы. Тогда не врать SLA.
6. Не второй syscall-агент «для надёжности» на те же tracepoints.

### Проверка прода (пока это не пройдено — это не прод)

На **каждом** кластере:

1. Версия **0.44.1**, modern eBPF, container plugin, DaemonSet на tainted-нодах (Kafka/Camunda тоже).
2. Контролируемый exec / event-generator → алерт в SIEM **и/или** NATS, не только в `kubectl logs`.
3. Метрики дропов живые; `ignore` нет.
4. HTTPS Sidekick; с мира UI/2801 — отказ.
5. Убить один под Sidekick: второй принимает; syslog/stdout всё ещё пишет.

МежЦОДовый прогон: выключить кластер ЦОД-1 — алерты ЦОД-2 (и ЦОД-3) идут. Нет «переключения Falco».

Неделя шума на препроде → exceptions → **потом** on-call. Без этого либо игнор, либо душат легитимный `exec`.

### Сильные / слабые стороны (контур на кластер + SIEM/NATS)

| Сильное | Слабое |
|---|---|
| Нет зависимости детекта от межЦОДового RTT | 2–3 выката правил; легко забыть новый кластер |
| Blast radius = ЦОД: чужие ноды мониторятся как раньше | Без общего SIEM/NATS — три правды |
| Согласовано с независимыми Kubernetes | Sidekick/сеть алертов — отдельный путь отказа |
| modern eBPF без kmod на ядрах ≥ 5.8 | Агент видит всю ноду; компрометация = наблюдаемость |

**Не готов к проду**, если: Deployment вместо DaemonSet для syscalls; поставили только в ЦОД-1; нет container plugin; `latest`; PLAIN HTTP Sidekick на общей сети; нет tolerations на Kafka; `syscall_event_drops: ignore`; единственный выход — UI; ждут, что Falco *остановит* атаку; ядро без BTF при modern_ebpf «надеемся»; выдумали глобальный кластер Falco вместо SIEM/NATS.

---

## Источники

- Релиз 0.44.1: https://github.com/falcosecurity/falco/releases/tag/0.44.1
- Драйверы, modern eBPF, capabilities: https://falco.org/docs/concepts/event-sources/kernel/
- Дропы: https://falco.org/docs/concepts/event-sources/kernel/dropped-events/
- Troubleshooting буферов: https://falco.org/docs/troubleshooting/dropping/
- HTTP/HTTPS, запрет invalid cert: https://falco.org/docs/concepts/outputs/channels/
- Operator, DaemonSet vs Deployment, K8s 1.29+: https://falco.org/docs/setup/operator/
- Матрица Operator → Falco 0.44.1: https://github.com/falcosecurity/falco-operator/blob/main/docs/version-matrix.md
- Operator v0.4.1: https://github.com/falcosecurity/falco-operator/releases/tag/v0.4.1
- Helm `falco` 9.1.0: https://artifacthub.io/packages/helm/falcosecurity/falco
- Правила и пробелы: `Falco.md`

Stretch-кластера Falco в документации проекта нет — в этой инструкции его нет. Алерты сходятся в SIEM/NATS, не в общий агент.
