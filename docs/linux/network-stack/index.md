# Путь пакета в Linux

## Коротко

Пакет проходит: Провод -> NIC -> DMA -> Ring Buffer -> Hard IRQ -> Soft IRQ (NAPI) -> Сетевой стек ядра -> Socket Buffer -> Userspace.

Ключевые механизмы: DMA (без участия CPU), прерывания (hard/soft), NAPI (polling вместо interrupt storm), sk_buff (структура пакета в ядре).

## Полный путь пакета (RX)

```
┌─────────────────────────────────────────────────────────────────┐
│  ФИЗИЧЕСКИЙ УРОВЕНЬ                                             │
├─────────────────────────────────────────────────────────────────┤
│  Провод/оптика                                                  │
│       ↓                                                         │
│  Трансивер (PHY) - преобразование сигнала                       │
│       ↓                                                         │
│  MAC (Media Access Control) - проверка FCS, извлечение фрейма   │
└─────────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────────┐
│  NIC (Network Interface Card)                                   │
├─────────────────────────────────────────────────────────────────┤
│  1. Принимает фрейм в hardware buffer                           │
│  2. Проверяет checksum (если поддерживается)                    │
│  3. DMA: копирует данные в Ring Buffer (RAM)                    │
│  4. Обновляет RX descriptor                                     │
│  5. Генерирует Hardware Interrupt                               │
└─────────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────────┐
│  HARD IRQ (аппаратное прерывание)                               │
├─────────────────────────────────────────────────────────────────┤
│  CPU прерывается, выполняет interrupt handler драйвера:         │
│  - Отключает прерывания от NIC (чтобы не было interrupt storm)  │
│  - Вызывает napi_schedule() - планирует обработку               │
│  - Возвращает управление (минимум работы)                       │
└─────────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────────┐
│  SOFT IRQ / NAPI (отложенная обработка)                         │
├─────────────────────────────────────────────────────────────────┤
│  ksoftirqd обрабатывает NET_RX_SOFTIRQ:                         │
│  - net_rx_action() вызывает poll-функцию драйвера               │
│  - Драйвер читает пакеты из Ring Buffer (polling, не interrupt) │
│  - Создает sk_buff для каждого пакета                           │
│  - Передает в napi_gro_receive() или netif_receive_skb()        │
│  - Когда Ring Buffer пуст - включает прерывания обратно         │
└─────────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────────┐
│  СЕТЕВОЙ СТЕК ЯДРА                                              │
├─────────────────────────────────────────────────────────────────┤
│  netif_receive_skb()                                            │
│       ↓                                                         │
│  Определяет протокол L3 (ETH_P_IP, ETH_P_IPV6...)               │
│       ↓                                                         │
│  ip_rcv() - обработка IP                                        │
│  - Проверка заголовка, checksum                                 │
│  - PREROUTING (netfilter/iptables)                              │
│  - ip_route_input() - lookup маршрута                           │
│  - INPUT или FORWARD                                            │
│       ↓                                                         │
│  tcp_v4_rcv() - обработка TCP                                   │
│  - Поиск сокета по 4-tuple (src ip:port, dst ip:port)           │
│  - TCP state machine (ACK, sequence numbers)                    │
│  - Добавление в sk_receive_queue сокета                         │
│       ↓                                                         │
│  sock_def_readable() - будит процесс                            │
└─────────────────────────────────────────────────────────────────┘
       ↓
┌─────────────────────────────────────────────────────────────────┐
│  USERSPACE                                                      │
├─────────────────────────────────────────────────────────────────┤
│  Процесс разблокируется (epoll_wait/select/read)                │
│       ↓                                                         │
│  recv()/read() - системный вызов                                │
│       ↓                                                         │
│  copy_to_user() - копирование из sk_receive_queue в userspace   │
│       ↓                                                         │
│  Данные доступны приложению                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Ring Buffer

Кольцевой буфер в RAM. NIC пишет туда через DMA. Драйвер читает оттуда.

Структура:
- RX descriptor ring - массив дескрипторов (указатели на буферы + метаданные)
- Буферы данных - память под сами пакеты

```
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  <- дескрипторы
  └─┬─┴─┬─┴───┴───┴───┴─┬─┴─┬─┴───┘
    │   │               │   │
    ↓   ↓               ↓   ↓
  [buf][buf]          [buf][buf]     <- буферы с данными
    ↑                       ↑
    │                       │
  HEAD                    TAIL
  (NIC пишет)         (драйвер читает)
```

Переполнение (overflow) - пакеты теряются. Счетчик: `ethtool -S eth0 | grep rx_missed`.

## sk_buff (skb)

Главная структура для пакета в ядре. Не содержит данные напрямую - только метаданные и указатели.

```
struct sk_buff {
    /* Указатели на заголовки */
    transport_header  -> TCP/UDP header
    network_header    -> IP header
    mac_header        -> Ethernet header

    /* Размеры */
    len               -> общий размер данных
    data_len          -> размер в page fragments

    /* Указатели на данные */
    head              -> начало буфера
    data              -> начало данных
    tail              -> конец данных
    end               -> конец буфера

    /* Очередь */
    next, prev        -> двусвязный список
}
```

## Hard IRQ vs Soft IRQ

**Hard IRQ (top half)**:
- Вызывается железом немедленно
- Прерывает текущую работу CPU
- Нельзя блокироваться
- Минимум работы: acknowledge + schedule soft irq
- Другие прерывания заблокированы

**Soft IRQ (bottom half)**:
- Выполняется ksoftirqd или после hard irq
- Можно прерывать
- Основная обработка пакетов
- Ограничен по времени и количеству пакетов

## NAPI (New API)

Проблема: при высокой нагрузке - interrupt storm (тысячи прерываний в секунду).

Решение NAPI:
1. Первое прерывание будит обработчик
2. Обработчик отключает прерывания
3. Polling: читает пакеты из ring buffer пока есть
4. Когда ring buffer пуст - включает прерывания

При низкой нагрузке: 1 прерывание на пакет.
При высокой: 1 прерывание на множество пакетов.

## Interrupt Coalescing

NIC накапливает несколько пакетов перед генерацией прерывания.

Параметры:
- rx-usecs: ждать N микросекунд
- rx-frames: ждать N пакетов

Компромисс: меньше прерываний = выше throughput, но выше latency.

## Hardware Offloads

NIC берет на себя часть работы CPU. Без offloads 100G невозможен - CPU не успеет.

**Checksum Offload**
- rx-checksumming: NIC проверяет checksum, ядро не пересчитывает
- tx-checksumming: NIC считает checksum при отправке

**Segmentation Offload**
- TSO (TCP Segmentation Offload): приложение отдает большой chunk (64KB), NIC сам нарезает на MSS-сегменты
- GSO (Generic Segmentation Offload): то же, но в софте (fallback если нет TSO)
- UFO (UDP Fragmentation Offload): аналог для UDP

**Receive Offload**
- GRO (Generic Receive Offload): склеивает мелкие пакеты одного потока в большие перед передачей в стек
- LRO (Large Receive Offload): аналог на железе (агрессивнее, может ломать routing/bridging)

**RSS (Receive Side Scaling)**
- NIC хеширует 5-tuple (src/dst IP, src/dst port, protocol)
- Распределяет пакеты по разным RX-очередям
- Каждая очередь - свой CPU
- 100G карта: 32-128 очередей = параллельная обработка

**Flow Director / ATR**
- NIC запоминает flow и отправляет на тот же CPU где сокет
- Лучше cache locality

```bash
# Посмотреть что включено
ethtool -k eth0

# Включить/выключить
ethtool -K eth0 tso on
ethtool -K eth0 gro on
ethtool -K eth0 rx-checksumming on
```

Типичный вывод `ethtool -k`:
```
rx-checksumming: on
tx-checksumming: on
scatter-gather: on
tcp-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off    # обычно off, ломает forwarding
receive-hashing: on           # RSS
```

**Что дает offload:**

Без TSO (CPU нарезает):
```
App: 64KB данных
    ↓
Kernel: создает ~44 sk_buff по 1460 байт
    ↓
NIC: отправляет 44 пакета
```

С TSO (NIC нарезает):
```
App: 64KB данных
    ↓
Kernel: создает 1 большой sk_buff
    ↓
NIC: сам нарезает и отправляет 44 пакета
```

Экономия: меньше sk_buff allocations, меньше проходов по стеку.

## Команды

```bash
ethtool -g eth0
```
Показать размер ring buffer (текущий и максимальный).

```bash
ethtool -G eth0 rx 4096
```
Увеличить RX ring buffer до 4096.

```bash
ethtool -c eth0
```
Показать настройки interrupt coalescing.

```bash
ethtool -C eth0 rx-usecs 100
```
Установить задержку прерывания 100 мкс.

```bash
ethtool -C eth0 adaptive-rx on
```
Включить адаптивный coalescing.

```bash
ethtool -S eth0 | grep -E "rx_missed|rx_dropped|rx_errors"
```
Проверить потери пакетов на уровне NIC.

```bash
cat /proc/interrupts | grep eth0
```
Прерывания по CPU для сетевой карты.

```bash
cat /proc/net/softnet_stat
```
Статистика softirq (столбцы: processed, dropped, time_squeeze).

```bash
cat /proc/softirqs | grep NET
```
Счетчики NET_RX_SOFTIRQ и NET_TX_SOFTIRQ.

## Примеры

Диагностика потерь пакетов:

```bash
# 1. Потери на NIC (ring buffer overflow)
ethtool -S eth0 | grep rx_missed

# 2. Потери на softirq (не успевает обработать)
cat /proc/net/softnet_stat
# Второй столбец - dropped, третий - time_squeeze

# 3. Потери на сокете
ss -tmpn | grep -A1 ESTAB
# Смотреть Recv-Q
```

Тюнинг для высокой нагрузки:

```bash
# Увеличить ring buffer
ethtool -G eth0 rx 4096 tx 4096

# Увеличить coalescing (throughput важнее latency)
ethtool -C eth0 rx-usecs 100 rx-frames 64

# Увеличить backlog (сколько пакетов может накопиться до обработки)
sysctl -w net.core.netdev_max_backlog=10000

# Увеличить budget softirq (сколько пакетов за один вызов)
sysctl -w net.core.netdev_budget=600
```

Тюнинг для низкой latency:

```bash
# Минимальный coalescing
ethtool -C eth0 rx-usecs 0 rx-frames 1

# Или адаптивный
ethtool -C eth0 adaptive-rx on
```
