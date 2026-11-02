# SMB Network Lab

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer%209-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![VLAN](https://img.shields.io/badge/VLAN-10%20%7C%2020%20%7C%202-0A66C2?style=for-the-badge)
![HSRP](https://img.shields.io/badge/HSRP-Gateway%20Redundancy-ED1C24?style=for-the-badge)
![EIGRP](https://img.shields.io/badge/EIGRP-AS%2010-F7B500?style=for-the-badge)
![ACL](https://img.shields.io/badge/ACL-PROTECT__SERVER-6E40C9?style=for-the-badge)
![Port Security](https://img.shields.io/badge/Port%20Security-Sticky%20MAC-2EA44F?style=for-the-badge)

Отказоустойчивая корпоративная сеть (SMB) на оборудовании **Cisco**, разделённая на **Синюю** и **Жёлтую** зоны.  
В лаборатории реализованы: **VLAN**, **802.1Q Trunking**, **Router-on-a-Stick**, **HSRP**, **EIGRP**, **Extended ACL** и **Port Security**.

> Файл топологии: [`first_project.pkt`](./first_project.pkt) · Cisco Packet Tracer 9.0

---

## Оглавление

- [О проекте](#о-проекте)
- [Структура репозитория](#структура-репозитория)
- [Технологический стек](#технологический-стек)
- [Топология (L2/L3)](#топология-l2l3)
- [Отказоустойчивость шлюза (HSRP)](#отказоустойчивость-шлюза-hsrp)
- [Динамическая маршрутизация (EIGRP)](#динамическая-маршрутизация-eigrp)
- [Безопасность (ACL & Port Security)](#безопасность-acl--port-security)
- [Адресация](#адресация)
- [Проверка работоспособности](#проверка-работоспособности)
- [Как открыть проект](#как-открыть-проект)

---

## О проекте

Проект моделирует типичную SMB-инфраструктуру:

| Зона | Цвет на схеме | Назначение |
|------|---------------|------------|
| **Синяя** | `#0000FF` | Центральный сегмент: пользователи VLAN 10, серверы VLAN 20, Router0 |
| **Жёлтая** | `#FFD20C` | Филиал: пользователи `192.168.30.0/24`, пара маршрутизаторов с HSRP |

Зоны связаны двумя независимыми P2P-линками `/30`. При отказе одного маршрутизатора Жёлтой зоны сохраняются и шлюз по умолчанию (HSRP), и маршрутизация (EIGRP).

### Состав оборудования

| Устройство | Модель | Зона | Роль |
|------------|--------|------|------|
| Router0 | Cisco 2911 | Синяя | Inter-VLAN routing, ACL, EIGRP, Loopback «Internet» |
| SW1, SW2 | Catalyst 2960-24TT | Синяя | Access / Trunk |
| PC0 | PC-PT | Синяя | Клиент Users (DHCP) |
| Server0 | Server-PT | Синяя | DHCP + HTTP (`192.168.20.10`) |
| Router1 | Cisco 2911 | Жёлтая | HSRP **Active** (priority 110) |
| Router2 | Cisco 2911 | Жёлтая | HSRP **Standby** (priority 100) |
| Switch2 | Catalyst 2960-24TT | Жёлтая | Access + Port Security |
| PC1, PC2 | PC-PT | Жёлтая | Клиенты филиала |

---

## Структура репозитория

```text
cisco_git/
├── README.md            # Документация лабораторной работы
└── first_project.pkt    # Готовая топология Cisco Packet Tracer
```

| Файл | Описание |
|------|----------|
| `README.md` | Описание архитектуры, настроек и сценариев проверки |
| `first_project.pkt` | Полная лабораторная среда (устройства + running-config) |

---

## Технологический стек

| Технология | Назначение в лаборатории |
|------------|--------------------------|
| **VLAN** | L2-сегментация: Users / Servers / Native |
| **802.1Q Trunk** | Передача нескольких VLAN между SW и Router |
| **Router-on-a-Stick** | Маршрутизация между VLAN через субинтерфейсы |
| **P2P `/30`** | Связь Router0↔Router1 и Router0↔Router2 без overlap |
| **HSRP** | Резервирование шлюза `192.168.30.1` в Жёлтой зоне |
| **EIGRP AS 10** | Динамическая маршрутизация между тремя роутерами |
| **Extended ACL** | Фильтрация доступа филиала к серверу (только HTTP) |
| **Port Security** | L2-защита портов пользователей (sticky, max 1, restrict) |
| **DHCP + helper** | Централизованная выдача адресов в VLAN 10 |

---

## Топология (L2/L3)

### VLAN

| VLAN ID | Имя | Назначение |
|--------:|-----|------------|
| **10** | Users | Пользовательский сегмент |
| **20** | Servers | Серверный сегмент |
| **2** | Native | Native VLAN на trunk (без тегирования базового трафика) |

### Trunking (802.1Q)

Между коммутаторами и маршрутизаторами настроены **транковые** линки стандарта **802.1Q** с **явным указанием Native VLAN 2**, чтобы:

- передавать несколько VLAN по одному физическому каналу;
- исключить нежелательное тегирование native-трафика;
- снизить риск VLAN hopping из-за несогласованного native VLAN.

### Router-on-a-Stick

Маршрутизация между VLAN реализована через субинтерфейсы маршрутизаторов, например:

- `GigabitEthernet0/0.2` / `GigabitEthernet0/1.2` — VLAN 2 (Native)
- `GigabitEthernet0/0.10` / `GigabitEthernet0/1.10` — VLAN 10 (Users)
- `GigabitEthernet0/0.20` — VLAN 20 (Servers) на Router0

Каждый субинтерфейс использует `encapsulation dot1Q <vlan>`.

### P2P-линки `/30`

Для связи маршрутизаторов применены подсети с маской **`255.255.255.252` (`/30`)**:

| Линк | Подсеть | Интерфейсы |
|------|---------|------------|
| Router0 ↔ Router1 | `10.10.1.0/30` | `Gi0/1` ↔ `Gi0/0` |
| Router0 ↔ Router2 | `10.10.1.4/30` | `Gi0/2` ↔ `Gi0/0` |

Такой выбор экономит адресное пространство и исключает пересечения (overlaps) адресных блоков.

### Логическая схема

```text
                 ┌──────────────── СИНЯЯ ЗОНА ────────────────┐
                 │  VLAN10 Users · VLAN20 Servers · VLAN2 Nat │
                 │                                            │
   PC0 ── SW1 ── SW2 ── Server0 (192.168.20.10)               │
                 │                                            │
                 └────────────── Router0 ─────────────────────┘
                         /              \
                 10.10.1.0/30        10.10.1.4/30
                       /                    \
               Router1 (Active)        Router2 (Standby)
               pri 110 · .30.2         pri 100 · .30.3
                       \                    /
                        └── VIP 192.168.30.1 ──┘
                                   │
                                Switch2
                          (Port Security)
                               /        \
                             PC1        PC2
                 └────────────── ЖЁЛТАЯ ЗОНА ──────────────┘
```

---

## Отказоустойчивость шлюза (HSRP)

В **Жёлтой зоне** (пользовательская сеть `192.168.30.0/24`) реализовано резервирование шлюза по умолчанию протоколом **HSRP**.

| Параметр | Значение |
|----------|----------|
| Группа HSRP | `1` |
| Virtual IP (VIP) | `192.168.30.1` |
| Router1 | Active · IP `192.168.30.2` · **priority 110** · `preempt` |
| Router2 | Standby · IP `192.168.30.3` · **priority 100** · `preempt` |
| Шлюз хостов | `192.168.30.1` (VIP) |

Поведение при сбоях:

1. Падение **Router1** → роль Active переходит на **Router2**, VIP остаётся `192.168.30.1`.
2. Восстановление Router1 → благодаря **`preempt`** он возвращает Active-роль автоматически.
3. Хосты филиала **не меняют** default gateway.

Пример фрагмента конфигурации:

```cisco
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.30.2 255.255.255.0
 standby 1 ip 192.168.30.1
 standby 1 priority 110
 standby 1 preempt
```

---

## Динамическая маршрутизация (EIGRP)

На всех трёх маршрутизаторах запущен **EIGRP в AS 10**.

Выполненные настройки:

- анонсированы **P2P-линки** и **локальные сети**;
- отключено автосуммирование (`no auto-summary` — для корректного поведения в несмежных подсетях / контроля агрегации);
- интерфейсы в сторону локальных сетей (**VLAN 10, VLAN 20, Native**) переведены в **`passive-interface`**, чтобы EIGRP hello не уходили конечным узлам;
- на **Router0** создан **Loopback0 `8.8.8.8/32`** для имитации сети Интернет и анонсирован в EIGRP.

```cisco
router eigrp 10
 passive-interface GigabitEthernet0/0.2
 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.20
 network 10.10.1.0 0.0.0.3
 network 10.10.1.4 0.0.0.3
 network 192.168.10.0
 network 192.168.20.0
 network 192.168.2.0
 network 8.8.8.8 0.0.0.0
```

Итог: при отказе одного из путей EIGRP пересчитывает маршруты через оставшийся `/30`-линк без ручного вмешательства.

---

## Безопасность (ACL & Port Security)

### Access Control List (Extended ACL)

На **Router0** создан расширенный список **`PROTECT_SERVER`**, применённый **outbound** на субинтерфейс сервера (`Gi0/0.20 out`).

Политика:

1. **Разрешить** трафик из Жёлтой зоны `192.168.30.0/24` к серверу `192.168.20.10` **только по HTTP** (TCP/80).
2. **Явно запретить** любой другой IP-трафик к этому серверу из той же подсети.
3. **Разрешить** остальной легитимный трафик (`permit ip any any`).

```cisco
ip access-list extended PROTECT_SERVER
 permit tcp 192.168.30.0 0.0.0.255 host 192.168.20.10 eq www
 deny   ip  192.168.30.0 0.0.0.255 host 192.168.20.10
 permit ip any any
!
interface GigabitEthernet0/0.20
 ip access-group PROTECT_SERVER out
```

### Port Security (L2)

На коммутаторе **Switch2** (Жёлтая зона) порты пользователей:

- режим **Access**;
- **sticky MAC** (`switchport port-security mac-address sticky`);
- лимит **1 устройство на порт** (`maximum 1`);
- violation **`restrict`** — чужой трафик блокируется, события фиксируются в логах.

```cisco
interface FastEthernet0/3
 switchport access vlan 10
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
```

---

## Адресация

### Синяя зона

| Сегмент | Подсеть | Шлюз | Примечание |
|---------|---------|------|------------|
| Native VLAN 2 | `192.168.2.0/24` | `192.168.2.1` | Native на trunk |
| Users VLAN 10 | `192.168.10.0/24` | `192.168.10.1` | DHCP через helper → Server0 |
| Servers VLAN 20 | `192.168.20.0/24` | `192.168.20.1` | Server0 = `192.168.20.10` |
| Loopback0 | `8.8.8.8/32` | — | Имитация Internet |

DHCP-пул для PC0: `192.168.10.100–192.168.10.199`, default-router `192.168.10.1`.

### Жёлтая зона

| Узел | Адрес |
|------|-------|
| HSRP VIP | `192.168.30.1` |
| Router1 | `192.168.30.2` |
| Router2 | `192.168.30.3` |
| PC1 | `192.168.30.10` / GW `192.168.30.1` |

### Межмаршрутизаторные линки

| Подсеть | Хосты |
|---------|-------|
| `10.10.1.0/30` | Router0 `10.10.1.1` · Router1 `10.10.1.2` |
| `10.10.1.4/30` | Router0 `10.10.1.5` · Router2 `10.10.1.6` |

---

## Проверка работоспособности

| № | Сценарий | Ожидаемый результат |
|---|----------|---------------------|
| 1 | PC0 получает адрес по DHCP | Адрес из `192.168.10.100–199`, GW `192.168.10.1` |
| 2 | PC1 → `http://192.168.20.10` | HTTP доступен (ACL permit www) |
| 3 | PC1 → ping `192.168.20.10` | Блокируется ACL (`deny ip ... host`) |
| 4 | Отключить Router1 | HSRP: Active → Router2, VIP без изменений |
| 5 | Восстановить Router1 | `preempt`: Active возвращается на Router1 |
| 6 | Подключить чужой хост к порту Switch2 | Port Security `restrict` + лог нарушения |
| 7 | `show ip eigrp neighbors` на Router0 | Соседи Router1 и Router2 |

Полезные команды:

```cisco
show standby brief
show ip eigrp neighbors
show ip route eigrp
show access-lists PROTECT_SERVER
show port-security interface FastEthernet0/3
```

---

## Как открыть проект

1. Установите [Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer) (рекомендуется **9.x**).
2. Клонируйте репозиторий и откройте `first_project.pkt`.
3. Пройдите сценарии из раздела [Проверка работоспособности](#проверка-работоспособности).

```bash
git clone <URL-репозитория>
cd cisco_git
# открыть first_project.pkt в Packet Tracer
```

---

## Цели лабораторной работы

- [x] Сегментировать сеть VLAN (Users / Servers / Native)
- [x] Настроить 802.1Q trunk с явным Native VLAN
- [x] Реализовать Router-on-a-Stick
- [x] Связать маршрутизаторы P2P-линками `/30`
- [x] Обеспечить отказоустойчивый шлюз через HSRP + preempt
- [x] Поднять EIGRP AS 10 с passive-interface и Loopback «Internet»
- [x] Ограничить доступ к серверу Extended ACL
- [x] Защитить access-порты Port Security (sticky / max 1 / restrict)

---

## Лицензия

Учебный проект для Cisco Packet Tracer. Конфигурации демонстрируют технологии CCNA-уровня и не являются готовым production-шаблоном (без AAA, шифрования control-plane и hardening).
