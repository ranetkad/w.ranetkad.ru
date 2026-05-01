---
title: "Network tools"
weight: 10
tags: [tools, troubleshooting, tcpdump, ss]
---

## Коротко

Cheatsheet утилит для отладки сети.

## tcpdump

```
tcpdump -i any -nn -vv 'tcp port 179'
tcpdump -i eth0 -nn 'host 10.0.0.1 and (tcp port 22 or icmp)'
tcpdump -i eth0 -w /tmp/capture.pcap -C 100 -W 5
tcpdump -r /tmp/capture.pcap 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

BPF-фильтры. `tcp[13] & 0x12 = 2` ловит SYN без ACK, `vlan and host 10.0.0.1` фильтрует tagged трафик до конкретного host.

## ss

```
ss -tulpn                       # listening tcp/udp с PID
ss -i                           # tcp_info: rtt, cwnd, retrans
ss -tan state established       # активные соединения
ss -K dport = 80                # kill connections (kernel 4.9+)
```

## ip / iproute2

```
ip -br -c addr
ip -br link
ip route show table all
ip route get 8.8.8.8
ip neigh show
ip rule list
ip netns list / ip netns exec <ns> <cmd>
```

## conntrack

```
conntrack -L
conntrack -E -e NEW             # event stream
conntrack -D -p tcp --orig-port-dst 80
```

## ethtool

```
ethtool eth0                    # link
ethtool -i eth0                 # driver
ethtool -S eth0                 # statistics
ethtool -k eth0                 # offloads (TSO, GRO, GSO, LRO)
ethtool -g eth0                 # ring buffer sizes
ethtool -l eth0                 # combined channels
```

## perf / bpftrace

```
perf top -e cycles
perf record -e syscalls:sys_enter_sendmsg -ag
bpftrace -e 'kprobe:tcp_sendmsg { @[comm] = count(); }'
```

## mtr

```
mtr -bzwc 100 8.8.8.8           # IP+ASN, batch, wide
mtr -T -P 443 example.com       # TCP probe
```
