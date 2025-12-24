# TCP

## Коротко

Transmission Control Protocol - надежный транспортный протокол с гарантией доставки и порядка.
Использует соединения (connection-oriented), подтверждения (ACK), контроль потока (flow control) и контроль перегрузок (congestion control).
Определен в RFC 793 (1981), актуальная версия - RFC 9293.

## Заголовок TCP

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Флаги (6 бит):

- SYN - синхронизация sequence numbers, установка соединения
- ACK - поле Acknowledgment Number валидно
- FIN - отправитель завершает передачу
- RST - сброс соединения
- PSH - передать данные приложению немедленно
- URG - поле Urgent Pointer валидно

## Установка соединения (3-way handshake)

```
Client                    Server
   |                         |
   |-------- SYN ----------->|   seq=x
   |                         |
   |<----- SYN+ACK ----------|   seq=y, ack=x+1
   |                         |
   |-------- ACK ----------->|   seq=x+1, ack=y+1
   |                         |
```

Цель: обмен начальными sequence numbers, предотвращение дубликатов старых соединений.

## Закрытие соединения (4-way handshake)

```
Client                    Server
   |                         |
   |-------- FIN ----------->|   active close
   |                         |
   |<------- ACK ------------|
   |                         |
   |<------- FIN ------------|   passive close
   |                         |
   |-------- ACK ----------->|
   |                         |
   | (TIME_WAIT 2*MSL)       |
```

## Состояния TCP

LISTEN - ожидание входящих соединений (passive open)
SYN_SENT - отправлен SYN, ждем SYN+ACK
SYN_RECEIVED - получен SYN, отправлен SYN+ACK
ESTABLISHED - соединение установлено, передача данных
FIN_WAIT_1 - отправлен FIN, ждем ACK
FIN_WAIT_2 - получен ACK на FIN, ждем FIN от другой стороны
CLOSE_WAIT - получен FIN, ждем close() от приложения
CLOSING - обе стороны отправили FIN одновременно
LAST_ACK - после отправки FIN в ответ на полученный FIN
TIME_WAIT - ждем 2*MSL (обычно 2 минуты) перед полным закрытием

## Контроль потока (Flow Control)

Sliding Window - получатель сообщает в поле Window сколько байт готов принять.
Отправитель не может отправить больше, чем разрешено окном.

Window = 0: отправитель останавливается и шлет probe-пакеты (persist timer).

Window Scaling (RFC 1323): опция при handshake, масштабирует 16-битное поле Window до 1 GB.

## Контроль перегрузок (Congestion Control)

RFC 5681 определяет 4 алгоритма:

**Slow Start**
- cwnd (congestion window) начинается с малого значения
- cwnd увеличивается на 1 MSS за каждый ACK
- Экспоненциальный рост до ssthresh

**Congestion Avoidance**
- Когда cwnd >= ssthresh
- cwnd увеличивается на ~1 MSS за RTT (линейный рост)
- AIMD: Additive Increase, Multiplicative Decrease

**Fast Retransmit**
- 3 duplicate ACK -> немедленная ретрансмиссия без ожидания таймера

**Fast Recovery**
- После fast retransmit: ssthresh = cwnd/2, cwnd = ssthresh + 3*MSS

## Команды

```bash
ss -t
```
Показать TCP-соединения.

```bash
ss -tlnp
```
Показать слушающие TCP-порты с процессами.

```bash
ss -s
```
Сводка по состояниям TCP (estab, timewait, и т.д.).

```bash
netstat -nat | awk '{print $6}' | sort | uniq -c | sort -rn
```
Подсчет соединений по состояниям.

```bash
tcpdump -i eth0 tcp port 80 -n
```
Захват TCP-трафика на порту 80.

```bash
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' -n
```
Захват только SYN-пакетов.

```bash
tcpdump -i any -w capture.pcap host 10.0.0.1
```
Запись в файл для анализа в Wireshark.

## Примеры

Проверка состояния соединений на сервере:

```bash
ss -tan state established
ss -tan state time-wait
ss -tan state close-wait
```

Много CLOSE_WAIT - приложение не закрывает сокеты (баг).
Много TIME_WAIT - много коротких соединений (норма для HTTP).

Просмотр TCP-параметров ядра:

```bash
sysctl net.ipv4.tcp_fin_timeout
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes
```

Изменение параметров:

```bash
# Уменьшить FIN_WAIT_2 timeout (по умолчанию 60)
sysctl -w net.ipv4.tcp_fin_timeout=15

# Keepalive: проверять через 5 минут вместо 2 часов
sysctl -w net.ipv4.tcp_keepalive_time=300
```

Для постоянных изменений - добавить в /etc/sysctl.conf:

```
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 5
```

Применить: `sysctl -p`
