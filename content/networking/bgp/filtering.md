---
title: "Фильтрация"
---

# Фильтрация

## Коротко

Фильтрация контролирует какие маршруты принимать (import) и отдавать (export).
Инструменты: prefix-list, as-path filter, route-map/policy.

## Prefix-list

Фильтрация по префиксам и маскам.

<details>
<summary>Juniper</summary>

```
set policy-options prefix-list ALLOWED 10.0.0.0/8
set policy-options prefix-list ALLOWED 172.16.0.0/12

set policy-options policy-statement IMPORT-FILTER term 1 from prefix-list ALLOWED
set policy-options policy-statement IMPORT-FILTER term 1 then accept
set policy-options policy-statement IMPORT-FILTER then reject

set protocols bgp group UPSTREAM import IMPORT-FILTER
```

</details>

<details>
<summary>Huawei</summary>

```
ip ip-prefix ALLOWED index 10 permit 10.0.0.0 8
ip ip-prefix ALLOWED index 20 permit 172.16.0.0 12
#
route-policy IMPORT-FILTER permit node 10
  if-match ip-prefix ALLOWED
route-policy IMPORT-FILTER deny node 100
#
bgp 65000
  peer 10.0.0.1 route-policy IMPORT-FILTER import
```

</details>

## route-filter (Juniper) / prefix с масками

Гибкая фильтрация с указанием длины префикса.

<details>
<summary>Juniper</summary>

```
# exact - точное совпадение
set policy-options policy-statement FILTER term 1 from route-filter 10.0.0.0/8 exact

# orlonger - /8 и более специфичные (/9, /10, ...)
set policy-options policy-statement FILTER term 2 from route-filter 10.0.0.0/8 orlonger

# prefix-length-range - диапазон длин
set policy-options policy-statement FILTER term 3 from route-filter 10.0.0.0/8 prefix-length-range /16-/24

# upto - до указанной длины
set policy-options policy-statement FILTER term 4 from route-filter 10.0.0.0/8 upto /16
```

</details>

<details>
<summary>Huawei</summary>

```
# exact (equal)
ip ip-prefix EXACT index 10 permit 10.0.0.0 8

# orlonger (le 32)
ip ip-prefix ORLONGER index 10 permit 10.0.0.0 8 less-equal 32

# prefix-length-range
ip ip-prefix RANGE index 10 permit 10.0.0.0 8 greater-equal 16 less-equal 24
```

</details>

## AS-PATH filter

Фильтрация по содержимому AS_PATH (regexp).

<details>
<summary>Juniper</summary>

```
# Маршруты от прямого соседа (одна AS в пути)
set policy-options as-path DIRECT "^[0-9]+$"

# Маршруты через AS 65001
set policy-options as-path VIA-65001 ".*65001.*"

# Маршруты originated AS 65002
set policy-options as-path FROM-65002 "65002$"

set policy-options policy-statement AS-FILTER term 1 from as-path DIRECT
set policy-options policy-statement AS-FILTER term 1 then accept
```

</details>

<details>
<summary>Huawei</summary>

```
ip as-path-filter DIRECT index 10 permit ^[0-9]+$
ip as-path-filter VIA-65001 index 10 permit _65001_
ip as-path-filter FROM-65002 index 10 permit _65002$
#
route-policy AS-FILTER permit node 10
  if-match as-path-filter DIRECT
```

</details>

Regex шаблоны:

- `^` - начало строки
- `$` - конец строки
- `.` - любой символ
- `*` - 0 или более повторений
- `+` - 1 или более повторений
- `[0-9]` - цифра
- `_` (Huawei) - разделитель AS

## Ограничение количества префиксов

Защита от переполнения таблицы.

<details>
<summary>Juniper</summary>

```
set protocols bgp group CUSTOMER prefix-limit maximum 1000
set protocols bgp group CUSTOMER prefix-limit teardown 80
set protocols bgp group CUSTOMER prefix-limit teardown idle-timeout 30
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
  peer 10.0.0.1 route-limit 1000 80
```

</details>

При 80% лимита - предупреждение. При 100% - сессия рвется.

## Фильтрация по длине AS_PATH

Отбрасывать маршруты с слишком длинным путем.

<details>
<summary>Juniper</summary>

```
set policy-options policy-statement LIMIT-ASPATH term 1 from as-path-length 10 orlonger
set policy-options policy-statement LIMIT-ASPATH term 1 then reject
```

</details>

<details>
<summary>Huawei</summary>

```
route-policy LIMIT-ASPATH deny node 10
  if-match as-path-length 10 or-greater
route-policy LIMIT-ASPATH permit node 100
```

</details>

## Направления фильтрации

**import (in)** - фильтрация входящих маршрутов от соседа.

**export (out)** - фильтрация исходящих маршрутов к соседу.

<details>
<summary>Juniper</summary>

```
set protocols bgp group UPSTREAM import IMPORT-POLICY
set protocols bgp group UPSTREAM export EXPORT-POLICY
```

</details>

<details>
<summary>Huawei</summary>

```
bgp 65000
  peer 10.0.0.1 route-policy IMPORT-POLICY import
  peer 10.0.0.1 route-policy EXPORT-POLICY export
```

</details>
