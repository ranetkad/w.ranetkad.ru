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

??? example "Juniper"
    ```
    set policy-options policy-statement PREPEND term 1 then as-path-prepend "65000 65000 65000"
    ```

??? example "Huawei"
    ```
    route-policy PREPEND permit node 10
      apply as-path 65000 65000 65000 additive
    ```

## NEXT_HOP

IP следующего хопа. Должен быть достижим через IGP.

- eBGP: меняется на IP отправителя
- iBGP: не меняется (проблема - нужен IGP до eBGP-соседа)

**next-hop-self** - принудительно ставить свой IP:

??? example "Juniper"
    ```
    set protocols bgp group INTERNAL export NHS
    set policy-options policy-statement NHS then next-hop self
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      peer IBGP-GROUP next-hop-local
    ```

## LOCAL_PREF

Приоритет пути внутри AS. Выше - лучше. По умолчанию 100.
Распространяется только внутри AS (iBGP).

Управление исходящим трафиком - какой выход из AS предпочтительнее.

??? example "Juniper"
    ```
    set policy-options policy-statement SET-LP term 1 from neighbor 10.0.0.1
    set policy-options policy-statement SET-LP term 1 then local-preference 200
    ```

??? example "Huawei"
    ```
    route-policy PREFER-ISP1 permit node 10
      apply local-preference 200
    #
    bgp 65000
      peer 10.0.0.1 route-policy PREFER-ISP1 import
    ```

## MED (Multi-Exit Discriminator)

Подсказка соседней AS какой вход предпочтительнее. Ниже - лучше.
Non-transitive - не передается дальше первой AS.

Управление входящим трафиком.

??? example "Juniper"
    ```
    set policy-options policy-statement SET-MED then metric 50
    set protocols bgp group UPSTREAM export SET-MED
    ```

??? example "Huawei"
    ```
    route-policy SET-MED permit node 10
      apply cost 50
    #
    bgp 65000
      peer 10.0.0.1 route-policy SET-MED export
    ```

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

??? example "Juniper"
    ```
    set routing-options aggregate route 10.0.0.0/16

    set policy-options policy-statement AGGREGATE term 1 from protocol aggregate
    set policy-options policy-statement AGGREGATE term 1 then accept
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      aggregate 10.0.0.0 255.255.0.0
    ```

**as-set** - сохранение AS_PATH компонентов в суммарном маршруте:

??? example "Juniper"
    ```
    set routing-options aggregate route 10.0.0.0/16 as-path origin IGP
    set routing-options aggregate route 10.0.0.0/16 as-path path "65001 65002"
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      aggregate 10.0.0.0 255.255.0.0 as-set
    ```

**suppress-map / detail-suppressed** - подавление более специфичных маршрутов:

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      aggregate 10.0.0.0 255.255.0.0 detail-suppressed
    ```
