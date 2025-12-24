# FreeBSD vs Linux

## Коротко

Netflix гонит 400Gb/s с одного сервера на FreeBSD. Почему не Linux? sendfile + KTLS в ядре дают zero-copy до самого NIC. В Linux данные копируются для TLS шифрования в nginx.

## Буферы: mbuf vs sk_buff

Linux пошел по пути "один пакет = один линейный буфер". Проще, меньше pointer chasing, многие операции быстрее. Минус - иногда резервируется лишняя память.

FreeBSD использует цепочки mbuf (256 байт) + кластеры (2K/4K/9K). Гибче под разные размеры, но нужен m_pullup() чтобы собрать заголовки в непрерывный блок.

На практике разница не критична для большинства задач.

## kqueue vs epoll

epoll раздражает тем, что на каждое изменение fd нужен отдельный epoll_ctl(). 100 сокетов поменялось - 100 syscalls.

kqueue делает всё через один kevent() - и регистрация, и получение событий. Меньше syscalls = меньше overhead.

Еще kqueue нормально работает с disk I/O через EVFILT_AIO. В Linux epoll на файлах бесполезен - файл "всегда ready", а реально блокируется на чтении с диска.

Источник: [kqueue vs epoll](https://thamizhelango.medium.com/freebsd-kqueue-vs-linux-epoll-why-kqueue-often-outperforms-and-its-impact-on-network-1ae39afb6e16)

## sendfile - вот где разница

FreeBSD sendfile делает share страниц между buffer cache и mbuf. Данные с диска попадают в cache, оттуда страницы расшариваются до самого NIC. Копирования нет.

Плюс он асинхронный - возвращает success до завершения disk I/O. Данные дописываются в сокет позже, порядок сохраняется.

Linux sendfile тоже zero-copy, но когда нужен TLS - nginx читает данные в userspace, шифрует, отдает обратно в ядро. Вот тут копирование.

## Почему Netflix на FreeBSD

Изначально выбрали потому что TCP/IP стек показывал меньше latency. Потом вложили кучу работы в оптимизации - уходить уже глупо.

Главная фишка - KTLS. TLS шифрование перенесли из nginx в ядро. sendfile pipeline сохраняется:

```
Без KTLS:  Disk -> Cache -> nginx (encrypt) -> Kernel -> NIC
С KTLS:   Disk -> Cache -> Kernel (encrypt) -> NIC
```

В 2022 добавили NIC kTLS offload - шифрование на сетевухе. CPU вообще не трогает данные.

Железо для 200Gb/s сервера: AMD EPYC 7502P, 256GB RAM, 4x100GbE Mellanox, 18 NVMe дисков. 200Gb/s это 25GB/s, нужно ~100GB/s memory bandwidth.

NUMA siloing тоже их работа - привязка сетевых соединений к NUMA node, локальная аллокация памяти под файлы и crypto буферы. Intel Xeon поднялся с 105Gb/s до 191Gb/s.

Источник: [Netflix Case Study](https://freebsdfoundation.org/end-user-stories/netflix-case-study/)

## Когда Linux лучше

Bulk transfer больших объемов (Science DMZ сценарии). L3 forwarding на много CPU - Linux лучше параллелится. Hardware support шире.

io_uring в свежих ядрах может быть быстрее kqueue на некоторых паттернах - полностью асинхронный I/O.

## Когда FreeBSD лучше

Стриминг статики с TLS. Web-серверы под высокой нагрузкой. Network appliances.

Используют Netflix, WhatsApp, Juniper.

Источник: [Klara Systems](https://klarasystems.com/articles/freebsd-vs-linux-networking/)
