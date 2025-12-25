# Keepalived

## Коротко

Демон для отказоустойчивости на Linux. Реализует VRRP (Virtual Router Redundancy Protocol) - автоматический failover виртуального IP между серверами. Также может выполнять балансировку нагрузки через IPVS (LVS).

Основные сценарии:
- HA для nginx, haproxy, database VIP
- Активный/резервный кластер из двух нод
- Балансировка с health checks реальных серверов

## Как работает VRRP

VRRP использует IP protocol 112 (не TCP/UDP).

Master отправляет advertisements каждые `advert_int` секунд (по умолчанию 1 сек) на multicast 224.0.0.18 (IPv4) или ff02::12 (IPv6).

Backup слушает эти пакеты. Если advertisements перестают приходить - backup становится master и забирает VIP.

Выборы master:
- Побеждает нода с большим `priority` (1-255)
- При равном priority - нода с большим IP

Preemption: если упавший master восстановился и его priority выше - он заберет VIP обратно. Отключается через `nopreempt`.

## Конфигурация

Файл: `/etc/keepalived/keepalived.conf`

### Минимальный MASTER/BACKUP

**Нода 1 (master):**

```
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 101
    priority 255
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret12
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

**Нода 2 (backup):**

```
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 101
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret12
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

Критичные параметры:
- `virtual_router_id` - должен совпадать на обеих нодах, уникален в сегменте (1-255)
- `auth_pass` - максимум 8 символов
- `priority` - у master выше чем у backup

### Failover без preemption

Оба сервера стартуют как BACKUP. Кто первый поднялся - тот master. При восстановлении упавшего - он остается backup.

```
vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 101
    priority 200
    advert_int 1

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

### Health check через track_script

Отслеживание сервиса. При падении сервиса - снижается priority и VIP уходит на backup.

```
vrrp_script chk_nginx {
    script "/usr/bin/pgrep nginx"
    interval 2
    weight -50
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 101
    priority 100

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

Параметры vrrp_script:
- `interval` - частота проверки в секундах (по умолчанию 1)
- `weight` - изменение priority при fail (отрицательное значение)
- `fall` - сколько раз должен упасть для перехода в FAULT
- `rise` - сколько раз должен отработать для выхода из FAULT

### Track interface

Падение интерфейса -> переход в FAULT -> VIP уходит на backup.

```
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 101
    priority 100

    track_interface {
        eth1
        eth2 weight -50
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

### Track process

```
vrrp_track_process track_haproxy {
    process haproxy
    delay 1
}

vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 101
    priority 200

    track_process {
        track_haproxy
    }

    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

## Команды

```bash
apt install keepalived
```
Установка (Debian/Ubuntu).

```bash
systemctl enable keepalived
systemctl start keepalived
```
Запуск сервиса.

```bash
systemctl status keepalived
journalctl -u keepalived -f
```
Статус и логи.

```bash
ip addr show eth0
```
Проверить наличие VIP на интерфейсе.

```bash
tcpdump -i eth0 -n proto 112
```
Захват VRRP advertisements.

## Firewall

VRRP использует IP protocol 112. Без правил firewall advertisements не пройдут.

```bash
# iptables
iptables -A INPUT -p 112 -j ACCEPT

# firewalld
firewall-cmd --add-protocol=vrrp --permanent
firewall-cmd --reload

# nftables
nft add rule inet filter input ip protocol 112 accept
```

## Примеры

### HA для nginx

Две ноды. VIP 10.0.0.100. При падении nginx на master - VIP переезжает на backup.

**Обе ноды:**

```
global_defs {
    router_id nginx_ha
}

vrrp_script chk_nginx {
    script "/usr/bin/pgrep nginx"
    interval 2
    weight -50
    fall 2
    rise 2
}

vrrp_instance VI_NGINX {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        10.0.0.100/24
    }
}
```

Разница между нодами - можно задать разный `priority` (100 vs 90).

### LVS балансировка

Keepalived как балансировщик на основе IPVS.

```
virtual_server 192.168.1.100 80 {
    delay_loop 6
    lvs_sched rr
    lvs_method NAT
    protocol TCP

    real_server 192.168.1.10 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
        }
    }

    real_server 192.168.1.11 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
        }
    }
}
```

Schedulers (`lvs_sched`):
- `rr` - round robin
- `wrr` - weighted round robin
- `lc` - least connections
- `wlc` - weighted least connections
- `sh` - source hash

Methods (`lvs_method`):
- `NAT` - destination NAT
- `DR` - direct routing (real servers отвечают напрямую)
- `TUN` - IP tunneling

## Проверка

1. Запустить keepalived на обеих нодах
2. `ip addr show` - VIP должен быть на master
3. Остановить keepalived на master: `systemctl stop keepalived`
4. `ip addr show` на backup - VIP должен появиться
5. `tcpdump -i eth0 proto 112` - должны идти VRRP пакеты
