---
title: "Дополнительно"
---

# Дополнительно

## Коротко

Multipath, communities, add-path, BFD, route refresh и другие фичи для production.

## Multipath

Установка нескольких путей в RIB для балансировки. По умолчанию BGP ставит только лучший маршрут.

**eBGP multipath** - несколько путей через разные AS:

<details>
<summary>Juniper</summary>

```
set routing-options autonomous-system 65000
set protocols bgp group UPSTREAM multipath
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  maximum load-balancing ebgp 4
```

</details>

**iBGP multipath** - несколько путей внутри AS:

<details>
<summary>Juniper</summary>

```
set protocols bgp group INTERNAL multipath
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  maximum load-balancing ibgp 4
```

</details>

**eiBGP multipath** - смешанный (eBGP + iBGP):

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  maximum load-balancing 4
```

</details>

## Communities

Метки для группировки маршрутов и применения политик.

**Форматы:**

- Standard: `AS:VALUE` (16 bit каждое) - RFC 1997
- Extended: 6 байт - используется в VPN
- Large: `AS:VALUE1:VALUE2` (32 bit AS) - RFC 8092

**Well-known communities:**

- `no-export` - не анонсировать за пределы AS
- `no-advertise` - не анонсировать никому
- `no-export-subconfed` - не анонсировать за пределы sub-AS

<details>
<summary>Juniper</summary>

```
# Установка community
set policy-options community MY-COMM members 65000:100

set policy-options policy-statement SET-COMM term 1 then community add MY-COMM

# Фильтрация по community
set policy-options policy-statement MATCH-COMM term 1 from community MY-COMM
set policy-options policy-statement MATCH-COMM term 1 then accept
```

</details>

<details>
<summary>Huawei</summary>

```
# Установка community
route-policy SET-COMM permit node 10
  apply community 65000:100

# Фильтрация по community
ip community-filter basic MY-COMM permit 65000:100
route-policy MATCH-COMM permit node 10
  if-match community-filter MY-COMM
```

</details>

**no-export пример:**

<details>
<summary>Juniper</summary>

```
set policy-options policy-statement NO-EXPORT then community add no-export
```

</details>

<details>
<summary>Huawei</summary>

```
route-policy NO-EXPORT permit node 10
  apply community no-export
```

</details>

## Add-Path

Анонс нескольких путей к одному префиксу. Полезно для RR - клиенты получают альтернативы.

<details>
<summary>Juniper</summary>

```
set protocols bgp group RR-CLIENTS family inet unicast add-path send path-count 2
set protocols bgp group RR-CLIENTS family inet unicast add-path receive
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  peer RR-CLIENTS advertise-ext-community
  peer RR-CLIENTS advertise-community
  peer RR-CLIENTS additional-paths send
  peer RR-CLIENTS additional-paths receive
```

</details>

## BFD (Bidirectional Forwarding Detection)

Быстрое обнаружение падения соседа (миллисекунды вместо Hold Timer).

<details>
<summary>Juniper</summary>

```
set protocols bgp group UPSTREAM bfd-liveness-detection minimum-interval 300
set protocols bgp group UPSTREAM bfd-liveness-detection multiplier 3
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
  peer 10.0.0.1 bfd enable
  peer 10.0.0.1 bfd min-tx-interval 300 min-rx-interval 300 detect-multiplier 3
```

</details>

## Route Refresh

Запрос повторной отправки маршрутов от соседа без сброса сессии.

<details>
<summary>Juniper</summary>

```
# В CLI
clear bgp neighbor 10.0.0.1 soft-inbound
```

</details>

<details>
<summary>Huawei</summary>

```
# В CLI
refresh bgp 10.0.0.1 ipv4 unicast import
```

</details>

## Graceful Restart

Сохранение forwarding при перезапуске BGP-процесса. Сосед не удаляет маршруты.

<details>
<summary>Juniper</summary>

```
set routing-options graceful-restart
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
  graceful-restart
  graceful-restart timer restart 120
```

</details>

## Next-hop tracking

Отслеживание доступности NEXT_HOP через IGP. При недоступности - маршрут удаляется.

<details>
<summary>Juniper</summary>

```
# Включено по умолчанию
set protocols bgp group INTERNAL family inet unicast nexthop-resolution
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  nexthop recursive-lookup route-policy CHECK-NH
```

</details>

## Selective route download

Установка в RIB только нужных маршрутов (экономия памяти).

<details>
<summary>Juniper</summary>

```
set protocols bgp group UPSTREAM family inet unicast rib-group SELECTIVE
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
#
ipv4-family unicast
  peer 10.0.0.1 route-policy ACCEPT-ONLY-DEFAULT import
```

</details>
