# Segment Routing

## Коротко

Segment Routing (SR) - архитектура маршрутизации от источника. Головной узел выбирает путь и кодирует его как упорядоченный список сегментов (SID). Каждый транзитный узел выполняет инструкцию из верхнего сегмента и удаляет его.

Два data plane:

- SR-MPLS - использует MPLS метки (20 бит), совместим с существующим оборудованием
- SRv6 - использует IPv6 заголовки (128 бит SID)

Зачем:

- Убирает LDP и RSVP-TE из control plane
- IGP (OSPF/IS-IS) сам распространяет SID
- Нет per-flow состояния на транзитных узлах
- Fast reroute < 50ms

## Типы сегментов

**Prefix SID** (глобальный):

- Привязан к IP-префиксу
- Кодируется как индекс (смещение от SRGB)
- IGP строит кратчайший путь до префикса
- Поддерживает ECMP

**Node SID** (частный случай Prefix SID):

- Привязан к loopback-адресу узла
- Флаг N-Flag = 1

**Adjacency SID** (локальный):

- Привязан к конкретному линку
- Абсолютное значение метки (не индекс)
- Выделяется динамически из SRLB
- Нет ECMP - форсирует трафик через конкретный интерфейс

**Anycast SID**:

- Один и тот же SID анонсируется несколькими узлами
- Трафик идет к ближайшему узлу по IGP

**Binding SID**:

- Локальный сегмент, представляет SR-TE policy
- Абстрагирует список сегментов за одной меткой

**BGP Prefix SID**:

- Глобальный сегмент для BGP-префикса
- Используется в inter-domain сценариях (Unified MPLS)

**BGP Peering SID**:

- Локальный сегмент для BGP-соседа
- Форсирует трафик через конкретного BGP peer

## Label Blocks

**SRGB** (Segment Routing Global Block):

- Диапазон меток для глобальных сегментов (Prefix/Node SID)
- По умолчанию: 16000-23999
- Prefix SID = SRGB base + index

**SRLB** (Segment Routing Local Block):

- Диапазон меток для локальных сегментов (Adjacency SID)
- Выделяется отдельно от SRGB

## SR-TE (Traffic Engineering)

Segment Routing Traffic Engineering - управление путем через сеть.

SR-TE Policy идентифицируется тройкой: (head-end, color, end-point).

- head-end: узел, где создается policy
- color: числовой идентификатор (1-4294967295)
- end-point: конечная точка

SID-list вставляется в пакет на head-end. Транзитные узлы выполняют инструкции.

**Prefix SID в TE**:

- ECMP, устойчивость к изменениям топологии
- Может удлинить путь

**Adjacency SID в TE**:

- Детерминированный путь
- Нет ECMP, уязвим к отказам

## On-Demand Next Hop (ODN)

Автоматическое создание SR-TE policy при появлении BGP-маршрута с color extended community. Head-end применяет шаблон для соответствующего color.

Варианты вычисления пути:

- Локально (head-end использует свою TE topology database)
- Централизованно (SR-PCE)

## Flexible Algorithm (Flex-Algo)

IGP вычисляет дополнительные пути на основе ограничений. Простой TE без контроллера.

Flex-Algo ID: 128-255 (до 128 алгоритмов).

Параметры:

- Calculation type: SPF или Strict SPF
- Metric type: IGP / delay / TE metric
- Constraints: affinity (include/exclude links), min bandwidth, max delay

Все узлы должны согласовать определение Flex-Algo (FAD). Один узел анонсирует FAD с приоритетом.

Prefix SID для Flex-Algo отличается от обычного - трафик идет по пути, вычисленному с ограничениями.

Применение:

- Low-latency routing (metric type = delay)
- Исключение определенных линков (affinity)
- Разделение трафика по типам сервисов

## TI-LFA (Fast Reroute)

Topology Independent Loop-Free Alternate - защита от отказов < 50ms.

Преимущества над классическим LFA/RLFA:

- 100% покрытие в любой топологии (не зависит от геометрии сети)
- Не требует LDP для RLFA туннелей
- Все SID уже в LSDB от IGP

Как работает:

- PLR (Point of Local Repair) заранее вычисляет backup path
- При отказе линка/узла - мгновенное переключение на backup
- Backup path использует SID-list для обхода отказавшего элемента

Защита:

- Link protection
- Node protection
- SRLG protection (Shared Risk Link Group)

## Microloop Avoidance

Предотвращение кратковременных петель при конвергенции сети.

Microloop возникает из-за неодновременной конвергенции разных узлов после изменения топологии.

Как работает:

- После события (link down/up, metric change) узел вычисляет post-convergence path
- Если возможен microloop - устанавливает SID-list для обхода
- После таймера (когда все узлы сконвергировались) - переключается на обычный путь

Работает локально - не требует поддержки на всех узлах.

## Path Computation Element (PCE)

Централизованный контроллер для вычисления путей. Узнает топологию через IGP или BGP-LS.

**PCEP** (Path Computation Element Protocol):

- TCP-based протокол между PCC и PCE
- PCC (Path Computation Client) - маршрутизатор, запрашивающий путь
- PCE вычисляет и инициирует SR-TE policy

Сообщения PCEP:

- PCInitiate - PCE инициирует policy на PCC
- PCRpt - PCC отчитывается о состоянии
- PCUpd - PCE обновляет policy
- PCErr - ошибка

SR-PCE поддерживает stateful режим: PCE хранит состояние всех LSP и может их модифицировать.

## Performance Measurement (PM)

Измерение характеристик сети: delay, packet loss, jitter.

Протоколы:

- TWAMP-Light (RFC 5357) - IP/UDP encap, по умолчанию
- PM-MPLS (RFC 6374) - MPLS encap

Режимы:

- Two-way delay (по умолчанию)
- One-way delay (требует синхронизации PTP)

Применение:

- Мониторинг SLA
- Вход для Flex-Algo (metric type = delay)
- Troubleshooting сети

Заменяет отдельные протоколы мониторинга (BFD для liveness + отдельный PM).

## SRv6

Segment Routing поверх IPv6 data plane. SID = 128 бит (IPv6 адрес).

Структура SID:

- Locator (префикс, маршрутизируемый по IGP)
- Function (инструкция: End, End.X, End.DT4, и т.д.)
- Arguments (опционально)

Преимущества:

- Нативная поддержка в IPv6 сетях
- Network Programming - богатый набор инструкций
- Не требует MPLS

**Micro-SID (uSID)**:

- Компактное кодирование: до 6 uSID в одном 128-бит carrier
- uSID = 16 бит (вместо 128)
- Путь из 5 хопов: 16 байт вместо 80 байт
- Если > 6 uSID - используется SRH (Segment Routing Header)

uSID Block - IPv6 префикс для блока micro-SID.
uN - базовая micro-инструкция: shortest path + активация следующего uSID.

## VPN Services поверх SR

**L3VPN (VPNv4/VPNv6)**:

Конфигурация не отличается от классического MPLS:

- VRF на PE
- MP-BGP для VPNv4 между PE (напрямую или через RR)
- Транспортные метки - SR вместо LDP

**L2VPN**:

- VPWS (point-to-point pseudowire)
- VPLS (multipoint, flooding/learning)
- EVPN (control plane learning через BGP)

L2VPN сервисы используют SR-TE policy через preferred-path.

EVPN над SR-MPLS/SRv6 - современный подход для DC и SP сетей.

## Inter-Domain SR

Сценарии для multi-area/multi-AS:

**Option A** - back-to-back VRF на ASBR

**Option B** - обмен labeled VPN routes между ASBR

**Option C** - обмен loopback через BGP-LU + VPNv4 через RR
- BGP Prefix SID обеспечивает сквозной LSP
- Каждый домен имеет свой SRGB

Unified MPLS / Seamless MPLS - end-to-end SR через несколько IGP доменов.

## Миграция с LDP

SR совместим с LDP. Варианты миграции:

**SR Prefer**:

- Если есть и SR label и LDP label для префикса - используется SR
- Позволяет постепенную миграцию

**Mapping Server**:

- Анонсирует SID для префиксов от LDP-only узлов
- Позволяет SR узлам достигать LDP узлов

## Стандарты

- RFC 8402 - Segment Routing Architecture
- RFC 8660 - Segment Routing with MPLS Data Plane
- RFC 8667 - IS-IS Extensions for Segment Routing
- RFC 8665 - OSPF Extensions for Segment Routing
- RFC 8664 - PCEP Extensions for Segment Routing
- RFC 9256 - Segment Routing Policy Architecture
- RFC 9862 - PCEP Extensions for SR Policy Candidate Paths
- RFC 9855 - TI-LFA using Segment Routing
- RFC 8986 - SRv6 Network Programming
