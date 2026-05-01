---
title: "KVM"
weight: 9
tags: [kvm, qemu, libvirt]
---

## Коротко

Гипервизор в Linux kernel. CPU через Intel VT-x / AMD-V, память через EPT/NPT.

QEMU делает device emulation и IO в userspace, KVM - CPU и memory в kernel.

libvirt - management layer поверх QEMU/KVM с единым API.

## Стек

```
guest VM
  |
QEMU (vCPU threads, devices)
  |
/dev/kvm  (ioctl) -> KVM module in kernel
  |
hardware (VT-x/AMD-V, EPT/NPT, IOMMU)
```

## Базовые сущности

| Термин | Что |
|---|---|
| domain | виртуальная машина в терминах libvirt |
| vCPU | поток QEMU привязанный к KVM_RUN ioctl |
| virtio | paravirtual device interface (net, block, scsi, fs) |
| vhost | offload virtio в kernel (vhost-net, vhost-scsi) |
| vfio | passthrough host PCI device в гостя |
| SR-IOV | hardware partitioning одной NIC на VF |

## virsh

```
virsh list --all
virsh dominfo <domain>
virsh dumpxml <domain>
virsh start/shutdown/destroy <domain>
virsh attach-device / detach-device
virsh console <domain>
```

## XML domain

`/etc/libvirt/qemu/<domain>.xml` - persistent definition.

## See also

- [Namespaces](/linux/namespaces/)
- [NUMA](/linux/numa/)
- [OVS](/networking/ovs/)
- [DPDK](/linux/dpdk/)
