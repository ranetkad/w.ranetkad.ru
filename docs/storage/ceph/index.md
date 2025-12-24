# Ceph

## TL;DR

Программно-определяемое распределённое хранилище. Даёт block (RBD), object (S3/Swift) и file (CephFS) storage из одного кластера.

- OSD (Object Storage Daemon) — хранит данные на дисках
- MON (Monitor) — следит за состоянием кластера, хранит карту
- MGR (Manager) — метрики, дашборд, балансировка
- MDS (Metadata Server) — только для CephFS

## Пакеты и пути

- Пакеты: `ceph`, `ceph-radosgw`
- Сервис: `ceph.service`
- Конфиг: `/etc/ceph/ceph.conf`
- Логи: `/var/log/ceph/*`
- Keyring: `/etc/ceph/ceph.client.admin.keyring`

## Команды

### Статус кластера

```bash
ceph status
ceph health detail
```
Общее состояние. HEALTH_OK / HEALTH_WARN / HEALTH_ERR.

```bash
ceph osd tree
```
Дерево OSD — какие диски на каких хостах, статус up/down.

```bash
ceph df
```
Использование места в кластере и по пулам.

### Пулы

```bash
ceph osd pool ls
```
Список пулов.

```bash
ceph osd pool create mypool 128
```
Создать пул с 128 PG (placement groups).

```bash
ceph osd pool set mypool size 3
```
Установить репликацию x3.

### RBD (block storage)

```bash
rbd create mypool/myimage --size 10G
```
Создать блочный образ 10 ГБ.

```bash
rbd map mypool/myimage
```
Подключить как /dev/rbdX.

```bash
rbd ls mypool
rbd info mypool/myimage
```
Список образов / информация.

### Сервисы

```bash
ceph orch ls
ceph orch ps
```
Список сервисов и процессов (cephadm).

```bash
ceph orch daemon restart osd.0
```
Перезапустить конкретный демон.

## Проверка

```bash
ceph health
```
Быстрая проверка — HEALTH_OK значит всё ок.

```bash
ceph osd stat
```
Сколько OSD всего / up / in.

```bash
ceph pg stat
```
Состояние placement groups.

## Подводные камни

- Минимум 3 MON для кворума
- PG count влияет на производительность — слишком мало или много = проблемы
- При падении OSD данные ребалансируются — нагрузка на сеть
- `ceph osd pool delete` требует подтверждения дважды (защита от случайного удаления)
