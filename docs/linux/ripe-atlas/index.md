# RIPE Atlas Software Probe

## Коротко

Программная probe для сети измерений RIPE NCC. Выполняет ping, traceroute, DNS, SSL/TLS, NTP, HTTP измерения по заданиям из облака.

## Установка

### Debian/Ubuntu

```bash
ARCH=$(dpkg --print-architecture)
CODENAME=$(. /etc/os-release && echo "$VERSION_CODENAME")

wget https://ftp.ripe.net/ripe/atlas/software-probe/debian/dists/"$CODENAME"/main/binary-"$ARCH"/ripe-atlas-repo_1.5-5_all.deb
sudo dpkg -i ripe-atlas-repo_1.5-5_all.deb
sudo apt update && sudo apt install ripe-atlas-probe -y
```

### RHEL/CentOS

```bash
EL_VER=$(. /etc/os-release && echo $PLATFORM_ID | cut -d':' -f2)

curl -fO https://ftp.ripe.net/ripe/atlas/software-probe/"$EL_VER"/noarch/ripe-atlas-repo-1.5-5."$EL_VER".noarch.rpm
sudo rpm -Uvh ripe-atlas-repo-1.5-5."$EL_VER".noarch.rpm
sudo dnf install ripe-atlas-probe -y
```

## Запуск

```bash
sudo systemctl enable --now ripe-atlas
```

## Регистрация

```bash
cat /etc/ripe-atlas/probe_key.pub
```

Скопировать ключ -> https://atlas.ripe.net/apply/swprobe/ -> вставить -> отправить.

## Проверка

```bash
sudo journalctl -u ripe-atlas | grep SESSION_ID
```

Есть SESSION_ID - probe подключился.

## Ссылки

- https://github.com/RIPE-NCC/ripe-atlas-software-probe
- https://atlas.ripe.net/apply/swprobe/
