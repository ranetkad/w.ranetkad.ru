# NUMA и сеть

## Коротко

NUMA (Non-Uniform Memory Access) - архитектура многопроцессорных систем, где доступ к "своей" памяти быстрее, чем к "чужой".

Для сети критично: NIC подключен к конкретному CPU socket через PCIe. Если обрабатывать пакеты на "чужом" сокете - latency +50-100 ns, throughput -30-50%.

## Архитектура NUMA

```
┌─────────────────────────────────────────────────────────────────────┐
│                          NUMA Node 0                                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │   CPU 0-15  │    │  L3 Cache   │    │      Local RAM          │  │
│  │             │◄──►│   32 MB     │◄──►│       128 GB            │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘  │
│         │                                         │                 │
│         │              PCIe Root Complex          │                 │
│         │                    │                    │                 │
│         │         ┌──────────┴──────────┐         │                 │
│         │         │     NIC 100G        │         │                 │
│         │         │   (eth0, eth1)      │         │                 │
│         │         └─────────────────────┘         │                 │
└─────────│─────────────────────────────────────────│─────────────────┘
          │                                         │
          │            QPI / UPI / Infinity Fabric  │
          │              (межсокетная связь)        │
          │                 ~100 ns                 │
          │                                         │
┌─────────│─────────────────────────────────────────│─────────────────┐
│         ▼                                         ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐  │
│  │  CPU 16-31  │    │  L3 Cache   │    │      Remote RAM         │  │
│  │             │◄──►│   32 MB     │◄──►│       128 GB            │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘  │
│                          NUMA Node 1                                │
└─────────────────────────────────────────────────────────────────────┘
```

## Почему NUMA влияет на сеть

**Правильно (локальный доступ):**
```
Пакет → NIC → DMA в RAM (Node 0) → IRQ на CPU 0-15 → обработка
                                         ↓
                              Все в одном NUMA node
                              Latency: ~80 ns к RAM
```

**Неправильно (удаленный доступ):**
```
Пакет → NIC → DMA в RAM (Node 0) → IRQ на CPU 16-31 (Node 1!)
                                         ↓
                              CPU идет за данными на Node 0
                              Latency: ~150 ns (QPI hop)
                              + contention на interconnect
```

## Влияние на производительность

| Сценарий | Throughput | Latency |
|----------|------------|---------|
| IRQ + App на локальном node | 100% | baseline |
| IRQ локально, App удаленно | 70-85% | +30-50 ns |
| IRQ удаленно, App локально | 60-80% | +50-70 ns |
| IRQ + App на удаленном node | 50-70% | +80-100 ns |

На 100G разница между правильной и неправильной конфигурацией: 40-100 Gbps vs 60-70 Gbps.

## Диагностика

**Посмотреть NUMA топологию:**

```bash
numactl --hardware
```

```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 16 17 18 19 20 21 22 23
node 0 size: 128000 MB
node 1 cpus: 8 9 10 11 12 13 14 15 24 25 26 27 28 29 30 31
node 1 size: 128000 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

Distance 10 = локальный доступ, 21 = удаленный (2.1x медленнее).

**Посмотреть CPU и NUMA:**

```bash
lscpu | grep -i numa
```

```
NUMA node(s):          2
NUMA node0 CPU(s):     0-7,16-23
NUMA node1 CPU(s):     8-15,24-31
```

**Узнать NUMA node сетевой карты:**

```bash
cat /sys/class/net/eth0/device/numa_node
```

```
0
```

Значение -1 = не привязано (виртуалка или старое железо).

**Визуальная топология (с PCIe устройствами):**

```bash
# Установка
apt install hwloc

# Текстовый вывод
lstopo --no-io

# С устройствами
lstopo

# Сохранить картинку
lstopo --output-format png > topology.png
```

**Проверить IRQ affinity:**

```bash
# Найти IRQ сетевой карты
grep eth0 /proc/interrupts

# Посмотреть на каких CPU обрабатывается
cat /proc/irq/<IRQ_NUM>/smp_affinity_list
```

**Проверить где работает приложение:**

```bash
# PID процесса
ps aux | grep nginx

# На каком NUMA node
numactl --show --pid <PID>

# Или через taskset
taskset -cp <PID>
```

## Настройка

**1. Отключить irqbalance (для ручного контроля):**

```bash
systemctl stop irqbalance
systemctl disable irqbalance
```

**2. Привязать IRQ к локальному NUMA node:**

```bash
# Узнать NUMA node карты
NODE=$(cat /sys/class/net/eth0/device/numa_node)

# Узнать CPU этого node
CPUS=$(numactl --hardware | grep "node $NODE cpus" | cut -d: -f2)

# Найти IRQ
IRQS=$(grep eth0 /proc/interrupts | awk '{print $1}' | tr -d ':')

# Привязать каждый IRQ к CPU из списка
for irq in $IRQS; do
    echo <CPU_MASK> > /proc/irq/$irq/smp_affinity
done
```

**3. Запустить приложение на правильном NUMA node:**

```bash
# Привязать к node 0 (память и CPU)
numactl --cpunodebind=0 --membind=0 ./my_app

# Или только CPU
taskset -c 0-7 ./my_app
```

**4. Для nginx/haproxy - в systemd unit:**

```ini
[Service]
ExecStart=/usr/sbin/nginx
# Привязка к NUMA node 0
CPUAffinity=0-7
NUMAPolicy=bind
NUMAMask=0
```

**5. Использовать set_irq_affinity скрипт от Intel/Mellanox:**

```bash
# Intel ixgbe/i40e
/usr/share/doc/ixgbe/set_irq_affinity.sh eth0

# Mellanox
mlnx_affinity eth0
```

## Автоматизация

**Скрипт для настройки affinity:**

```bash
#!/bin/bash
IFACE=$1
NODE=$(cat /sys/class/net/$IFACE/device/numa_node)

if [ "$NODE" == "-1" ]; then
    echo "NIC $IFACE has no NUMA affinity"
    exit 1
fi

# Получить CPU mask для этого node
CPUMASK=$(cat /sys/devices/system/node/node$NODE/cpumap)

# Применить ко всем IRQ этого интерфейса
for irq in $(grep $IFACE /proc/interrupts | awk '{print $1}' | tr -d ':'); do
    echo $CPUMASK > /proc/irq/$irq/smp_affinity
    echo "IRQ $irq -> NUMA node $NODE (mask $CPUMASK)"
done
```

## Команды

```bash
numactl --hardware
```
Показать NUMA топологию: nodes, CPUs, память, distances.

```bash
numactl --cpunodebind=0 --membind=0 ./app
```
Запустить приложение с привязкой к NUMA node 0.

```bash
numastat
```
Статистика аллокаций памяти по NUMA nodes.

```bash
numastat -p <PID>
```
Статистика памяти конкретного процесса по nodes.

```bash
cat /sys/class/net/eth0/device/numa_node
```
NUMA node сетевой карты.

```bash
lstopo
```
Визуальная топология системы (CPU, cache, PCIe, NUMA).

```bash
cat /proc/irq/<N>/smp_affinity_list
```
На каких CPU обрабатывается IRQ.

```bash
echo 0-7 > /proc/irq/<N>/smp_affinity_list
```
Привязать IRQ к CPU 0-7.

## Проверка

Убедиться что все правильно:

```bash
# 1. NIC на каком node?
cat /sys/class/net/eth0/device/numa_node
# Допустим: 0

# 2. IRQ обрабатываются на этом node?
for irq in $(grep eth0 /proc/interrupts | awk '{print $1}' | tr -d ':'); do
    echo -n "IRQ $irq: "
    cat /proc/irq/$irq/smp_affinity_list
done
# Должны быть CPU из node 0

# 3. Приложение работает на этом node?
taskset -cp $(pgrep nginx | head -1)
# Должны быть CPU из node 0

# 4. Память аллоцирована локально?
numastat -p $(pgrep nginx | head -1)
# Большинство в Node 0
```

## Примеры

**Типичная проблема:**

```bash
# NIC на node 0
cat /sys/class/net/eth0/device/numa_node
0

# Но IRQ раскиданы по всем CPU (irqbalance)
grep eth0 /proc/interrupts
 45: 1000  2000  500  800  ...  eth0-TxRx-0
 46:  900  1500  600  700  ...  eth0-TxRx-1
# Числа примерно равны = IRQ прыгают между CPU
```

**После настройки:**

```bash
# IRQ только на CPU node 0
grep eth0 /proc/interrupts
 45: 50000  0  0  0  0  0  0  0  ...  eth0-TxRx-0
 46: 0  48000  0  0  0  0  0  0  ...  eth0-TxRx-1
# Все прерывания на CPU 0 и 1
```

**Бенчмарк до/после (iperf3, 100G):**

```
До (irqbalance, случайные CPU):
  Bandwidth: 62 Gbps
  CPU: 45% на node0, 35% на node1

После (IRQ + app на node0):
  Bandwidth: 94 Gbps
  CPU: 70% на node0, 5% на node1
```
