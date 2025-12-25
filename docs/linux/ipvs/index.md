# IPVS

## Коротко

IP Virtual Server - балансировщик нагрузки L4 в ядре Linux. Часть проекта LVS (Linux Virtual Server).

Клиент обращается на VIP, IPVS распределяет соединения на real servers. Работает с TCP и UDP.

Три метода доставки:
- NAT - изменение dst IP, ответ через балансировщик
- DR (Direct Routing) - изменение MAC, ответ напрямую клиенту
- TUN (Tunneling) - IPIP инкапсуляция

Используется в keepalived, kube-proxy (Kubernetes).

## Методы forwarding

### NAT (-m)

Балансировщик переписывает dst IP с VIP на IP real server. Ответ идет обратно через балансировщик.

```
Client -> [VIP] -> LVS (DNAT) -> Real Server
Client <- [VIP] <- LVS (SNAT) <- Real Server
```

- Real servers используют LVS как default gateway
- Поддерживает port mapping (VIP:80 -> RIP:8080)
- Балансировщик обрабатывает весь трафик
- Real servers могут быть в приватной сети

### DR (-g)

Балансировщик меняет только MAC-адрес. IP остается VIP. Real server отвечает напрямую клиенту.

```
Client -> [VIP] -> LVS (MAC rewrite) -> Real Server
Client <------------------------------ Real Server
```

- Real servers должны иметь VIP на loopback (без ARP)
- Не поддерживает port mapping
- Высокая производительность
- Real servers в одном L2 сегменте с LVS

### TUN (-i)

Балансировщик инкапсулирует пакет в IPIP туннель. Real server декапсулирует и отвечает напрямую.

- Real servers могут быть в разных сетях
- Не поддерживает port mapping
- Требует IPIP на real servers

## Schedulers

**rr** - round robin. По очереди.

**wrr** - weighted round robin. Учитывает weight серверов.

**lc** - least connections. На сервер с наименьшим числом соединений.

**wlc** - weighted least connections. lc с учетом weight. По умолчанию.

**sh** - source hash. Один клиент всегда на один сервер.

**dh** - destination hash. Для прокси/кешей.

**sed** - shortest expected delay.

**nq** - never queue. Сначала на idle серверы.

## Команды

```bash
apt install ipvsadm
```
Установка.

```bash
modprobe ip_vs
```
Загрузка модуля ядра.

```bash
ipvsadm -A -t 192.168.1.100:80 -s rr
```
Создать сервис (TCP, round robin).

```bash
ipvsadm -A -u 192.168.1.100:53 -s wlc
```
Создать сервис (UDP).

```bash
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.1:80 -m -w 1
```
Добавить real server (NAT, weight 1).

```bash
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.2:80 -g -w 2
```
Добавить real server (DR, weight 2).

```bash
ipvsadm -e -t 192.168.1.100:80 -r 10.0.0.1:80 -w 0
```
Weight 0 - graceful drain.

```bash
ipvsadm -d -t 192.168.1.100:80 -r 10.0.0.1:80
```
Удалить real server.

```bash
ipvsadm -D -t 192.168.1.100:80
```
Удалить сервис.

```bash
ipvsadm -ln
```
Показать конфигурацию.

```bash
ipvsadm -ln --stats
```
Статистика.

```bash
ipvsadm -lnc
```
Текущие соединения.

```bash
ipvsadm -C
```
Очистить все правила.

```bash
ipvsadm-save > /etc/ipvsadm.rules
ipvsadm-restore < /etc/ipvsadm.rules
```
Сохранить/восстановить.

## Примеры

### HTTP балансировщик (NAT)

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward

ipvsadm -A -t 192.168.1.100:80 -s wlc
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.10:80 -m -w 1
ipvsadm -a -t 192.168.1.100:80 -r 10.0.0.11:80 -m -w 1
```

Real servers: default gateway на IP балансировщика.

### HTTP балансировщик (DR)

На балансировщике:

```bash
ip addr add 192.168.1.100/32 dev eth0

ipvsadm -A -t 192.168.1.100:80 -s wlc
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.10:80 -g -w 1
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.11:80 -g -w 1
```

На каждом real server:

```bash
ip addr add 192.168.1.100/32 dev lo

echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

### Persistence

```bash
ipvsadm -A -t 192.168.1.100:80 -s rr -p 300
```

Клиент привязан к серверу на 300 секунд.

## Проверка

```bash
lsmod | grep ip_vs
```
Модуль загружен.

```bash
ipvsadm -ln
```
Текущие правила.

```bash
ipvsadm -lnc
```
Активные соединения.
