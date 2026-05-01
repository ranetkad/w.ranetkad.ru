---
title: "MPLS"
weight: 3
tags: [mpls, bgp, routing]
---

## Коротко

Multiprotocol Label Switching - инкапсуляция с label stack между L2 и L3. Forwarding по меткам (LFIB), а не по IP. RFC 3031.

Label - 20 бит, EXP - 3 (QoS), S - 1 (bottom-of-stack), TTL - 8.

## Сигналинг меток

| Протокол | Что делает |
| LDP | hop-by-hop signaling, простой, RFC 5036 |
| RSVP-TE | TE-туннели с резервированием, RFC 3209 |
| BGP-LU | labeled unicast, RFC 8277, для inter-AS L3VPN и SR-MPLS |
| Segment Routing | source routing через стек меток (SR-MPLS, RFC 8660) |

## Reserved labels

| Label | Имя | Смысл |
| 0 | Explicit NULL IPv4 | PHP off, get IPv4 |
| 1 | Router Alert | |
| 2 | Explicit NULL IPv6 | |
| 3 | Implicit NULL | PHP, label pop на penultimate hop |

## Применения

| Применение | Что |
| L3VPN | VPNv4/VPNv6 с PE-CE, RFC 4364 |
| L2VPN | VPLS и EVPN over MPLS |
| TE | explicit paths, FRR |
| SR-MPLS | segment routing на стеке меток |

## See also

- [BGP](/networking/bgp/)
- [Segment Routing](/networking/segment-routing/)
- [EVPN](/networking/evpn/)
