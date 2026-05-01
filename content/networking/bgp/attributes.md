---
title: "Атрибуты"
---

# Атрибуты

## Коротко

Атрибуты определяют характеристики маршрута и влияют на выбор лучшего пути.
Well-known (обязательные) vs Optional. Transitive (передаются дальше) vs Non-transitive.

## Классификация

**Well-known Mandatory** - обязательны, всегда присутствуют:

- ORIGIN
- AS_PATH
- NEXT_HOP

**Well-known Discretionary** - известны всем, но опциональны:

- LOCAL_PREF
- ATOMIC_AGGREGATE

**Optional Transitive** - опциональны, передаются дальше:

- AGGREGATOR
- COMMUNITY

**Optional Non-transitive** - опциональны, не передаются:

- MED
- ORIGINATOR_ID
- CLUSTER_LIST

## AS_PATH

Список AS, через которые прошел маршрут. Защита от петель.

- При eBGP: добавляется своя AS слева
- При iBGP: не меняется
- Если своя AS в пути - маршрут отбрасывается

**AS_PATH prepend** - добавление своей AS несколько раз для удлинения пути:

<details>
<summary>Juniper</summary>

```
set policy-options policy-statement PREPEND term 1 then as-path-prepend "65000 65000 65000"
```

</details>

<details>
<summary>Huawei</summary>

```
route-policy PREPEND permit node 10
  apply as-path 65000 65000 65000 additive
```

</details>

## NEXT_HOP

IP следующего хопа. Должен быть достижим через IGP.

- eBGP: меняется на IP отправителя
- iBGP: не меняется (проблема - нужен IGP до eBGP-соседа)

**next-hop-self** - принудительно ставить свой IP:

<details>
<summary>Juniper</summary>

```
set protocols bgp group INTERNAL export NHS
set policy-options policy-statement NHS then next-hop self
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  peer IBGP-GROUP next-hop-local
```

</details>

## LOCAL_PREF

Приоритет пути внутри AS. Выше - лучше. По умолчанию 100.
Распространяется только внутри AS (iBGP).

Управление исходящим трафиком - какой выход из AS предпочтительнее.

<details>
<summary>Juniper</summary>

```
set policy-options policy-statement SET-LP term 1 from neighbor 10.0.0.1
set policy-options policy-statement SET-LP term 1 then local-preference 200
```

</details>

<details>
<summary>Huawei</summary>

```
route-policy PREFER-ISP1 permit node 10
  apply local-preference 200
#
bgp 65000
  peer 10.0.0.1 route-policy PREFER-ISP1 import
```

</details>

## MED (Multi-Exit Discriminator)

Подсказка соседней AS какой вход предпочтительнее. Ниже - лучше.
Non-transitive - не передается дальше первой AS.

Управление входящим трафиком.

<details>
<summary>Juniper</summary>

```
set policy-options policy-statement SET-MED then metric 50
set protocols bgp group UPSTREAM export SET-MED
```

</details>

<details>
<summary>Huawei</summary>

```
route-policy SET-MED permit node 10
  apply cost 50
#
bgp 65000
  peer 10.0.0.1 route-policy SET-MED export
```

</details>

## ORIGIN

Как маршрут попал в BGP:

- **IGP** (i) - команда network
- **EGP** (e) - устаревший EGP протокол
- **Incomplete** (?) - redistribute

Предпочтение: IGP > EGP > Incomplete.

## Weight (Cisco-specific)

Локальный атрибут, не передается. Выше - лучше.
На Juniper/Huawei аналога нет - используй LOCAL_PREF.

## Суммаризация

**aggregate** - создание суммарного маршрута.

<details>
<summary>Juniper</summary>

```
set routing-options aggregate route 10.0.0.0/16

set policy-options policy-statement AGGREGATE term 1 from protocol aggregate
set policy-options policy-statement AGGREGATE term 1 then accept
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  aggregate 10.0.0.0 255.255.0.0
```

</details>

**as-set** - сохранение AS_PATH компонентов в суммарном маршруте:

<details>
<summary>Juniper</summary>

```
set routing-options aggregate route 10.0.0.0/16 as-path origin IGP
set routing-options aggregate route 10.0.0.0/16 as-path path "65001 65002"
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  aggregate 10.0.0.0 255.255.0.0 as-set
```

</details>

**suppress-map / detail-suppressed** - подавление более специфичных маршрутов:

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  aggregate 10.0.0.0 255.255.0.0 detail-suppressed
```

</details>
