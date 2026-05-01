---
title: "netfilter"
weight: 8
tags: [netfilter, iptables, nftables, conntrack, dataplane]
---

## Коротко

Подсистема ядра для фильтрации, NAT и conntrack. Hook-points PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING. Поверх hooks работают iptables, nftables и eBPF.

## Hook order

```
ingress -> PREROUTING -> [routing] -> INPUT -> local
                                   -> FORWARD -> POSTROUTING -> egress
local -> OUTPUT -> [routing] -> POSTROUTING -> egress
```

## iptables vs nftables

| Аспект | iptables | nftables |
|---|---|---|
| backend | x_tables | nf_tables |
| таблицы | filter, nat, mangle, raw, security | произвольные, named |
| family | один на v4 (ip6tables на v6) | inet, ip, ip6, arp, bridge, netdev |
| atomic update | нет, правило за правилом | да, через transactions |
| sets/maps | ipset (внешний) | встроены |
| статус | legacy с RHEL 9 / kernel 5.x | default везде |

## conntrack

`/proc/net/nf_conntrack` - state table.

States. NEW, ESTABLISHED, RELATED, INVALID, UNTRACKED.

| sysctl | Что |
|---|---|
| `net.netfilter.nf_conntrack_max` | размер таблицы |
| `net.netfilter.nf_conntrack_tcp_timeout_established` | дефолт 5 дней |
| `net.netfilter.nf_conntrack_buckets` | bucket count для hashtable |

## NAT

- SNAT/DNAT в PREROUTING/POSTROUTING
- conntrack хранит NAT mapping
- masquerade - SNAT на адрес исходящего интерфейса

## See also

- [IPVS](/linux/ipvs/)
- [Network stack](/linux/network-stack/)
- [Network tools](/linux/tools/)
