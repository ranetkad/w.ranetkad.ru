##### Контроллер и вычислительный узел Nova 

Название: OpenStack Compute
Назначение: управление виртуальными машинами и сетью
Пакет: openstack-nova
Имена сервисов: openstack-nova-api, openstack-nova-consoleauth,openstack-nova-scheduler, openstack-nova-conductor.service, openstack-nova-novncproxy.service, openstack-nova-compute.service
Порты: 8773, 8774, 8775, 8778, 5900-5999 (VNC)
Конфиги: /etc/nova/nova.conf
Логи: /var/log/nova/nova-*