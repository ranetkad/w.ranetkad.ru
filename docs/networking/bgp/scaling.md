# Масштабирование

## Коротко

Проблема iBGP: требуется full-mesh (n*(n-1)/2 сессий).
Решения: Route Reflector, Confederation, peer-group/templates.

## Route Reflector

RR получает маршруты от клиентов и отражает их другим клиентам.
Клиенты не нужны full-mesh между собой - только сессия с RR.

**Роли:**

- RR (Route Reflector) - отражает маршруты
- RR Client - подключен к RR
- Non-client - обычный iBGP-сосед

**Атрибуты:**

- ORIGINATOR_ID - router-id источника маршрута
- CLUSTER_LIST - список cluster-id пройденных RR

Защита от петель: если свой router-id в ORIGINATOR_ID или cluster-id в CLUSTER_LIST - маршрут отбрасывается.

??? example "Juniper"
    ```
    # На RR
    set protocols bgp group RR-CLIENTS type internal
    set protocols bgp group RR-CLIENTS cluster 1.1.1.1
    set protocols bgp group RR-CLIENTS neighbor 10.0.0.2
    set protocols bgp group RR-CLIENTS neighbor 10.0.0.3
    ```

??? example "Huawei"
    ```
    # На RR
    bgp 65000
      reflector cluster-id 1.1.1.1
      peer 10.0.0.2 as-number 65000
      peer 10.0.0.3 as-number 65000
    #
    ipv4-family unicast
      peer 10.0.0.2 reflect-client
      peer 10.0.0.3 reflect-client
    ```

**Иерархия RR:**

Можно строить несколько уровней RR для больших сетей.
RR верхнего уровня являются non-client друг для друга.

## Confederation

AS разбивается на sub-AS. Между sub-AS - eBGP-подобное поведение, но AS_PATH не удлиняется снаружи.

??? example "Juniper"
    ```
    set routing-options autonomous-system 65000
    set routing-options confederation 65000 members 65001
    set routing-options confederation 65000 members 65002

    set protocols bgp group CONFED type external
    set protocols bgp group CONFED peer-as 65002
    set protocols bgp group CONFED neighbor 10.0.0.2
    ```

??? example "Huawei"
    ```
    bgp 65001
      confederation id 65000
      confederation peer-as 65002
      peer 10.0.0.2 as-number 65002
    ```

**Когда использовать:**

- RR проще, подходит для большинства случаев
- Confederation лучше для очень больших AS с независимыми регионами

## Peer-group

Группировка соседей с одинаковыми политиками. Упрощает конфиг и снижает нагрузку на CPU.

??? example "Juniper"
    ```
    set protocols bgp group UPSTREAM type external
    set protocols bgp group UPSTREAM import IMPORT-POLICY
    set protocols bgp group UPSTREAM export EXPORT-POLICY
    set protocols bgp group UPSTREAM neighbor 10.0.0.1 peer-as 65001
    set protocols bgp group UPSTREAM neighbor 10.0.0.2 peer-as 65002
    ```

??? example "Huawei"
    ```
    bgp 65000
      group UPSTREAM external
      peer UPSTREAM as-number 65001
      peer 10.0.0.1 group UPSTREAM
      peer 10.0.0.2 group UPSTREAM
    #
    ipv4-family unicast
      peer UPSTREAM enable
      peer UPSTREAM route-policy IMPORT-POLICY import
    ```

## Dynamic neighbors

Автоматическое принятие BGP-сессий из диапазона IP.

??? example "Juniper"
    ```
    set protocols bgp group DYNAMIC-PEERS type external
    set protocols bgp group DYNAMIC-PEERS peer-as 65001
    set protocols bgp group DYNAMIC-PEERS allow 10.0.0.0/24
    ```

??? example "Huawei"
    ```
    bgp 65000
      peer 10.0.0.0 255.255.255.0 as-number 65001
    ```

## Peer templates (Huawei)

Шаблоны для переиспользования конфигурации.

??? example "Huawei"
    ```
    bgp 65000
      peer UPSTREAM-TEMPLATE as-number 65001
      peer UPSTREAM-TEMPLATE ebgp-max-hop 2
      peer UPSTREAM-TEMPLATE route-policy IMPORT import

      peer 10.0.0.1 inherit peer UPSTREAM-TEMPLATE
      peer 10.0.0.2 inherit peer UPSTREAM-TEMPLATE
    ```
