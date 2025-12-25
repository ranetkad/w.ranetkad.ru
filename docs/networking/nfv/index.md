# NFV (Network Function Virtualization)

## Коротко

NFV - перенос сетевых функций (firewall, load balancer, router, NAT, VPN) с железных appliances на виртуальные машины или контейнеры поверх стандартных x86 серверов.

Цель: уйти от vendor lock-in, снизить CAPEX/OPEX, ускорить деплой новых сервисов.

SDN и NFV - дополняющие технологии: SDN отделяет control plane от data plane и управляет сетью программно, NFV виртуализирует сами сетевые функции. SDN часто используется для связи VNF между собой (service chaining).

## Архитектура ETSI NFV

Стандарт ETSI определяет три основных компонента:

**NFVI (NFV Infrastructure)** - инфраструктурный слой:

- compute/storage/network ресурсы (обычные серверы)
- гипервизор или container runtime
- виртуальные ресурсы для VNF

**VNF (Virtual Network Function)** - виртуализированная сетевая функция:

- vRouter, vFirewall, vLB, vNAT, vVPN
- работает на VM или в контейнере
- заменяет физический appliance

**MANO (Management and Orchestration)** - управление:

- NFVO (NFV Orchestrator) - оркестрация сервисов, lifecycle NS
- VNFM (VNF Manager) - lifecycle VNF
- VIM (Virtualized Infrastructure Manager) - управление инфраструктурой (OpenStack, VMware)

```
+------------------------------------------+
|              MANO                        |
|  +--------+  +--------+  +--------+      |
|  |  NFVO  |  |  VNFM  |  |  VIM   |      |
|  +--------+  +--------+  +--------+      |
+------------------------------------------+
                    |
+------------------------------------------+
|              VNF Layer                   |
|  [vRouter] [vFirewall] [vLB] [vNAT]      |
+------------------------------------------+
                    |
+------------------------------------------+
|              NFVI                        |
|  Hypervisor / Container Runtime          |
|  Compute | Storage | Network             |
+------------------------------------------+
```

## VNF vs CNF

**VNF (Virtual Network Function)** - сетевая функция в VM:

- тяжелый: гостевая ОС + гипервизор
- оркестрация через NFV-MANO (OpenStack/VMware)
- медленный старт (минуты)

**CNF (Cloud Native Network Function)** - сетевая функция в контейнере:

- легкий: только контейнер, без guest OS
- оркестрация через Kubernetes
- быстрый старт (секунды)
- микросервисная архитектура
- интеграция с CI/CD

CNF - эволюция VNF для cloud-native инфраструктуры. Современные проекты предпочитают CNF.

## Use Cases в DC/Cloud

**Service Chaining** - цепочка VNF для обработки трафика:

```
Traffic -> vFirewall -> vLB -> vWAF -> App
```

SDN контроллер управляет маршрутизацией между VNF.

**vCPE (virtual Customer Premise Equipment)** - виртуальный роутер/firewall для клиента вместо железки на площадке.

**Network Segmentation** - vFirewall между сегментами в DC без физических appliances.

**Elastic Scaling** - автоматический scale out/in VNF под нагрузку (через MANO или Kubernetes для CNF).

## Примеры реализаций

**VIM:**

- OpenStack (основной выбор для NFV)
- VMware vCloud NFV

**VNF/CNF:**

- Open vSwitch (vSwitch)
- VPP (Vector Packet Processing) - высокопроизводительный data plane
- Cilium, Calico - CNI для Kubernetes с network policy

**MANO:**

- ONAP (Open Network Automation Platform)
- OSM (Open Source MANO от ETSI)

## Ссылки

- [ETSI NFV](https://www.etsi.org/technologies/nfv)
- [ETSI GS NFV-MAN 001](https://www.etsi.org/deliver/etsi_gs/NFV-MAN/001_099/001/01.01.01_60/gs_NFV-MAN001v010101p.pdf)
