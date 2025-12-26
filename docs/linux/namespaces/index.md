# Изоляция процессов в Linux

## Коротко

Механизмы ядра Linux для изоляции и ограничения процессов. На них построены контейнеры (Docker, LXC, Podman).

| Механизм | Что делает |
|----------|------------|
| **Namespaces** | Изоляция ресурсов (сеть, PID, FS, users) |
| **Cgroups** | Лимиты CPU, памяти, I/O |
| **Capabilities** | Гранулярные привилегии вместо полного root |
| **Seccomp** | Фильтрация системных вызовов |
| **AppArmor/SELinux** | Mandatory Access Control - доступ к файлам/сети |
| **pivot_root** | Смена корневой файловой системы |

Контейнер = namespaces + cgroups + capabilities + seccomp + MAC + pivot_root.

## Namespaces

8 типов namespaces (Linux 6.1+):

| Namespace | Clone Flag | Что изолирует |
|-----------|------------|---------------|
| **pid** | CLONE_NEWPID | Process IDs - процессы не видят PID из других ns |
| **net** | CLONE_NEWNET | Сетевой стек: интерфейсы, IP, routes, iptables, сокеты |
| **mnt** | CLONE_NEWNS | Mount points - своя файловая система |
| **user** | CLONE_NEWUSER | UID/GID mapping - root внутри != root снаружи |
| **uts** | CLONE_NEWUTS | Hostname и domain name |
| **ipc** | CLONE_NEWIPC | System V IPC, POSIX message queues, shared memory |
| **cgroup** | CLONE_NEWCGROUP | Изолированный view cgroup hierarchy |
| **time** | CLONE_NEWTIME | Boot и monotonic clocks (с kernel 5.6) |

## Команды

Посмотреть namespaces процесса:

```bash
ls -la /proc/$$/ns/
```

```
lrwxrwxrwx 1 root root 0 Dec 25 12:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Dec 25 12:00 uts -> 'uts:[4026531838]'
```

Список всех namespaces в системе:

```bash
lsns
```

Создать процесс в новом namespace:

```bash
# новый network + uts namespace
unshare --net --uts /bin/bash

# проверить - пустой сетевой стек
ip link
# только lo
```

Войти в namespace другого процесса:

```bash
# войти в net namespace процесса 1234
nsenter -t 1234 -n ip addr

# войти во все namespaces контейнера
nsenter -t $(docker inspect -f '{{.State.Pid}}' container_name) -a
```

## Network Namespace - пример

Создание изолированной сети между двумя netns:

```bash
# создать два netns
ip netns add ns1
ip netns add ns2

# создать veth pair
ip link add veth1 type veth peer name veth2

# переместить интерфейсы в namespaces
ip link set veth1 netns ns1
ip link set veth2 netns ns2

# настроить IP
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
ip netns exec ns1 ip link set veth1 up
ip netns exec ns1 ip link set lo up

ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
ip netns exec ns2 ip link set veth2 up
ip netns exec ns2 ip link set lo up

# проверить связность
ip netns exec ns1 ping 10.0.0.2
```

Удаление:

```bash
ip netns del ns1
ip netns del ns2
```

## Cgroups

Cgroups (control groups) - лимитирование и учет ресурсов для группы процессов.

**cgroups v1** - отдельная иерархия на каждый контроллер (устаревший)
**cgroups v2** - единая иерархия, делегирование non-root пользователям (современный)

Основные контроллеры:

| Контроллер | Что лимитирует |
|------------|----------------|
| **cpu** | CPU time, веса, квоты |
| **memory** | RAM, swap, OOM killer |
| **io** | Disk I/O bandwidth, IOPS |
| **pids** | Максимальное число процессов |
| **cpuset** | Привязка к конкретным CPU/NUMA |

## Команды cgroups v2

Проверить что используется v2:

```bash
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 ...
```

Посмотреть доступные контроллеры:

```bash
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory pids
```

Создать cgroup и добавить процесс:

```bash
# создать группу
mkdir /sys/fs/cgroup/mygroup

# включить контроллеры для дочерних групп
echo "+cpu +memory +io" > /sys/fs/cgroup/cgroup.subtree_control

# лимит памяти 100MB
echo 100M > /sys/fs/cgroup/mygroup/memory.max

# лимит CPU 50%
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# добавить процесс
echo $$ > /sys/fs/cgroup/mygroup/cgroup.procs
```

Посмотреть текущее использование:

```bash
cat /sys/fs/cgroup/mygroup/memory.current
cat /sys/fs/cgroup/mygroup/cpu.stat
```

## systemd и cgroups

systemd использует cgroups для управления сервисами.

Посмотреть дерево cgroups:

```bash
systemd-cgls
```

Лимиты для сервиса:

```ini
# /etc/systemd/system/myservice.service.d/limits.conf
[Service]
MemoryMax=512M
CPUQuota=50%
IOWeight=100
```

Применить:

```bash
systemctl daemon-reload
systemctl restart myservice
```

## Capabilities

Linux capabilities - разбиение привилегий root на отдельные единицы. Вместо "всё или ничего" процесс получает только нужные привилегии.

Основные capabilities:

| Capability | Что разрешает |
|------------|---------------|
| **CAP_NET_ADMIN** | Настройка сети: IP, routes, iptables |
| **CAP_NET_BIND_SERVICE** | Bind на порты < 1024 |
| **CAP_NET_RAW** | RAW/PACKET сокеты |
| **CAP_SYS_ADMIN** | "Новый root" - mount, namespaces, много всего |
| **CAP_SYS_PTRACE** | ptrace любого процесса |
| **CAP_CHOWN** | Смена владельца файлов |
| **CAP_DAC_OVERRIDE** | Игнорировать права доступа к файлам |
| **CAP_SETUID/SETGID** | Смена UID/GID |

Docker по умолчанию даёт контейнеру ~14 capabilities и дропает опасные (SYS_ADMIN, NET_ADMIN).

Посмотреть capabilities процесса:

```bash
# текущий процесс
capsh --print

# конкретный процесс
getpcaps $$

# файла
getcap /usr/bin/ping
```

Управление в Docker:

```bash
# убрать все и добавить только нужные
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# добавить capability
docker run --cap-add=NET_ADMIN alpine ip link
```

## Seccomp

Seccomp (Secure Computing) - фильтрация системных вызовов на уровне ядра. Процесс может вызывать только разрешённые syscalls.

Режимы:
- **strict** - только read/write/exit/sigreturn (почти бесполезен)
- **filter (BPF)** - гибкие правила через BPF программу

Docker по умолчанию блокирует ~44 syscalls: mount, reboot, setns, swapon и др.

Посмотреть seccomp статус процесса:

```bash
grep Seccomp /proc/$$/status
# Seccomp:        0  - отключен
# Seccomp:        2  - filter mode
```

Docker с кастомным профилем:

```bash
# использовать свой профиль
docker run --security-opt seccomp=/path/to/profile.json alpine

# отключить seccomp (небезопасно)
docker run --security-opt seccomp=unconfined alpine
```

Пример профиля (разрешить всё кроме mount):

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["mount", "umount2"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

## AppArmor

AppArmor - Mandatory Access Control (MAC) для Debian/Ubuntu. Профили ограничивают доступ процесса к файлам, сети, capabilities.

Режимы профилей:
- **enforce** - блокирует нарушения
- **complain** - только логирует

Статус:

```bash
aa-status
```

Docker автоматически применяет профиль `docker-default` к контейнерам.

```bash
# свой профиль
docker run --security-opt apparmor=my-profile alpine

# отключить (небезопасно)
docker run --security-opt apparmor=unconfined alpine
```

Пример профиля `/etc/apparmor.d/my-container`:

```
#include <tunables/global>

profile my-container flags=(attach_disconnected) {
  #include <abstractions/base>

  # разрешить чтение
  /etc/passwd r,
  /app/** r,

  # запретить запись в /etc
  deny /etc/** w,

  # разрешить сеть
  network inet tcp,
}
```

Загрузить профиль:

```bash
apparmor_parser -r /etc/apparmor.d/my-container
```

## SELinux

SELinux - MAC для RHEL/CentOS/Fedora. Использует метки (labels) на процессах и ресурсах.

Режимы:
- **enforcing** - блокирует нарушения
- **permissive** - только логирует
- **disabled** - отключен

```bash
# текущий режим
getenforce

# временно переключить
setenforce 0  # permissive
setenforce 1  # enforcing
```

Контекст безопасности (метка):

```bash
# файла
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0

# процесса
ps -Z
```

Docker и SELinux:

```bash
# включить разделение контейнеров
docker run --security-opt label=type:container_t alpine

# отключить SELinux для контейнера
docker run --security-opt label=disable alpine

# разрешить доступ к volume
docker run -v /data:/data:Z alpine  # :Z перемаркирует файлы
```

## pivot_root

pivot_root - смена корневой файловой системы. Контейнеры используют вместо chroot (более безопасен).

Отличие от chroot:
- chroot меняет только `/` для процесса, старый root доступен
- pivot_root меняет root для всего mount namespace, старый root можно отмонтировать

```bash
# подготовка нового root
mkdir /newroot
mount --bind /path/to/rootfs /newroot

# pivot
cd /newroot
mkdir old_root
pivot_root . old_root

# отмонтировать старый root
umount -l /old_root
rmdir /old_root
```

В контейнерах это делает runtime (runc, crun) автоматически.

## Проверка

Посмотреть все механизмы изоляции процесса:

```bash
# namespaces
ls -la /proc/$$/ns/

# cgroup
cat /proc/$$/cgroup

# capabilities
getpcaps $$

# seccomp
grep Seccomp /proc/$$/status

# SELinux контекст (если включен)
ps -Z -p $$

# AppArmor профиль (если включен)
cat /proc/$$/attr/current
```

Для контейнера Docker:

```bash
# PID контейнера на хосте
PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# проверить всё
ls -la /proc/$PID/ns/
cat /proc/$PID/cgroup
getpcaps $PID
grep Seccomp /proc/$PID/status
```

## Ссылки

- [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [cgroups v2 - kernel.org](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html)
