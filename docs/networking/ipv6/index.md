# IPv6

## Коротко

128-битные адреса (2^128). Заголовок упрощен: 8 полей вместо 13 в IPv4, фиксированный размер 40 байт. Нет broadcast - вместо него multicast. Нет ARP - вместо него NDP (Neighbor Discovery Protocol) на базе ICMPv6.

## Формат адреса

8 групп по 4 hex-символа, разделенных двоеточием:

```
2001:0db8:0000:0000:0000:ff00:0042:8329
```

Правила сокращения:

- Ведущие нули в группе можно убирать: `0db8` -> `db8`
- Последовательные группы нулей заменяются на `::` (один раз): `0000:0000:0000` -> `::`

Результат: `2001:db8::ff00:42:8329`

## Типы адресов

**Unicast** - один-к-одному:

- Global Unicast (2000::/3) - публичные адреса
- Link-Local (FE80::/10) - только в пределах сегмента, не маршрутизируются
- Unique Local (FD00::/8) - аналог RFC1918 (10.x, 172.16.x, 192.168.x)

**Multicast** (FF00::/8) - один-ко-многим. Scope задается 4-м октетом:

- FF02::1 - все узлы в сегменте
- FF02::2 - все роутеры в сегменте

**Anycast** - один-к-ближайшему. Выделяются из unicast-пространства. Пакет доставляется ближайшему узлу с таким адресом.

**Специальные**:

- `::1` - loopback
- `::` - unspecified (0.0.0.0 в IPv4)

## Заголовок

| Поле | Бит | Описание |
|------|-----|----------|
| Version | 4 | Всегда 6 |
| Traffic Class | 8 | Аналог ToS/DSCP |
| Flow Label | 20 | Идентификатор потока для QoS |
| Payload Length | 16 | Длина данных без заголовка |
| Next Header | 8 | Тип следующего заголовка (TCP=6, UDP=17, ICMPv6=58) |
| Hop Limit | 8 | Аналог TTL |
| Source | 128 | Адрес источника |
| Destination | 128 | Адрес назначения |

Убрано по сравнению с IPv4: Header Checksum, Fragmentation fields, Options. Фрагментация вынесена в Extension Header - роутеры не фрагментируют.

## NDP (Neighbor Discovery Protocol)

Работает поверх ICMPv6. Заменяет ARP, ICMP Router Discovery, ICMP Redirect.

5 типов сообщений:

| Тип | ICMPv6 | Назначение |
|-----|--------|------------|
| Router Solicitation (RS) | 133 | Хост ищет роутеры (dst: FF02::2) |
| Router Advertisement (RA) | 134 | Роутер объявляет себя (dst: FF02::1) |
| Neighbor Solicitation (NS) | 135 | Разрешение адреса (аналог ARP request) |
| Neighbor Advertisement (NA) | 136 | Ответ с MAC-адресом (аналог ARP reply) |
| Redirect | 137 | Указание лучшего next-hop |

Функции NDP:

- Address resolution (замена ARP)
- Router discovery
- Duplicate Address Detection (DAD)
- Neighbor Unreachability Detection (NUD)

## SLAAC

Stateless Address Autoconfiguration (RFC 4862). Хост сам генерирует адрес без DHCP.

Процесс:

1. Хост создает link-local адрес (FE80:: + Interface ID)
2. Выполняет DAD - проверяет уникальность через NS
3. Отправляет RS на FF02::2
4. Получает RA с префиксом сети
5. Формирует global unicast: префикс из RA + Interface ID

**EUI-64** - метод генерации Interface ID из MAC:

1. MAC разбивается пополам: `00:1A:2B` и `3C:4D:5E`
2. Вставляется `FF:FE`: `00:1A:2B:FF:FE:3C:4D:5E`
3. Инвертируется 7-й бит: `02:1A:2B:FF:FE:3C:4D:5E`

**Privacy Extensions** (RFC 4941) - случайный Interface ID вместо EUI-64 для приватности. Включен по умолчанию в Windows, macOS, современных Linux.

## Подсети

Стандартная длина префикса для LAN - /64 (требование SLAAC и NDP).

Nibble boundary - деление по 4-битным границам для читаемости:

- /48 - типичный блок для организации
- /56 - блок для сайта/офиса
- /64 - одна подсеть

Пример: из /48 можно сделать 65536 подсетей /64.

## DHCPv6

Два режима:

**Stateful** - сервер выдает адреса и хранит lease (как DHCPv4).

**Stateless** - адрес через SLAAC, DHCPv6 только для опций (DNS, NTP и др.). RA содержит флаг O (Other config).

Флаги в RA:

- M (Managed) = 1 - используй DHCPv6 для адреса
- O (Other) = 1 - используй DHCPv6 для опций

## Примеры

Просмотр IPv6 адресов:

```bash
ip -6 addr show
```

Таблица соседей (аналог arp -a):

```bash
ip -6 neigh show
```

Ping:

```bash
ping6 2001:db8::1
ping6 fe80::1%eth0  # link-local требует указания интерфейса
```

Traceroute:

```bash
traceroute6 2001:db8::1
```
