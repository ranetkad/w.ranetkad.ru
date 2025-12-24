# OpenStack - сервисы

## Коротко

Основные сервисы OpenStack: порты, конфиги, логи. Быстрая справка.

## Keystone (Identity)

Аутентификация и авторизация.

- Порты: 5000, 35357
- Конфиг: `/etc/keystone/keystone.conf`
- Логи: `/var/log/keystone.log`
- Пакет: `openstack-keystone`

## Nova (Compute)

Управление виртуальными машинами.

- Порты: 8774 (API), 8775, 5900-5999 (VNC)
- Конфиг: `/etc/nova/nova.conf`
- Логи: `/var/log/nova/`
- Пакет: `openstack-nova`
- Сервисы: `openstack-nova-api`, `openstack-nova-scheduler`, `openstack-nova-conductor`, `openstack-nova-compute`

## Glance (Image)

Хранение образов ВМ.

- Порт: 9292
- Конфиги: `/etc/glance/glance-api.conf`, `/etc/glance/glance-registry.conf`
- Логи: `/var/log/glance/`
- Пакет: `openstack-glance`
- Сервисы: `openstack-glance-api`, `openstack-glance-registry`

## Neutron (Networking)

Сеть для ВМ.

- Порт: 9696
- Конфиги: `/etc/neutron/`
- Логи: `/var/log/neutron/`
- Пакет: `openstack-neutron`

## Cinder (Block Storage)

Блочные тома для ВМ.

- Порты: 8776 (API), 3260 (iSCSI)
- Конфиг: `/etc/cinder/cinder.conf`
- Логи: `/var/log/cinder/`
- Пакет: `openstack-cinder`
- Сервисы: `openstack-cinder-api`, `openstack-cinder-scheduler`, `openstack-cinder-volume`

## Swift (Object Storage)

Объектное хранилище.

- Порты: 8080 (proxy), 6000-6002, 873 (rsync)
- Конфиги: `/etc/swift/`
- Логи: `/var/log/messages`
- Пакеты: `openstack-swift-*`
- Сервисы: `openstack-swift-proxy`, `openstack-swift-object`, `openstack-swift-container`, `openstack-swift-account`
