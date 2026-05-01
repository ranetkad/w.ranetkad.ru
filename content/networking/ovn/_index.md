---
title: "OVN"
weight: 4
tags: [ovn, ovs, sdn, dataplane]
---

## Коротко

Control plane поверх OVS. OVS остается dataplane, OVN добавляет logical switch, logical router и distributed flows.

## Архитектура

```
ovn-nbctl -> OVN Northbound DB (logical: switches, routers, ACLs)
              |
           ovn-northd (compiles to logical flows)
              |
           OVN Southbound DB (chassis, port-bindings, logical flows)
              |
ovs-vsctl -> ovn-controller (на каждом hypervisor) -> OVS flows
```

## Термины

| | |
|---|---|
| Logical Switch | виртуальный L2 segment, multi-host |
| Logical Router | distributed router, logical |
| Chassis | hypervisor с ovn-controller |
| Port Binding | привязка logical port к chassis |
| Localnet | L2 attachment к физической сети через bridge mapping |
| Geneve | default tunnel encapsulation между chassis |

## Команды

```
ovn-nbctl ls-list / ls-add / ls-del
ovn-nbctl lr-list / lr-add / lr-del
ovn-nbctl lsp-add / lsp-set-addresses
ovn-sbctl show
ovn-sbctl chassis-list
ovn-trace --detailed <ls> <flow>
```

## Применения

- OpenStack Neutron (ML2/OVN driver, заменил ML2/OVS с Yoga)
- Kubernetes CNI (ovn-kubernetes)
- Standalone SDN

## See also

- [OVS](/networking/ovs/)
- [EVPN](/networking/evpn/)
