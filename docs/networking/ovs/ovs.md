# Open vSwitch

## Коротко

Программный коммутатор уровня L2-L4. Поддерживает OpenFlow, VLAN, bonding, QoS.
Используется в виртуализации (OpenStack, KVM) и SDN.

## Архитектура

Три основных компонента:

**ovsdb-server** - база данных конфигурации. Хранит настройки bridges, ports, interfaces. Конфигурация переживает перезагрузку.

**ovs-vswitchd** - демон коммутатора. Читает конфиг из ovsdb, управляет datapath, обрабатывает OpenFlow.

**datapath** - модуль ядра (или userspace/DPDK). Выполняет быструю пересылку пакетов по закэшированным flow. Если flow нет - отправляет пакет в ovs-vswitchd.

Путь пакета:

1. Пакет приходит на порт
2. Datapath ищет matching flow
3. Есть flow -> выполняет actions в ядре (быстро)
4. Нет flow -> upcall в ovs-vswitchd -> решение -> новый flow в datapath

## Команды

### ovs-vsctl

```bash
ovs-vsctl show
```

Общая информация о bridges и портах.

```bash
ovs-vsctl list-br
```

Список bridges.

```bash
ovs-vsctl list-ports br0
```

Список портов на bridge.

```bash
ovs-vsctl list interface
```

Список всех интерфейсов.

```bash
ovs-vsctl get port eth0 tag
```

Получить VLAN tag порта.

### ovs-ofctl

```bash
ovs-ofctl show br0
```

Информация о bridge и портах (OpenFlow port numbers).

```bash
ovs-ofctl dump-flows br0
```

Все flow rules.

```bash
ovs-ofctl dump-flows br0 table=0
```

Flows из конкретной таблицы.

```bash
ovs-ofctl dump-ports br0
```

Статистика портов (пакеты, байты).

```bash
ovs-ofctl dump-ports-desc br0
```

Описание портов (state, speed).

### ovs-appctl

```bash
ovs-appctl ofproto/trace br0 in_port=1
```

Трассировка пакета через flow tables.

```bash
ovs-appctl fdb/show br0
```

MAC-таблица (FDB).

```bash
ovs-appctl dpctl/show -s
```

Статистика datapath.

```bash
ovs-appctl bridge/dump-flows br0
```

Flows включая скрытые.
