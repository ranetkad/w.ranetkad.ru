# LVM

## TL;DR

- PV (Physical Volume) — физический диск/раздел
- VG (Volume Group) — объединение PV в пул
- LV (Logical Volume) — виртуальный раздел из VG
- Можно менять размер LV на лету, делать снапшоты

## Быстрый старт

Добавить новый диск и создать том:

```bash
pvcreate /dev/sdb
vgcreate data_vg /dev/sdb
lvcreate -L 50G -n app_lv data_vg
mkfs.ext4 /dev/data_vg/app_lv
mount /dev/data_vg/app_lv /mnt/app
```

## Команды

### Создание

```bash
pvcreate /dev/sdX
```
Инициализировать диск как PV.

```bash
vgcreate my_vg /dev/sdb /dev/sdc
```
Создать VG из нескольких PV.

```bash
lvcreate -L 10G -n my_lv my_vg
```
Создать LV на 10 ГБ.

### Расширение

```bash
lvextend -L +5G /dev/my_vg/my_lv
resize2fs /dev/my_vg/my_lv
```
Добавить 5 ГБ к LV и расширить ФС (ext4).

### Сокращение

```bash
umount /dev/my_vg/my_lv
e2fsck -f /dev/my_vg/my_lv
resize2fs /dev/my_vg/my_lv 15G
lvreduce -L 15G /dev/my_vg/my_lv
mount /dev/my_vg/my_lv /mnt
```
Сначала уменьшить ФС, потом LV. Иначе потеря данных.

### Снапшоты

```bash
lvcreate -L 5G -s -n my_lv_snap /dev/my_vg/my_lv
```
Снапшот для бэкапа. Размер — под изменения на время жизни снапшота.

### Просмотр

```bash
pvs / pvdisplay
vgs / vgdisplay
lvs / lvdisplay
```
Короткий / подробный вывод.

### Удаление

```bash
lvremove /dev/my_vg/my_lv
```

## Подводные камни

- `lvreduce` без предварительного `resize2fs` = потеря данных
- Снапшот переполнится если изменений больше чем его размер — станет невалидным
- Для XFS используй `xfs_growfs` вместо `resize2fs`

