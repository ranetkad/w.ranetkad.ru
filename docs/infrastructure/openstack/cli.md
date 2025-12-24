# CLI

## Коротко

Unified CLI для OpenStack. Перед использованием нужно source keystonerc файла проекта.

## Команды

Аутентификация:

```bash
source keystonerc_admin
```

Загрузить credentials проекта в переменные окружения.

Серверы (ВМ):

```bash
openstack server list
```

Список ВМ.

```bash
openstack server show my-server
```

Детали ВМ.

```bash
openstack server create --image ubuntu --flavor m1.small --network private my-server
```

Создать ВМ.

```bash
openstack server delete my-server
```

Удалить ВМ.

```bash
openstack server stop my-server
openstack server start my-server
```

Остановить/запустить ВМ.

Образы:

```bash
openstack image list
```

Список образов.

Flavors:

```bash
openstack flavor list
```

Список flavor (типоразмеры ВМ).

```bash
openstack flavor create --ram 512 --disk 10 --vcpus 1 m1.tiny
```

Создать flavor.

Сеть:

```bash
openstack network list
```

Список сетей.

```bash
openstack network create my-network
```

Создать сеть.

```bash
openstack subnet create --network my-network --subnet-range 192.168.1.0/24 my-subnet
```

Создать подсеть.

Тома:

```bash
openstack volume list
```

Список томов.

```bash
openstack volume create --size 10 my-volume
```

Создать том 10 GB.

```bash
openstack server add volume my-server my-volume
```

Подключить том к ВМ.

Security Groups:

```bash
openstack security group list
```

Список групп.

```bash
openstack security group rule create default --protocol tcp --dst-port 22
```

Разрешить SSH.

```bash
openstack security group rule create default --protocol icmp
```

Разрешить ping.

## Примеры

Вывод в JSON:

```bash
openstack server list -f json
```

Фильтрация:

```bash
openstack server list | grep running
```
