# SRv6

## Коротко

SRv6 - реализация Segment Routing на IPv6. SID - это 128-битный IPv6 адрес, который одновременно и адрес узла, и инструкция. Underlay - чистый IPv6 с IGP (IS-IS). Overlay - VPN сервисы через BGP.

Нет MPLS. Нет LDP. Один протокол для transport и services.

## Структура SID

```
|<-- Locator (L) -->|<-- Function (F) -->|<-- Args (A) -->|
```

Locator - маршрутизируемый IPv6 префикс (/48 или /64). IGP знает как дойти до locator.

Function - что делать когда дошли. 16-32 бита. End.DT4, End.DX2 и т.д.

Args - дополнительные параметры. Обычно не используется.

Locator состоит из Block (адресное пространство домена) и Node ID.

## Адресный план

Типичный формат (F3216):

```
BBBB:BBBF:SSNN::/48

B - block организации (28 бит)
F - Flex-Algo (4 бита, если нужен)
S - Set/POD (8 бит)
N - Node (8 бит)
```

Главное - планировать суммаризацию. На границе POD анонсируется один /48 вместо сотен /64 от каждого узла.

## SRH

Segment Routing Header - extension header для списка сегментов.

Segments Left (SL) - сколько сегментов осталось. Segment List - массив SID.

Активный сегмент лежит в DA пакета. Узел обрабатывает, уменьшает SL, копирует следующий SID в DA.

Если путь из одного сегмента - SRH вообще не нужен, хватает DA.

## Behaviors

### На входе (Headend)

H.Encaps - добавляет внешний IPv6 header и SRH. Оригинальный пакет становится payload.

H.Encaps.Red - то же самое, но первый SID не дублируется в SRH (только в DA). Экономит 16 байт.

H.Encaps.L2 - для L2 фреймов.

### Транзит

End - базовый. Обновляет SL и DA, forwarding дальше.

End.X - то же + форсирует конкретный nexthop (как Adjacency SID).

### L3VPN

End.DT4 - снимает внешний IPv6, делает lookup в IPv4 VRF. Аналог per-VRF label.

End.DT6 - то же для IPv6 VRF.

End.DT46 - один SID для dual-stack VRF. Смотрит inner protocol.

### L2VPN

End.DX2 - снимает IPv6, выдает L2 frame на конкретный AC. Для VPWS.

End.DT2U - снимает IPv6, MAC lookup в bridge table. Для ELAN unicast.

End.DT2M - для BUM трафика.

## Micro-SID

Проблема полных SID - overhead. 5 хопов = 80 байт SID-листа.

uSID - компрессия. В один 128-битный carrier влезает до 6 микро-SID по 16 бит.

```
|<-- Block 32 -->|<-- uSID1 16 -->|<-- uSID2 16 -->|...|
```

Те же 5 хопов = 16 байт.

uN behavior - сдвигает uSID влево и делает lookup по новому DA.

Если больше 6 uSID - добавляется SRH с дополнительными carriers.

## L3VPN

Underlay: IS-IS анонсирует locators.

Overlay: MP-BGP VPNv4/VPNv6 между PE.

На PE создается VRF, назначается locator, автоматически аллоцируется End.DT4/DT6 SID.

Путь пакета:

```
CE -> Ingress PE: VRF lookup, находит remote PE
Ingress PE: H.Encaps, DA = End.DT4 SID удаленного PE
Transit P: обычный IPv6 forwarding по DA (это же locator prefix)
Egress PE: видит свой End.DT4, decap, lookup в VRF
Egress PE -> CE
```

## EVPN

VPWS (E-Line) - point-to-point. EVPN Type 1 route несет End.DX2 SID.

ELAN (E-LAN) - multipoint. EVPN Type 2 (MAC/IP) несет End.DT2U SID. MAC learning через BGP, не flooding.

IRB - L2 и L3 вместе. End.DT2U для bridging внутри VLAN, End.DT46 для routing между VLAN. Anycast gateway на всех leaf.

## DC Fabric

Spine-Leaf архитектура.

Spine - чистый P router. IPv6 forwarding, IS-IS с SRv6, никаких VRF.

Leaf - PE router. VRF для тенантов, locator, BGP EVPN с другими leaf.

Locators:

```
Leaf-1: 2001:db8:0:1::/64
Leaf-2: 2001:db8:0:2::/64
Leaf-3: 2001:db8:0:3::/64
```

Spine видит summary 2001:db8::/48 и locator каждого leaf.

Каждый VRF на leaf получает свой function:

```
Leaf-1, Tenant-A: 2001:db8:0:1::100 (End.DT46)
Leaf-1, Tenant-B: 2001:db8:0:1::200 (End.DT46)
```

## DCI

Вариант 1 - SRv6 core между DC:

```
DC-1 Fabric <-> SRv6 WAN <-> DC-2 Fabric
```

Border leaf анонсирует EVPN routes в core. End-to-end SRv6, никаких gateway.

Вариант 2 - DC на VXLAN, core на SRv6:

```
VXLAN Fabric <-> BorderPE <-> SRv6 Core
```

BorderPE терминирует VXLAN, импортирует routes в VRF, реанонсирует в L3VPN SRv6.

## Best Practices

### Адресация

- Выделить минимум /32 block под SRv6 (лучше /28 с запасом)
- Зарезервировать один nibble (4 бита) под Flex-Algo ID
- Flex-Algo кодировать в старших битах - так проще суммаризация
- Планировать иерархию: Block -> Region -> POD -> Node
- На границе POD суммаризировать до /48

Пример иерархии:

```
2001:db8::/32        - весь SRv6 домен
2001:db8:0::/40      - Flex-Algo 0 (default)
2001:db8:1::/40      - Flex-Algo 1 (low-latency)
2001:db8:0:10::/48   - POD-1
2001:db8:0:10:1::/64 - Leaf-1 в POD-1
```

### Locators

- /48 для uSID, /64 для full-length SID
- Одинаковая длина locator во всем домене
- Один locator на Flex-Algo (разные пути - разные locators)
- Не менять locator после деплоя (ломает все SID)

### Function Allocation

- Разделить function space на static и dynamic ranges
- Static (например 0x0001-0x00FF) - для manually assigned SID
- Dynamic (0x0100-0xFFFF) - для auto-allocated (VRF, EVPN)
- Документировать static assignments

### Underlay

- IS-IS предпочтительнее OSPF (лучше SRv6 extensions, TLV flexibility)
- Отдельный IS-IS process/instance для SRv6 (не мешать с legacy)
- TI-LFA на всех интерфейсах
- BFD с aggressive timers (50ms x 3)
- Micro-loop avoidance если большой домен

### Overlay

- BGP EVPN для L2/L3 services
- Минимум 2 Route Reflector (redundancy)
- RR в отдельной plane от data traffic
- RT-based filtering - import только нужные VRF
- Не анонсировать все во все (full mesh RT = проблемы)

### MTU

- Базовый overhead: IPv6 header 40 байт
- SRH: 8 байт header + 16 байт на каждый SID
- С uSID: 8 байт header + 16 байт carrier (до 6 uSID)
- Планировать MTU 9216 на fabric, 9000 к серверам
- Проверять path MTU до включения сервисов

### Мониторинг

- Performance Measurement для delay/loss на SRv6 paths
- Delay metrics как input для Flex-Algo
- TWAMP-Light между PE для SLA мониторинга
- Отслеживать SID allocation (не закончился ли function range)

## RFC

- RFC 8986 - SRv6 Network Programming
- RFC 8754 - Segment Routing Header
- RFC 9252 - BGP Overlay Services over SRv6
- RFC 9602 - SRv6 SID Addressing
