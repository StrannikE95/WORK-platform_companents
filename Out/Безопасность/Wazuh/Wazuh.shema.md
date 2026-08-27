# Wazuh 4.14.7 — схемы устройства

Связанные документы: правила — `Wazuh.md`; установка — `Wazuh.install.md`.

C4 / «D4»: контекст → контейнеры → компоненты. Код агента не рисуем.

Допущения:

1. Stretch indexer/manager на 2–3 ЦОДа **нет**. HA центра — **внутри ЦОД-1**.
2. Self-hosted 4.14.7 (`wazuh-kubernetes` v4.14.7). Indexer — **свой** OpenSearch Wazuh, **не** платформенный OpenSearch 3.8.
3. Нагрузки (APS) нет — на схемах нет «N ядер». Есть *что крутить*.

---

## 1. Контекст (C4 system context)

Wazuh — **SIEM/XDR**: логи, FIM, SCA, поиск алертов. Не шина, не WAF, не Falco.

```mermaid
flowchart LR
  subgraph endpoints["Наблюдаемые хосты"]
    ND["Ноды K8s / VM Linux"]
    WIN["Windows / бастионы"]
  end

  WZ["Wazuh 4.14.7\nагент → server → indexer"]
  FC["Falco\nsyscall на ноде"]
  WF["WAF\nHTTP на входе"]
  KF["Kafka / Camunda\nисточник логов"]

  ND -->|"агент 1514"| WZ
  WIN --> WZ
  KF -.->|"логи / FIM, не SoT"| WZ
  FC -.->|"другой слой"| WZ
  WF -.->|"не замена"| WZ
```

| Стрелка | Зачем помнить |
|---|---|
| Агент → server | События AES по 1514; enrollment 1515 на **master** |
| Falco / WAF рядом | Syscall и HTTP-эксплойт — не роль Wazuh |
| Kafka как хост | Без агента (или syslog) SIEM слеп там, где деньги |

**Слабое место контекста:** три изолированных Wazuh = **три SIEM**, три индекса, три правды для SOC.

---

## 2. Контейнеры (четыре роли, два кластера)

Один логический Wazuh = агенты + **один** master + worker'ы + indexer + dashboard. All-in-one — стенд, не прод.

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1: мозг SIEM"]
    LB["TCP LB 1514 / 1515"]
    subgraph srv["Server-кластер"]
      MST["Master\nauthd 1515, API 55000"]
      W1["Worker анализ"]
      W2["Worker"]
    end
    subgraph idx["Indexer Wazuh\nНЕ OpenSearch 3.8"]
      I1["нода 9200 / 9300"]
      I2["нода"]
      I3["нода"]
    end
    DASH["Dashboard\n443 / контейнер 5601"]
  end

  AG["Агенты ЦОД-1/2/3"]

  AG -->|"1514 события"| LB
  AG -->|"1515 enrollment"| MST
  LB --> W1
  LB --> W2
  MST <-->|"1516 cluster key"| W1
  MST <--> W2
  W1 -->|"Filebeat TLS 9200"| I1
  DASH --> I1
  DASH -->|"API"| MST
  I1 <-->|"9300 cluster state"| I2
  I2 <--> I3
```

**Сильное:** приём событий переживает падение worker (есть LB). **Слабое:** второго master **нет**; падение master = enrollment и управление встают. Это не Raft.

---

## 3. Компоненты, которые путают

```mermaid
flowchart LR
  subgraph wazuh["Это Wazuh"]
    AG2["Agent: логи, FIM, SCA"]
    AN["analysisd / remoted"]
    FB["Filebeat на server"]
    IX["Indexer шарды + ISM"]
  end

  subgraph not["Это не Wazuh"]
    OS["Платформенный OpenSearch 3.8"]
    NP["NetworkPolicy / WAF"]
    FL["Falco eBPF"]
  end
```

Порт **1516** — только узлы server. **9300–9400** — только ноды *этого* indexer. Смешать с чужим OpenSearch нельзя: другая линейка, другие индексы.

Дефолт шаблона алертов: 3 primary / **0 replica** — не HA. Overlay-секреты в Git (`password`, `SecretPassword`) в прод не копировать.

---

## 4. Поток: агент шлёт событие

```mermaid
sequenceDiagram
  participant Ag as Агент на ноде
  participant Lb as TCP LB
  participant Wk as Worker
  participant Ms as Master
  participant Ix as Indexer
  participant Ui as Dashboard

  Ag->>Ms: enrollment 1515 (ключ агента)
  Ms-->>Ag: unique key
  Ag->>Lb: событие TCP 1514 AES
  Lb->>Wk: leastconn
  Wk->>Wk: decoder + rules
  Note over Ms,Wk: 1516: ключи/rules с master; ossec.conf worker не едет сам
  Wk->>Ix: Filebeat TLS 9200
  Ui->>Ix: поиск алерта
  Ui->>Ms: управление агентами
```

Без Filebeat алерт лежит в файлах manager, UI его не видит. `events_dropped` / `discarded_count` должны быть **0**.

---

## 5. Что настраивать: отказоустойчивость

```mermaid
flowchart LR
  subgraph inside["Внутри ЦОД-1"]
    IX3["indexer 3 ноды\nreplica шардов 1+"]
    WK2["1 master + 2+ worker"]
    LBH["LB перед 1514"]
    ISM["ISM retention"]
  end

  subgraph edge["Другие ЦОДы"]
    AGX["Только агенты\nпуть до VIP 1514"]
  end

  inside -->|"падение worker"| OK["приём жив через LB"]
  inside -->|"падение ЦОД-1"| DR["нет SIEM, пока restore"]
  edge --> inside
```

| Ручка | Если не настроить |
|---|---|
| Indexer ≥ 3 + replica ≥ 1 | Падение узла = дырка в индексе (red/неполный) |
| LB на 1514 | Агент знает один IP пода — слепой при выкате |
| Один master, не три | «HA master» вендор **запрещает** |
| PVC manager+indexer | Deployment без диска = потеря ключей агентов |
| `vm.max_map_count` ≥ 262144 | Indexer не стартует |

Пережить **два** ЦОДа на трёх cluster-manager нельзя честно. Цель отказа — 1 площадка у мозга, не 2 из 3.

---

## 6. Что настраивать: масштабируемость

```mermaid
flowchart TB
  Q["Где упёрлись?"]
  Q --> E["События / APS"]
  Q --> S["Поиск / диск indexer"]
  Q --> A["Число агентов"]

  E --> E1["Ещё worker\nне второй master"]
  S --> S1["Ноды indexer + heap\nISM, не archives вслепую"]
  A --> A1["DaemonSet / пакет\nмасштаб = хосты"]
```

Три оси независимы: worker ≠ indexer. Ресурсы EKS-патча (2 Gi) — стенд, не ёмкость. FIM на data-dir Kafka взорвёт APS.

---

## 7. Прод: 2 или 3 ЦОДа без stretch

```mermaid
flowchart TB
  subgraph dc1["ЦОД-1"]
    BR["Indexer HA + master/worker"]
  end
  subgraph dc2["ЦОД-2"]
    A2["Агенты → VIP ЦОД-1"]
  end
  subgraph dc3["ЦОД-3"]
    A3["Агенты → VIP ЦОД-1"]
  end

  A2 -->|"1514 / 1515"| BR
  A3 --> BR
```

Один SoT алертов. Три полных стека «по кластеру как Falco» = три SIEM: SOC склеивает вручную, ruleset разъедется.

**Сильное:** кворум indexer и 1516 не едут по межЦОДовому RTT (порога мс у Wazuh **нет** — поэтому не растягиваем). **Слабое:** смерть ЦОД-1 = нет анализа/поиска для всех площадок; агенты копят очередь, бесконечный буфер не обещан.

---

## 8. Безопасность (слои)

1. Сменить все overlay-секреты; cluster key 32 символа = root кластера.
2. 9200 / 9300 / 1516 / 55000 не с мира; 1514 только от агентов/LB.
3. Свои cert, не self-signed скрипта. Dashboard не LoadBalancer в интернет.
4. Агент в K8s без hostPath/`/var/log` — декорация, хост не видит.
5. Active Response по умолчанию **выкл** на интеграционном контуре.

Источники: `Wazuh.md` (роли, порты, один master, indexer ≠ OpenSearch 3.8). Stretch на схемах не целевой.
