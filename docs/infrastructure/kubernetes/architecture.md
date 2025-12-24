# Архитектура

## Коротко

Кластер = Control Plane (управление) + Worker Nodes (запуск подов).
Состояние хранится в etcd. Все взаимодействие через API Server.

## Control Plane

**kube-apiserver** - REST API кластера, единая точка входа для всех операций.

**etcd** - распределенное key-value хранилище состояния кластера.

**kube-scheduler** - распределяет поды по узлам на основе ресурсов и политик.

**kube-controller-manager** - запускает контроллеры (Node, Deployment, ReplicaSet и др.).

**cloud-controller-manager** - интеграция с облачными провайдерами (опционально).

## Worker Node

**kubelet** - агент на узле, запускает поды, отчитывается в API Server.

**kube-proxy** - сетевой прокси, маршрутизация трафика к сервисам.

**Container Runtime** - запуск контейнеров (containerd, CRI-O).

## Дополнительные компоненты

**CoreDNS** - DNS для разрешения имен сервисов внутри кластера.

**CNI-плагин** - сетевая подсистема (Calico, Flannel, Cilium).

## etcd

Бэкап:

```bash
etcdctl snapshot save backup.db
```

Восстановление:

```bash
etcdctl snapshot restore backup.db
```
