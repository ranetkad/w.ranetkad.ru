# Анонсирование

## Коротко

NLRI (Network Layer Reachability Information) - префиксы, которые BGP анонсирует соседям.
Два способа: явное объявление (network) и редистрибуция из других протоколов.

## network

Явное добавление префикса в BGP. Префикс должен существовать в RIB (таблице маршрутизации).

??? example "Juniper"
    ```
    set policy-options policy-statement EXPORT-DIRECT term 1 from protocol direct
    set policy-options policy-statement EXPORT-DIRECT term 1 from route-filter 10.0.0.0/24 exact
    set policy-options policy-statement EXPORT-DIRECT term 1 then accept

    set protocols bgp group PEERS export EXPORT-DIRECT
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      network 10.0.0.0 255.255.255.0
    ```

Важно: в Juniper нет команды network - используется export policy.

## redistribute (import из IGP)

Импорт маршрутов из других протоколов (OSPF, IS-IS, static, connected).

??? example "Juniper"
    ```
    set policy-options policy-statement REDIST-OSPF term 1 from protocol ospf
    set policy-options policy-statement REDIST-OSPF term 1 then accept

    set protocols bgp group PEERS export REDIST-OSPF
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      import-route ospf 1
      import-route direct
      import-route static
    ```

## Модификация атрибутов при анонсе

Можно менять атрибуты (MED, community, as-path) при экспорте.

??? example "Juniper"
    ```
    set policy-options policy-statement SET-MED term 1 from route-filter 10.0.0.0/24 exact
    set policy-options policy-statement SET-MED term 1 then metric 100
    set policy-options policy-statement SET-MED term 1 then accept
    ```

??? example "Huawei"
    ```
    route-policy SET-MED permit node 10
      if-match ip-prefix MY-PREFIX
      apply cost 100
    #
    bgp 65000
      peer 10.0.0.1 route-policy SET-MED export
    ```

## default-originate

Анонс default route (0.0.0.0/0) соседу.

??? example "Juniper"
    ```
    set policy-options policy-statement SEND-DEFAULT term 1 from route-filter 0.0.0.0/0 exact
    set policy-options policy-statement SEND-DEFAULT term 1 then accept

    set routing-options generate route 0.0.0.0/0 discard
    set protocols bgp group DOWNSTREAM export SEND-DEFAULT
    ```

??? example "Huawei"
    ```
    bgp 65000
    #
    ipv4-family unicast
      peer 10.0.0.1 default-route-advertise
    ```
