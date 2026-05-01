---
title: "EVPN"
weight: 2
tags: [evpn, bgp, vxlan, dataplane]
---

## Коротко

EVPN (Ethernet VPN, RFC 7432) - control plane на BGP для L2/L3 overlay сетей. Заменяет flood-and-learn в VXLAN: MAC/IP анонсируются через MP-BGP вместо BUM-traffic.

Underlay - IP fabric (обычно eBGP unnumbered). Overlay - VXLAN (data plane), EVPN (control plane).

## Address family

AFI 25 (L2VPN), SAFI 70 (EVPN).

## Route types

| Type | Назначение |
| 1 | Ethernet Auto-Discovery, multi-homing |
| 2 | MAC/IP Advertisement |
| 3 | Inclusive Multicast Ethernet Tag (BUM, replication list) |
| 4 | Ethernet Segment, multi-homing DF election |
| 5 | IP Prefix Route (L3VPN-like) |

## Базовые сущности

| Термин | Что это |
| RD | Route Distinguisher, делает префикс уникальным в BGP |
| RT | Route Target, политика import/export между VRF/EVI |
| EVI | EVPN Instance |
| ESI | Ethernet Segment Identifier |
| VTEP | VXLAN Tunnel Endpoint |
| VNI | VXLAN Network Identifier (24 бит) |

## Сценарии

| Сценарий | Что |
| L2VNI | bridging одного broadcast domain через fabric |
| L3VNI | routing между VRF поверх fabric |
| Symmetric IRB | route-and-bridge симметричный, маршрутизация на ingress и egress VTEP |
| Asymmetric IRB | route-only-on-ingress, на egress только bridging |
| Multi-homing all-active | через Type-1 и Type-4, без блокировки линков как в STP |

## See also

- [BGP](/networking/bgp/)
- [OVN](/networking/ovn/)
- [Segment Routing](/networking/segment-routing/)
