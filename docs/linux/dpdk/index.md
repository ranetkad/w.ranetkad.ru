# DPDK

## Коротко

Data Plane Development Kit - набор библиотек для высокопроизводительной обработки пакетов в userspace. Разработан Intel, сейчас под Linux Foundation.

Главная идея: kernel bypass. Пакеты идут напрямую из NIC в приложение, минуя ядро Linux.

## Проблема

Классический путь пакета: NIC -> драйвер ядра -> сетевой стек ядра -> socket API -> приложение.

Проблемы этого пути:
- Прерывание на каждый пакет (или группу)
- Context switch между kernel space и user space
- Копирование данных между буферами
- Выделение/освобождение sk_buff на каждый пакет
- Обработка netfilter/iptables

При 10 Gbit/s и пакетах 64 байта: ~14.88 миллионов пакетов в секунду. Ядро не справляется.

## Решение DPDK

Перенос fast path в userspace:
- Драйвер NIC работает в userspace (PMD)
- Polling вместо прерываний
- Hugepages для памяти (меньше TLB misses)
- Lockless структуры данных
- Batch обработка пакетов
- CPU pinning (lcore привязан к ядру)

Результат: миллионы pps на обычном CPU.

## Архитектура

### EAL (Environment Abstraction Layer)

Инициализация DPDK:
- Выделение hugepages
- Маппинг устройств в userspace
- Привязка потоков к ядрам CPU
- NUMA awareness
- Логирование, таймеры

Скрывает специфику ОС от приложения.

### PMD (Poll Mode Driver)

Userspace драйвер для NIC. Работает без прерываний - постоянно опрашивает (poll) RX/TX дескрипторы.

Один lcore (logical core) крутится в busy loop:
```
while (true) {
    nb_rx = rte_eth_rx_burst(port, queue, pkts, BURST_SIZE);
    process(pkts, nb_rx);
    rte_eth_tx_burst(port, queue, pkts, nb_tx);
}
```

CPU занят на 100%, но латентность минимальна.

### Mempool

Пул заранее выделенных буферов. Буферы создаются при старте, не выделяются динамически.

Использует hugepages (2MB или 1GB страницы). Меньше записей в TLB, меньше cache misses.

### mbuf

Структура для хранения пакета. Аналог sk_buff в ядре, но проще и быстрее. Выделяется из mempool.

### Ring

Lockless FIFO очередь. Multi-producer/multi-consumer. Используется для передачи пакетов между ядрами в pipeline модели.

## Модели обработки

### Run-to-completion

Один lcore делает все: RX -> обработка -> TX. Пакет не покидает ядро.

```
lcore 1: NIC RX -> parse -> forward -> NIC TX
lcore 2: NIC RX -> parse -> forward -> NIC TX
```

Простая модель, хорошая локальность кэша.

### Pipeline

Пакет проходит через несколько ядер. Между ядрами - ring очереди.

```
lcore 1: NIC RX -> ring
lcore 2: ring -> parse -> ring
lcore 3: ring -> forward -> NIC TX
```

Сложнее, но позволяет разделить тяжелую обработку.

## Binding устройств

NIC надо отвязать от kernel драйвера и привязать к DPDK.

Userspace драйверы для доступа к PCI:
- **vfio-pci** - современный, безопасный (IOMMU)
- **uio_pci_generic** - старый, требует root
- **igb_uio** - DPDK-специфичный (deprecated)

После binding: NIC не виден в `ip link`, только через DPDK API.

## Hugepages

DPDK требует hugepages для mempool. Стандартные 4KB страницы создают overhead на TLB.

Типичные размеры:
- 2MB hugepages (по умолчанию на x86)
- 1GB hugepages (для больших mempool)

Память выделяется при старте, не динамически.

## NUMA

DPDK учитывает NUMA топологию:
- Mempool выделяется на нужной NUMA node
- lcore запускается на ядре той же NUMA node что и NIC
- Минимизация cross-NUMA доступов

## Use cases

**Сетевые функции (NFV):**
- vSwitch (OVS-DPDK)
- vRouter (VPP/FD.io)
- Firewall, NAT, Load Balancer

**Телеком:**
- 5G UPF (User Plane Function)
- EPC (Evolved Packet Core)
- BNG (Broadband Network Gateway)

**Безопасность:**
- DDoS mitigation
- IDS/IPS (Suricata с DPDK)
- Deep Packet Inspection

**Инфраструктура:**
- Traffic generator (TRex)
- Packet capture
- Network monitoring

## Ограничения

- NIC не виден в Linux (нет ip link, tcpdump)
- Требует выделенных CPU ядер (100% load)
- Сложнее отладка
- Не все NIC поддерживаются
- Требует hugepages, иногда IOMMU

## Альтернативы

**XDP/eBPF** - обработка в ядре, но очень рано (до сетевого стека). Проще интеграция с Linux.

**AF_XDP** - сокеты с zero-copy из XDP в userspace. Компромисс между DPDK и ядром.

**Netmap** - похожий на DPDK подход, но менее распространен.
