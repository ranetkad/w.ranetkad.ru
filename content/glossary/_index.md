---
title: "Glossary"
weight: 100
toc: false
---

Сокращения и термины из всех разделов wiki.

## Routing и control plane

| Термин | Расшифровка |
| AS | Autonomous System |
| ASN | AS Number |
| AFI | Address Family Identifier |
| SAFI | Subsequent AFI |
| BGP | Border Gateway Protocol |
| eBGP | external BGP, между разными AS |
| iBGP | internal BGP, внутри одной AS |
| MP-BGP | Multiprotocol BGP, RFC 4760 |
| RR | Route Reflector |
| RD | Route Distinguisher |
| RT | Route Target |
| RIB | Routing Information Base |
| FIB | Forwarding Information Base |
| LPM | Longest Prefix Match |
| LFIB | Label FIB (для MPLS) |
| MED | Multi-Exit Discriminator |
| NLRI | Network Layer Reachability Information |
| ORF | Outbound Route Filtering |
| PHP | Penultimate Hop Popping |
| PE | Provider Edge |
| CE | Customer Edge |
| P | Provider (core router) |
| RPKI | Resource Public Key Infrastructure |
| ROA | Route Origin Authorization |

## Overlay и L2VPN

| Термин | Расшифровка |
| EVPN | Ethernet VPN, RFC 7432 |
| EVI | EVPN Instance |
| ESI | Ethernet Segment Identifier |
| VTEP | VXLAN Tunnel Endpoint |
| VNI | VXLAN Network Identifier, 24 bit |
| VXLAN | Virtual eXtensible LAN, RFC 7348 |
| Geneve | Generic Network Virtualization Encapsulation, RFC 8926 |
| IRB | Integrated Routing and Bridging |
| L2VNI | L2 VNI, bridging |
| L3VNI | L3 VNI, routing |
| BUM | Broadcast, Unknown unicast, Multicast |
| DF | Designated Forwarder |

## Segment Routing и MPLS

| Термин | Расшифровка |
| SR | Segment Routing, RFC 8402 |
| SR-MPLS | SR на data plane MPLS, RFC 8660 |
| SRv6 | SR на data plane IPv6, RFC 8754 |
| SID | Segment Identifier |
| SRGB | Segment Routing Global Block |
| SRH | Segment Routing Header (IPv6) |
| TI-LFA | Topology Independent Loop-Free Alternate |
| TE | Traffic Engineering |
| LDP | Label Distribution Protocol, RFC 5036 |
| RSVP-TE | RSVP Traffic Engineering, RFC 3209 |
| BGP-LU | BGP Labeled Unicast, RFC 8277 |

## Linux dataplane

| Термин | Расшифровка |
| NAPI | New API, polling в Linux network drivers |
| GRO | Generic Receive Offload |
| GSO | Generic Segmentation Offload |
| TSO | TCP Segmentation Offload |
| LRO | Large Receive Offload |
| RPS | Receive Packet Steering |
| RFS | Receive Flow Steering |
| XPS | Transmit Packet Steering |
| RSS | Receive Side Scaling |
| XDP | eXpress Data Path |
| DPDK | Data Plane Development Kit |
| PMD | Poll Mode Driver (DPDK) |
| EAL | Environment Abstraction Layer (DPDK) |
| SR-IOV | Single Root I/O Virtualization |
| VF | Virtual Function (SR-IOV) |
| PF | Physical Function (SR-IOV) |

## Виртуализация и контейнеры

| Термин | Расшифровка |
| KVM | Kernel-based Virtual Machine |
| QEMU | Quick Emulator |
| virtio | paravirtual device interface |
| vhost | offload virtio в kernel |
| vfio | Virtual Function IO, passthrough |
| IOMMU | Input-Output Memory Management Unit |
| EPT | Extended Page Tables (Intel) |
| NPT | Nested Page Tables (AMD) |
| ns | Linux namespace |
| cgroup | control group |
| OCI | Open Container Initiative |

## Kubernetes

| Термин | Расшифровка |
| CNI | Container Network Interface |
| CRI | Container Runtime Interface |
| CSI | Container Storage Interface |
| HPA | Horizontal Pod Autoscaler |
| kubelet | node agent |
| kube-proxy | service routing на nodes |
| etcd | distributed key-value store |
| OVN-Kubernetes | OVN-based CNI |

## Storage

| Термин | Расшифровка |
| OSD | Object Storage Daemon (Ceph) |
| MON | Monitor (Ceph) |
| MGR | Manager (Ceph) |
| MDS | Metadata Server (Ceph) |
| RGW | RADOS Gateway (Ceph S3/Swift) |
| RBD | RADOS Block Device |
| CRUSH | Controlled Replication Under Scalable Hashing |
| PG | Placement Group (Ceph) |
| PV | Physical Volume (LVM) |
| VG | Volume Group (LVM) |
| LV | Logical Volume (LVM) |

## L4 и HA

| Термин | Расшифровка |
| VRRP | Virtual Router Redundancy Protocol, RFC 5798 |
| IPVS | IP Virtual Server |
| LVS | Linux Virtual Server |
| WLC | Weighted Least Connections (IPVS scheduler) |
| WRR | Weighted Round Robin |
| DR | Direct Routing (IPVS mode) |
| TUN | IP tunneling (IPVS mode) |
| NAT | Network Address Translation |
| LACP | Link Aggregation Control Protocol, IEEE 802.3ad |
| MLAG | Multi-Chassis Link Aggregation |

## Транспорт

| Термин | Расшифровка |
| MTU | Maximum Transmission Unit |
| MSS | Maximum Segment Size |
| BBR | Bottleneck Bandwidth and RTT (TCP CC) |
| CUBIC | TCP CC algorithm |
| RTO | Retransmission Timeout |
| RTT | Round Trip Time |
| ECN | Explicit Congestion Notification |
| TLS | Transport Layer Security |
| ALPN | Application-Layer Protocol Negotiation |
| SNI | Server Name Indication |
