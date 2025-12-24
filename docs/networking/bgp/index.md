# Обзор

## Коротко

Border Gateway Protocol - протокол маршрутизации между автономными системами (AS).
Работает поверх TCP:179. Path-vector протокол - выбирает маршрут на основе атрибутов, а не метрики.

Когда нужен:
- Подключение к нескольким провайдерам (multihoming)
- Управление входящим/исходящим трафиком
- Обмен маршрутами между AS

## MP-BGP

Multiprotocol BGP (RFC 4760) - расширение для поддержки разных протоколов и типов маршрутов.
Классический BGP умел только IPv4 unicast. MP-BGP добавляет address families.

## AFI/SAFI

**AFI** (Address Family Identifier) - тип адресации:

- 1 - IPv4
- 2 - IPv6

**SAFI** (Subsequent AFI) - тип маршрутизации:

- 1 - Unicast
- 2 - Multicast
- 128 - MPLS VPN

Комбинации:

- AFI 1, SAFI 1 - IPv4 Unicast (классический BGP)
- AFI 1, SAFI 128 - VPNv4 (L3VPN)
- AFI 2, SAFI 1 - IPv6 Unicast
- AFI 2, SAFI 128 - VPNv6

## eBGP vs iBGP

**eBGP** (external):

- Соседи из разных AS
- TTL = 1 по умолчанию
- Меняет NEXT_HOP на себя
- Добавляет свою AS в AS_PATH

**iBGP** (internal):

- Соседи внутри одной AS
- TTL = 255
- Не меняет NEXT_HOP
- Не добавляет AS в AS_PATH
- Требует full-mesh или RR/confederation

## Выбор лучшего маршрута

Порядок сравнения (сверху вниз, до первого отличия):

1. Weight (выше лучше, Cisco-specific)
2. LOCAL_PREF (выше лучше)
3. Locally originated
4. AS_PATH (короче лучше)
5. ORIGIN (IGP > EGP > Incomplete)
6. MED (ниже лучше)
7. eBGP > iBGP
8. IGP metric до NEXT_HOP (ниже лучше)
9. Oldest route
10. Lowest Router ID
11. Lowest neighbor IP
