# Соседство

## Коротко

BGP-сессия устанавливается поверх TCP:179.
Соседи обмениваются OPEN, согласовывают параметры, затем UPDATE с маршрутами.

## Типы сообщений

**OPEN** - первое сообщение после TCP handshake:

- BGP version (4)
- Local AS number
- Hold Time (по умолчанию 180 сек)
- BGP Identifier (Router ID)
- Capabilities (MP-BGP, route refresh и др.)

**UPDATE** - обмен маршрутной информацией:

- Withdrawn Routes - отзыв маршрутов
- Path Attributes - атрибуты пути
- NLRI - анонсируемые префиксы

**KEEPALIVE** - поддержание сессии:

- Отправляется каждые 1/3 от Hold Time (по умолчанию 60 сек)
- Если 3 KEEPALIVE пропущены - сессия падает

**NOTIFICATION** - ошибка, сессия закрывается:

- Hold Timer Expired
- Cease (административный сброс)
- Bad AS, Bad Router ID и др.

## FSM (Finite State Machine)

1. **Idle** - начальное состояние, ничего не делает
2. **Connect** - ждет TCP-соединение
3. **Active** - пытается установить TCP
4. **OpenSent** - отправил OPEN, ждет ответ
5. **OpenConfirm** - получил OPEN, ждет KEEPALIVE
6. **Established** - сессия установлена, обмен маршрутами

Коллизия: если оба инициируют TCP - остается соединение от роутера с большим Router ID.

## Аутентификация

MD5 - подпись TCP-сегментов общим ключом. Защита от спуфинга.

??? example "Juniper"
    ```
    set protocols bgp group PEERS authentication-key "secret123"
    ```

??? example "Huawei"
    ```
    bgp 65000
      peer 10.0.0.1 password cipher secret123
    ```

## Примеры конфигурации

### eBGP

??? example "Juniper"
    ```
    set protocols bgp group UPSTREAM type external
    set protocols bgp group UPSTREAM peer-as 65001
    set protocols bgp group UPSTREAM neighbor 192.168.1.1
    ```

??? example "Huawei"
    ```
    bgp 65000
      peer 192.168.1.1 as-number 65001
    #
    ipv4-family unicast
      peer 192.168.1.1 enable
    ```

### iBGP (через loopback)

??? example "Juniper"
    ```
    set protocols bgp group INTERNAL type internal
    set protocols bgp group INTERNAL local-address 10.255.0.1
    set protocols bgp group INTERNAL neighbor 10.255.0.2
    ```

??? example "Huawei"
    ```
    bgp 65000
      peer 10.255.0.2 as-number 65000
      peer 10.255.0.2 connect-interface LoopBack0
    #
    ipv4-family unicast
      peer 10.255.0.2 enable
    ```

### eBGP multihop

Для соседства через loopback между AS (TTL > 1).

??? example "Juniper"
    ```
    set protocols bgp group EBGP-MULTIHOP multihop ttl 2
    set protocols bgp group EBGP-MULTIHOP local-address 10.255.0.1
    ```

??? example "Huawei"
    ```
    bgp 65000
      peer 10.255.1.1 ebgp-max-hop 2
      peer 10.255.1.1 connect-interface LoopBack0
    ```
