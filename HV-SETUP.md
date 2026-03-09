# Настройка гипервизоров: практическое руководство

| | |
|---|---|
| **Тип** | Руководство по развёртыванию |
| **Статус** | Verified |
| **Создан** | 2026-03-09 |
| **Область** | HV setup / PoC с контейнерами |

---

## Оглавление

0. [Переменные стенда](#0-переменные-стенда)
1. [Топология стенда](#1-топология-стенда)
2. [Установка компонентов](#2-установка-компонентов)
3. [Сетевой стек (на каждом HV)](#3-сетевой-стек-на-каждом-hv)
4. [FRR конфигурация](#4-frr-конфигурация)
5. [Контейнеры](#5-контейнеры)
6. [Security: anti-spoofing](#6-security-anti-spoofing)
7. [Проверка связности](#7-проверка-связности)
8. [Устойчивость к перезагрузке](#8-устойчивость-к-перезагрузке)
9. [Диагностика](#9-диагностика)

---

## 0. Переменные стенда

При пересоздании VM меняются только **underlay IP**. Задайте переменные **один раз** на каждом HV перед выполнением команд — все bash-блоки ниже используют `$LOCAL_IP`, `$RR1_IP`, `$RR2_IP` и т.д.

```bash
# ─── Underlay IP (менять при каждом пересоздании VM) ───
RR1_IP=10.123.0.23
RR2_IP=10.123.0.6
HV1_IP=10.123.0.25
HV2_IP=10.123.0.20

# ─── Роль этого узла (выбрать ОДНО значение) ───
NODE=hv1        # ← на HV1
# NODE=hv2      # ← на HV2

# ─── Вычисляемые переменные (не менять) ───
if [ "$NODE" = "hv1" ]; then
  export LOCAL_IP=$HV1_IP  PEER_HV_IP=$HV2_IP
elif [ "$NODE" = "hv2" ]; then
  export LOCAL_IP=$HV2_IP  PEER_HV_IP=$HV1_IP
else
  echo "ОШИБКА: NODE должен быть hv1 или hv2" >&2
fi

export RR1_IP RR2_IP
export BGP_ASN=65000
export BGP_PASSWORD="in-cloud-bgp-secret"

echo "[$NODE] LOCAL_IP=$LOCAL_IP  PEER_HV_IP=$PEER_HV_IP"
```

> **Важно**: bash-блоки далее используют **unquoted heredocs** (`<<DELIM` вместо `<<'DELIM'`),
> чтобы shell подставлял переменные. Переменные должны быть
> экспортированы (`export`) перед выполнением команд.

---

## 1. Топология стенда

Два гипервизора, два VPC с **одинаковой сетью** (`10.0.0.0/24`), контейнеры вместо VM.

VPC изолированы через **VRF** (Virtual Routing and Forwarding) — у каждого VPC своя таблица маршрутизации внутри ядра Linux. Благодаря этому один и тот же CIDR `10.0.0.0/24` может использоваться в обоих VPC без конфликтов.

Трафик между гипервизорами передаётся через **VXLAN** — overlay-туннель поверх обычной L3-сети (underlay). Каждый VPC получает свой **VNI** (VXLAN Network Identifier), что обеспечивает изоляцию на уровне инкапсуляции.

Обнаружение remote-эндпоинтов выполняет **EVPN** (Ethernet VPN) через **BGP** — FRR-демон на каждом HV обменивается MAC/IP-адресами с соседями, избавляя от необходимости flood & learn.

### 1.1 Схема

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    UNDERLAY (L3, связность между HV)                     │
│                                                                           │
│            HV1: ${HV1_IP}          HV2: ${HV2_IP}                     │
│                                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                              OVERLAY                                      │
│                                                                           │
│  VPC-1 (VNI 100001, Subnet 10.0.0.0/24):                                │
│                                                                           │
│    HV1:  container-a1  10.0.0.1                                          │
│    HV2:  container-a2  10.0.0.2                                          │
│                                                                           │
│  VPC-2 (VNI 100002, Subnet 10.0.0.0/24):                                │
│                                                                           │
│    HV1:  container-b1  10.0.0.3                                          │
│    HV2:  container-b2  10.0.0.4                                          │
│                                                                           │
│  Одинаковый CIDR, но разные VRF и VNI → полная изоляция                 │
│  container-a1 видит container-a2, но НЕ видит container-b1/b2           │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Параметры

| Параметр | HV1 | HV2 |
|---|---|---|
| Underlay IP | `$HV1_IP` | `$HV2_IP` |
| BGP ASN | `$BGP_ASN` | `$BGP_ASN` (iBGP) |
| BGP Router ID | `$LOCAL_IP` | `$LOCAL_IP` |

| VPC | VNI | Subnet | HV1 контейнер | HV2 контейнер |
|---|---|---|---|---|
| vpc-1 | 100001 | 10.0.0.0/24 | container-a1 (10.0.0.1) | container-a2 (10.0.0.2) |
| vpc-2 | 100002 | 10.0.0.0/24 | container-b1 (10.0.0.3) | container-b2 (10.0.0.4) |

### 1.3 Стек на каждом HV

```
Контейнер (namespace)
  └── eth0 (veth guest-end)
        │
        ▼
  veth host-end ──► bridge (br-VNIID) ──► vxlan (VNI) ──► UDP:4789 ──► underlay
                         │
                    master: VRF
```

Каждый компонент стека отвечает за свой уровень:

| Компонент | Роль |
|---|---|
| **VRF** | Изолированная таблица маршрутизации; весь трафик VPC живёт внутри своего VRF |
| **Bridge** | L2-коммутация внутри VPC на одном HV; связывает veth-интерфейсы контейнеров с VXLAN |
| **VXLAN** | Tunnel endpoint — инкапсулирует Ethernet-кадры в UDP для передачи через underlay |
| **veth pair** | Виртуальный кабель; один конец в namespace контейнера (eth0), другой — порт bridge'а |
| **FRR (BGP EVPN)** | Control plane — обмен MAC/IP-информацией между HV, заполняет FDB на VXLAN |

### 1.4 Что проверяем

- container-a1 (HV1) ↔ container-a2 (HV2) — связность внутри VPC-1 через VXLAN
- container-b1 (HV1) ↔ container-b2 (HV2) — связность внутри VPC-2 через VXLAN
- container-a1 ✗ container-b1 — изоляция между VPC (разные VRF, разные VNI, одинаковый CIDR)

---

## 2. Установка компонентов

Выполнить на **обоих HV**.

### 2.1 Модули ядра

VRF, VXLAN и bridge-netfilter реализованы как модули ядра Linux. Без них команды создания интерфейсов вернут `Error: Unknown device type.`

```bash
sudo apt update
sudo apt install -y linux-modules-extra-$(uname -r)

sudo modprobe vrf
sudo modprobe vxlan
sudo modprobe br_netfilter

sudo ip link add test-vrf type vrf table 999 && \
  echo "VRF: OK" && \
  sudo ip link del test-vrf || \
  echo "VRF: FAILED — ядро не поддерживает VRF"

cat <<'EOF' | sudo tee /etc/modules-load.d/in-cloud.conf
vrf
vxlan
br_netfilter
EOF
```

### 2.2 FRR (BGP/EVPN daemon)

FRRouting — routing suite с поддержкой BGP, EVPN, zebra. Именно через него HV обмениваются MAC/IP-таблицами, избавляя VXLAN от broadcast-flood.

```bash
curl -s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null
FRRVER="frr-stable"
echo "deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER" | \
  sudo tee /etc/apt/sources.list.d/frr.list

sudo apt update
sudo apt install -y frr frr-pythontools
```

### 2.3 Включить нужные демоны FRR

По умолчанию bgpd и zebra выключены в `/etc/frr/daemons`. Без них FRR не будет обрабатывать BGP-сессии и не будет программировать FDB/маршруты в ядро.

- **zebra** — взаимодействует с ядром (маршруты, интерфейсы, FDB)
- **bgpd** — BGP-сессии, обмен EVPN type-2/3 маршрутами

```bash
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^zebra=no/zebra=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

### 2.4 Системные утилиты

```bash
sudo apt install -y iproute2 bridge-utils docker.io jq nftables
```

### 2.5 Системные параметры ядра

```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/99-in-cloud.conf
# Разрешить пересылку пакетов между интерфейсами — без этого HV не будет
# маршрутизировать трафик между контейнерами и VXLAN-туннелем
net.ipv4.ip_forward = 1

# Разрешить сервисам (TCP/UDP) принимать соединения внутри VRF —
# без этого процессы в default VRF не смогут обслуживать трафик из VRF
net.ipv4.tcp_l3mdev_accept = 1
net.ipv4.udp_l3mdev_accept = 1

# Отключить Reverse Path Filtering — rp_filter отбрасывает пакеты, если
# обратный маршрут идёт через другой интерфейс. С VRF и VXLAN это ложные
# срабатывания, так как пакет приходит через VXLAN, а обратный путь через bridge
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# bridge-nf-call-iptables: контролирует, пропускать ли L2-трафик bridge'а через iptables.
#
# Когда = 1 (по умолчанию, Docker включает при старте):
#   Кадры, коммутируемые bridge'ем на L2 уровне, попадают в цепочки iptables
#   FORWARD/INPUT/OUTPUT. Docker использует это для изоляции своих сетей.
#   Проблема: наши bridge'и (br-100001, br-100002) — не Docker-сети. Docker'овские
#   правила их не знают, политика FORWARD = DROP → кадры молча отбрасываются:
#     container → veth → br-100001 → [iptables FORWARD: DROP] → кадр потерян
#
# Когда = 0:
#   Bridge-трафик обрабатывается только на L2 — iptables его не видит.
#   Кадры свободно коммутируются между портами bridge'а, включая VXLAN:
#     container → veth → br-100001 → vxlan-100001 → UDP → underlay
#   Маршрутизируемый (L3) трафик по-прежнему проходит через iptables как обычно.
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
EOF

sudo sysctl --system
```

### 2.6 nftables: разрешить VXLAN и forwarding

Docker устанавливает политику `FORWARD DROP` в netfilter. Хотя bridge-nf-call отключён (L2-трафик через bridge не фильтруется), входящие VXLAN UDP-пакеты на порт 4789 проходят через цепочку INPUT — её нужно открыть.

> **Security**: принимаем VXLAN UDP **только от известных peer IP**. Без этого
> ограничения любой хост в underlay-сети может отправить crafted VXLAN-пакет
> с произвольным VNI и инжектировать трафик в VPC (VXLAN injection attack).

**На каждом HV** (переменная `$PEER_HV_IP` задана в разделе 0):

> VXLAN-трафик идёт **напрямую между HV**, минуя RR. Поэтому в `allowed_vteps`
> добавляем IP других HV, **не** IP Route Reflector.

```bash
sudo nft -f - <<NFT
table inet in-cloud-firewall {
  set allowed_vteps {
    type ipv4_addr
    elements = { $PEER_HV_IP }
  }

  chain input {
    type filter hook input priority 0; policy accept;
    udp dport 4789 ip saddr @allowed_vteps accept
    udp dport 4789 drop
  }

  chain forward {
    type filter hook forward priority 0; policy accept;
  }
}
NFT
```

> В production с N гипервизорами — Agent динамически добавляет VTEP IP в set:
> `nft add element inet in-cloud-firewall allowed_vteps { 10.123.0.99 }`
> Lookup по set — O(1), не зависит от количества peer'ов.

> Эти правила не переживают перезагрузку. Для persistence см. раздел 8.

### 2.7 Проверка готовности

```bash
echo "=== Модули ==="
lsmod | grep -E 'vrf|vxlan' && echo "OK" || echo "ОШИБКА: модули не загружены"

echo "=== FRR ==="
sudo systemctl is-active frr && echo "OK" || echo "FRR не запущен"

echo "=== Docker ==="
sudo docker info > /dev/null 2>&1 && echo "OK" || echo "Docker не работает"

echo "=== IP forwarding ==="
sysctl net.ipv4.ip_forward | grep "= 1" && echo "OK" || echo "ОШИБКА"

echo "=== bridge-nf-call-iptables ==="
cat /proc/sys/net/bridge/bridge-nf-call-iptables | grep "0" && echo "OK" || echo "ОШИБКА: должно быть 0"

echo "=== nftables: VXLAN firewall ==="
sudo nft list table inet in-cloud-firewall > /dev/null 2>&1 && echo "OK" || echo "ОШИБКА: таблица in-cloud-firewall не создана"
```

---

## 3. Сетевой стек (на каждом HV)

Выполнить на **каждом HV**. Единственное отличие между HV — `$LOCAL_IP` в VXLAN-интерфейсах; переменная задана в разделе 0.

На этом этапе создаём сетевую инфраструктуру хоста: VRF для изоляции, VXLAN-туннели для overlay, bridge'и для L2-коммутации. Контейнеры и veth-пары создаются позже (раздел 5).

### 3.1 VRF

Каждый VPC получает свой VRF — это изолированная таблица маршрутизации в ядре Linux. Трафик VPC-1 не может «увидеть» маршруты VPC-2, даже если подсети одинаковые.

`table 100` / `table 200` — номера routing-таблиц ядра; должны быть уникальны на хосте.

```bash
sudo ip link add vrf-vpc-1 type vrf table 100
sudo ip link set vrf-vpc-1 up

sudo ip link add vrf-vpc-2 type vrf table 200
sudo ip link set vrf-vpc-2 up
```

### 3.2 VXLAN интерфейсы

VXLAN — overlay-туннель. Каждый VXLAN-интерфейс — это VTEP (VXLAN Tunnel Endpoint). Он инкапсулирует Ethernet-кадры в UDP и отправляет их через underlay-сеть на remote VTEP.

- `id 100001` — **VNI** (VXLAN Network Identifier); должен совпадать на обоих HV для одного VPC
- `local $LOCAL_IP` — source IP для VXLAN-пакетов (underlay-адрес этого HV)
- `dstport 4789` — стандартный UDP-порт VXLAN (RFC 7348)
- `nolearning` — отключает data-plane MAC-learning; MAC-таблица заполняется через EVPN control plane (FRR), что надёжнее и масштабируемее

```bash
sudo ip link add vxlan-100001 type vxlan id 100001 local $LOCAL_IP dstport 4789 nolearning
sudo ip link set vxlan-100001 up

sudo ip link add vxlan-100002 type vxlan id 100002 local $LOCAL_IP dstport 4789 nolearning
sudo ip link set vxlan-100002 up
```

### 3.3 Bridge

Bridge выполняет L2-коммутацию: связывает veth-интерфейсы контейнеров с VXLAN-туннелем. Когда кадр поступает на bridge от контейнера, bridge смотрит FDB-таблицу и решает — отправить кадр локально (другому порту) или через VXLAN на remote VTEP.

- `master vrf-vpc-1` — помещает bridge в VRF; весь трафик bridge'а принадлежит этому VRF
- VXLAN-интерфейс добавляется как порт bridge'а — декапсулированные кадры попадают в bridge

Порядок важен: сначала `master vrf`, потом `up`, потом добавление VXLAN в bridge. Иначе FRR может неправильно определить принадлежность VNI к VRF.

```bash
sudo ip link add br-100001 type bridge
sudo ip link set br-100001 master vrf-vpc-1
sudo ip link set br-100001 up
sudo ip link set vxlan-100001 master br-100001

sudo ip link add br-100002 type bridge
sudo ip link set br-100002 master vrf-vpc-2
sudo ip link set br-100002 up
sudo ip link set vxlan-100002 master br-100002
```

### 3.4 Gateway IP

IP-адрес на bridge работает как **default gateway** для контейнеров этого VPC. Контейнер отправляет пакет на `10.0.0.254`, bridge принимает его, ядро маршрутизирует через VRF-таблицу.

Один и тот же адрес `10.0.0.254` на обоих bridge'ах — не конфликтует, потому что bridge'и в разных VRF (разные таблицы маршрутизации).

```bash
sudo ip addr add 10.0.0.254/24 dev br-100001
sudo ip addr add 10.0.0.254/24 dev br-100002
```

### 3.5 Проверка

```bash
# Должен показать br-100001 и vxlan-100001
ip link show master vrf-vpc-1

# Должен показать br-100002 и vxlan-100002
ip link show master vrf-vpc-2

# Все порты bridge'ей
bridge link show
```

---

## 4. FRR конфигурация

FRR — routing suite, который поднимает BGP-сессию с Route Reflector и обменивается EVPN-маршрутами. Через EVPN каждый HV узнаёт, какие MAC/IP-адреса живут на remote VTEP, и программирует FDB-таблицу VXLAN-интерфейса.

HV — **BGP client** двух Route Reflector (RR1 + RR2). RR отражает маршруты между клиентами. HV не пирятся друг с другом напрямую — это позволяет масштабироваться без full-mesh. Настройка RR — см. [RR-SETUP.md](RR-SETUP.md).

> **Важно**: конфигурируем через `vtysh`, а не через запись в `/etc/frr/frr.conf`.
> Прямая запись в файл с `frr version 10.0` вызывает ошибки парсинга на FRR 10.5.x —
> VNI-маппинг «съезжает» между VRF. Команда `vtysh` + `write memory` гарантирует
> корректный формат для текущей версии FRR.

### 4.1 BGP (на каждом HV)

Одна и та же конфигурация на обоих HV — `$LOCAL_IP`, `$RR1_IP`, `$RR2_IP`, `$BGP_ASN`, `$BGP_PASSWORD` подставляются из переменных раздела 0.

```bash
sudo vtysh <<VTYSH
configure terminal
!
! --- Основной BGP-процесс (default VRF) ---
router bgp $BGP_ASN
 ! Router ID — уникальный идентификатор в BGP, используем underlay IP
 bgp router-id $LOCAL_IP
 ! Отключаем IPv4 unicast по умолчанию — нам нужен только L2VPN EVPN
 no bgp default ipv4-unicast
 ! Отключаем проверку политик для eBGP (у нас iBGP, но на всякий случай)
 no bgp ebgp-requires-policy
 !
 ! --- Route Reflector 1 ---
 neighbor $RR1_IP remote-as $BGP_ASN
 neighbor $RR1_IP update-source $LOCAL_IP
 neighbor $RR1_IP password $BGP_PASSWORD
 !
 ! --- Route Reflector 2 (HA) ---
 neighbor $RR2_IP remote-as $BGP_ASN
 neighbor $RR2_IP update-source $LOCAL_IP
 neighbor $RR2_IP password $BGP_PASSWORD
 !
 ! ttl-security hops 1 — НЕ используем на облачных VM:
 ! underlay провайдера добавляет 2–4 хопа, пакеты приходят с TTL ~61,
 ! а ttl-security требует TTL >= 254. На bare metal (directly connected) — включить.
 ! iBGP по умолчанию использует TTL=255 — дополнительных настроек не нужно.
 !
 ! Активируем EVPN address-family для обмена MAC/IP маршрутами
 address-family l2vpn evpn
  neighbor $RR1_IP activate
  neighbor $RR2_IP activate
  ! FRR автоматически обнаруживает все VNI (VXLAN-интерфейсы в bridge'ах)
  ! и анонсирует их через EVPN type-2 (MAC/IP) и type-3 (BUM) маршруты
  advertise-all-vni
 exit-address-family
exit
!
! --- BGP для VRF vpc-1 ---
! Отдельный BGP-процесс в VRF — для redistribution connected/static маршрутов
router bgp $BGP_ASN vrf vrf-vpc-1
 address-family ipv4 unicast
  ! Анонсировать connected-маршруты (10.0.0.0/24 от bridge) в BGP
  redistribute connected
  ! Анонсировать static-маршруты (если будут)
  redistribute static
 exit-address-family
 address-family l2vpn evpn
  ! Анонсировать IPv4 маршруты VRF через EVPN type-5 (IP prefix route)
  advertise ipv4 unicast
 exit-address-family
exit
!
! --- BGP для VRF vpc-2 ---
router bgp $BGP_ASN vrf vrf-vpc-2
 address-family ipv4 unicast
  redistribute connected
  redistribute static
 exit-address-family
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
exit
!
end
! Сохранить конфиг в /etc/frr/frr.conf в формате текущей версии FRR
write memory
VTYSH
```

### 4.2 Security: BGP MD5 и TTL security

**MD5 auth** (`neighbor ... password`) — включена в PoC. TCP MD5 (RFC 2385) подписывает каждый TCP-сегмент BGP-сессии. Без правильного пароля атакующий не сможет установить BGP-сессию и инжектировать маршруты. Пароль должен совпадать на обоих peer'ах.

**TTL security** (`neighbor ... ttl-security hops 1`) — **не используется** на облачных VM. GTSM (RFC 5082) требует TTL >= 254 на входящих BGP-пакетах, что означает directly connected peer (1 хоп). Облачная underlay провайдера добавляет 2–4 промежуточные хопа — пакеты приходят с TTL ~61 и молча отбрасываются ядром.

Для **production** на bare metal (directly connected peers) добавить:

```
neighbor <PEER_IP> ttl-security hops 1
```

> **Симптомы при неправильной настройке:**
> - `ttl-security` на облачных VM → BGP `Idle`, `nc -zv <peer> 179` работает, но FRR не подключается
> - Разные пароли MD5 → SYN и SYN-ACK ходят, ACK не отправляется (ядро молча дропает при несовпадении MD5-подписи)
> - Оба peer'а одновременно в `Connect` → collision resolution: оба шлют SYN, один RST — это нормально, через секунду сессия поднимается

### 4.3 Почему нет блоков `vrf / vni`

В FRR конфигурации отсутствуют блоки вида:

```
vrf vrf-vpc-1
 vni 100001
exit-vrf
```

Это **сознательное решение**. Блок `vrf / vni` превращает VNI в **L3 VNI** (для маршрутизации между разными подсетями в одном VRF). В нашем PoC все контейнеры одного VPC находятся в **одной подсети** — им нужен **L2 VNI** (bridging в пределах одного L2-сегмента).

Без явного `vrf / vni` FRR автоматически обнаруживает VXLAN-интерфейсы через `advertise-all-vni` и регистрирует их как L2 VNI. Это именно то, что нужно для same-subnet связности.

L3 VNI понадобится позже, когда появится inter-subnet routing.

### 4.4 Проверка (на обоих HV)

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show evpn vni"
sudo vtysh -c "show vrf vni"
```

Ожидаемый результат:

```
# show bgp summary — обе сессии Established:
Neighbor        V   AS   ... Up/Down  State/PfxRcd
<RR1_IP>        4   65000 ... 00:05:xx  N
<RR2_IP>        4   65000 ... 00:05:xx  N

# show evpn vni — тип L2, Remote VTEP = 1:
VNI     Type  VxLAN IF       # MACs  # ARPs  # Remote VTEPs  Tenant VRF
100001  L2    vxlan-100001   ...     ...     1               vrf-vpc-1
100002  L2    vxlan-100002   ...     ...     1               vrf-vpc-2
```

Если VNI показывает тип **L3** вместо L2 — см. раздел 9.6 (типичные проблемы).

---

## 5. Контейнеры

В PoC контейнеры заменяют виртуальные машины. Каждый контейнер получает свой network namespace (через `--network none` + ручная настройка), что полностью изолирует его сетевой стек — аналогично VM.

> **Важно**: создание veth, запуск контейнера и перемещение veth в namespace
> выполняются **одним блоком**. Если создать veth в разделе 3, а контейнер запустить
> позже — guest-end veth останется в host namespace, host-end покажет `NO-CARRIER`,
> и трафик не пойдёт.

Для каждого контейнера выполняется 4 шага:

1. **veth pair** — виртуальный «кабель»; host-end идёт в bridge, guest-end — в контейнер
2. **docker run** — запуск контейнера без сети (`--network none`)
3. **nsenter** — перемещение guest-end veth в namespace контейнера и настройка IP/MAC/route
4. **ARP** — статическая ARP-запись на bridge, чтобы HV знал MAC контейнера без ARP-запроса

### 5.1 HV1: container-a1 (VPC-1, IP 10.0.0.1)

```bash
sudo ip link add veth-a1-h type veth peer name veth-a1-g
sudo ip link set veth-a1-h master br-100001
sudo ip link set veth-a1-h up

sudo docker run -d --name container-a1 \
  --network none \
  --cap-add NET_ADMIN \
  alpine:latest sleep infinity

A1_PID=$(sudo docker inspect -f '{{.State.Pid}}' container-a1)
sudo ip link set veth-a1-g netns $A1_PID
sudo nsenter -t $A1_PID -n ip link set veth-a1-g name eth0
sudo nsenter -t $A1_PID -n ip link set eth0 address 02:42:0a:00:00:01
sudo nsenter -t $A1_PID -n ip addr add 10.0.0.1/24 dev eth0
sudo nsenter -t $A1_PID -n ip link set eth0 up
sudo nsenter -t $A1_PID -n ip route add default via 10.0.0.254

sudo ip neigh replace 10.0.0.1 lladdr 02:42:0a:00:00:01 nud permanent dev br-100001
```

### 5.2 HV1: container-b1 (VPC-2, IP 10.0.0.3)

```bash
sudo ip link add veth-b1-h type veth peer name veth-b1-g
sudo ip link set veth-b1-h master br-100002
sudo ip link set veth-b1-h up

sudo docker run -d --name container-b1 \
  --network none \
  --cap-add NET_ADMIN \
  alpine:latest sleep infinity

B1_PID=$(sudo docker inspect -f '{{.State.Pid}}' container-b1)
sudo ip link set veth-b1-g netns $B1_PID
sudo nsenter -t $B1_PID -n ip link set veth-b1-g name eth0
sudo nsenter -t $B1_PID -n ip link set eth0 address 02:42:0a:00:00:03
sudo nsenter -t $B1_PID -n ip addr add 10.0.0.3/24 dev eth0
sudo nsenter -t $B1_PID -n ip link set eth0 up
sudo nsenter -t $B1_PID -n ip route add default via 10.0.0.254

sudo ip neigh replace 10.0.0.3 lladdr 02:42:0a:00:00:03 nud permanent dev br-100002
```

### 5.3 HV2: container-a2 (VPC-1, IP 10.0.0.2)

```bash
sudo ip link add veth-a2-h type veth peer name veth-a2-g
sudo ip link set veth-a2-h master br-100001
sudo ip link set veth-a2-h up

sudo docker run -d --name container-a2 \
  --network none \
  --cap-add NET_ADMIN \
  alpine:latest sleep infinity

A2_PID=$(sudo docker inspect -f '{{.State.Pid}}' container-a2)
sudo ip link set veth-a2-g netns $A2_PID
sudo nsenter -t $A2_PID -n ip link set veth-a2-g name eth0
sudo nsenter -t $A2_PID -n ip link set eth0 address 02:42:0a:00:00:02
sudo nsenter -t $A2_PID -n ip addr add 10.0.0.2/24 dev eth0
sudo nsenter -t $A2_PID -n ip link set eth0 up
sudo nsenter -t $A2_PID -n ip route add default via 10.0.0.254

sudo ip neigh replace 10.0.0.2 lladdr 02:42:0a:00:00:02 nud permanent dev br-100001
```

### 5.4 HV2: container-b2 (VPC-2, IP 10.0.0.4)

```bash
sudo ip link add veth-b2-h type veth peer name veth-b2-g
sudo ip link set veth-b2-h master br-100002
sudo ip link set veth-b2-h up

sudo docker run -d --name container-b2 \
  --network none \
  --cap-add NET_ADMIN \
  alpine:latest sleep infinity

B2_PID=$(sudo docker inspect -f '{{.State.Pid}}' container-b2)
sudo ip link set veth-b2-g netns $B2_PID
sudo nsenter -t $B2_PID -n ip link set veth-b2-g name eth0
sudo nsenter -t $B2_PID -n ip link set eth0 address 02:42:0a:00:00:04
sudo nsenter -t $B2_PID -n ip addr add 10.0.0.4/24 dev eth0
sudo nsenter -t $B2_PID -n ip link set eth0 up
sudo nsenter -t $B2_PID -n ip route add default via 10.0.0.254

sudo ip neigh replace 10.0.0.4 lladdr 02:42:0a:00:00:04 nud permanent dev br-100002
```

---

## 6. Security: anti-spoofing

Контейнер (или VM в production) — **недоверенная зона**. Он может попытаться:
- Сменить MAC-адрес и перехватить чужой трафик (MAC spoofing)
- Назначить себе чужой IP и отвечать за другую VM (IP spoofing)
- Отправить gratuitous ARP с чужим IP/MAC и отравить ARP-таблицу bridge'а (ARP spoofing)

Все правила ниже применяются на **host-end veth** (на стороне гипервизора, вне контейнера) — контейнер не может их обойти.

Используем **nftables** (bridge family) — современная замена ebtables. Преимущества:
- Компиляция правил в bytecode VM — быстрее последовательной проверки ebtables
- Поддержка sets/maps — O(1) lookup по MAC/IP вместо линейного перебора
- Единый фреймворк для L2/L3/L4 фильтрации
- В production с сотнями NI на HV разница в скорости существенная

### 6.1 Установка nftables

```bash
sudo apt install -y nftables
sudo systemctl enable nftables
```

### 6.2 Anti-spoofing правила HV1

Все три уровня защиты (MAC, IP, ARP) описываются в одной таблице nftables.
Цепочка `input` в bridge family перехватывает кадры на входе в bridge — до коммутации.

```bash
sudo nft -f - <<'NFT'
table bridge antispoofing {

  chain input {
    type filter hook input priority 0; policy accept;

    # --- container-a1 (VPC-1, MAC 02:42:0a:00:00:01, IP 10.0.0.1) ---

    iif veth-a1-h ether saddr != 02:42:0a:00:00:01 drop

    iif veth-a1-h ether type ip ip saddr != 10.0.0.1 drop

    iif veth-a1-h ether type arp arp saddr ip != 10.0.0.1 drop

    # --- container-b1 (VPC-2, MAC 02:42:0a:00:00:03, IP 10.0.0.3) ---

    iif veth-b1-h ether saddr != 02:42:0a:00:00:03 drop
    iif veth-b1-h ether type ip ip saddr != 10.0.0.3 drop
    iif veth-b1-h ether type arp arp saddr ip != 10.0.0.3 drop
  }
}
NFT
```

### 6.3 Anti-spoofing правила HV2

```bash
sudo nft -f - <<'NFT'
table bridge antispoofing {
  chain input {
    type filter hook input priority 0; policy accept;

    # --- container-a2 (VPC-1, MAC 02:42:0a:00:00:02, IP 10.0.0.2) ---
    iif veth-a2-h ether saddr != 02:42:0a:00:00:02 drop
    iif veth-a2-h ether type ip ip saddr != 10.0.0.2 drop
    iif veth-a2-h ether type arp arp saddr ip != 10.0.0.2 drop

    # --- container-b2 (VPC-2, MAC 02:42:0a:00:00:04, IP 10.0.0.4) ---
    iif veth-b2-h ether saddr != 02:42:0a:00:00:04 drop
    iif veth-b2-h ether type ip ip saddr != 10.0.0.4 drop
    iif veth-b2-h ether type arp arp saddr ip != 10.0.0.4 drop
  }
}
NFT
```

### 6.4 Дополнительные sysctl-параметры

```bash
sudo sysctl -w net.ipv4.conf.br-100001.arp_accept=0
sudo sysctl -w net.ipv4.conf.br-100002.arp_accept=0
```

### 6.5 Проверка

```bash
sudo nft list table bridge antispoofing
sudo nft list counters table bridge antispoofing 2>/dev/null || true
```

#### MAC spoofing

```bash
# Подменить MAC — трафик должен быть заблокирован
sudo docker exec container-a1 ip link set eth0 address 02:42:0a:00:00:99
sudo docker exec container-a1 ping -c 2 10.0.0.254
# Ожидание: 100% packet loss

# Вернуть правильный MAC
sudo docker exec container-a1 ip link set eth0 address 02:42:0a:00:00:01
sudo docker exec container-a1 ping -c 2 10.0.0.254
# Ожидание: пинг работает
```

#### IP spoofing

```bash
# Добавить чужой IP и отправить трафик с него
sudo docker exec container-a1 ip addr add 10.0.0.99/24 dev eth0
sudo docker exec container-a1 ping -c 2 -I 10.0.0.99 10.0.0.254
# Ожидание: 100% packet loss (IP saddr != 10.0.0.1 → drop)

# Убрать чужой IP, проверить что легитимный трафик работает
sudo docker exec container-a1 ip addr del 10.0.0.99/24 dev eth0
sudo docker exec container-a1 ping -c 2 -I 10.0.0.1 10.0.0.254
# Ожидание: пинг работает
```

#### ARP spoofing

```bash
# Отправить gratuitous ARP с чужим IP (требуется arping)
sudo docker exec container-a1 arping -c 2 -s 10.0.0.99 -U 10.0.0.254 -I eth0
# Ожидание: ARP заблокирован (arp saddr ip != 10.0.0.1 → drop)

# Проверить что ARP с правильным IP проходит
sudo docker exec container-a1 arping -c 2 -s 10.0.0.1 10.0.0.254 -I eth0
# Ожидание: ответ получен
```

#### Cross-VPC изоляция

```bash
# container-a1 (VPC-1, 10.0.0.1) не должен достучаться до container-b1 (VPC-2, 10.0.0.3)
# даже если IP совпадают в адресном пространстве — они в разных bridge/VRF
sudo docker exec container-a1 ping -c 2 10.0.0.3
# Ожидание: 100% packet loss или "Network is unreachable"
```

### 6.6 Persistence

nftables правила не переживают перезагрузку. Сохраняем все таблицы в файлы:

```bash
sudo nft list table inet in-cloud-firewall > /etc/nftables-firewall.conf

sudo nft list table bridge antispoofing > /etc/nftables-antispoofing.conf

echo 'include "/etc/nftables-firewall.conf"' | sudo tee -a /etc/nftables.conf
echo 'include "/etc/nftables-antispoofing.conf"' | sudo tee -a /etc/nftables.conf
```

### 6.7 Production: динамическое управление правилами

В production Agent добавляет/удаляет правила при создании/удалении NI:

```bash
sudo nft add rule bridge antispoofing input iif veth-NEW-h ether saddr != <MAC> drop
sudo nft add rule bridge antispoofing input iif veth-NEW-h ether type ip ip saddr != <IP> drop
sudo nft add rule bridge antispoofing input iif veth-NEW-h ether type arp arp saddr ip != <IP> drop
```

При большом количестве NI на HV — использовать **nftables sets** для O(1) lookup:

```bash
sudo nft add set bridge antispoofing allowed_macs { type ifname . ether_addr \; }
sudo nft add element bridge antispoofing allowed_macs { veth-a1-h . 02:42:0a:00:00:01 }
sudo nft add rule bridge antispoofing input ether saddr != @allowed_macs drop
```

### 6.8 Сводка защит

| Атака | Защита | Уровень | Инструмент |
|---|---|---|---|
| MAC spoofing | Фильтр source MAC на veth | L2 | nftables `ether saddr` |
| IP spoofing | Фильтр source IP на veth | L3 | nftables `ip saddr` |
| ARP spoofing | Фильтр ARP sender IP на veth | L2 | nftables `arp saddr ip` |
| VXLAN injection | VXLAN UDP только от peer IP | L3 | nftables `@allowed_vteps` |
| BGP hijacking | MD5 auth + TTL security | L4 | FRR `password` (PoC) + `ttl-security` (bare metal) |

---

## 7. Проверка связности

### 7.1 VPC-1: container-a1 (HV1) ↔ container-a2 (HV2)

Трафик идёт: container-a1 → veth → br-100001 → vxlan-100001 → UDP:4789 через underlay → HV2 vxlan-100001 → br-100001 → veth → container-a2.

```bash
# С HV1:
sudo docker exec container-a1 ping -c 3 10.0.0.2

# С HV2:
sudo docker exec container-a2 ping -c 3 10.0.0.1
```

### 7.2 VPC-2: container-b1 (HV1) ↔ container-b2 (HV2)

```bash
# С HV1:
sudo docker exec container-b1 ping -c 3 10.0.0.4

# С HV2:
sudo docker exec container-b2 ping -c 3 10.0.0.3
```

### 7.3 Изоляция VPC: container-a1 ✗ container-b1

Даже если IP-адреса в одном диапазоне, контейнеры разных VPC не могут общаться — ARP-запрос уходит в bridge VPC-1, а container-b1 подключён к bridge VPC-2. Они находятся в разных L2-доменах.

```bash
# VPC-1 → VPC-2 на том же HV (не должен пройти)
sudo docker exec container-a1 ping -c 3 -W 2 10.0.0.3

# VPC-1 → VPC-2 межхостовая (не должен пройти)
sudo docker exec container-a1 ping -c 3 -W 2 10.0.0.4
```

### 7.4 Сводка ожидаемых результатов

| Откуда | Куда | VPC | Ожидание |
|---|---|---|---|
| container-a1 (HV1, 10.0.0.1) | container-a2 (HV2, 10.0.0.2) | VPC-1 ↔ VPC-1 | **Работает** |
| container-a2 (HV2, 10.0.0.2) | container-a1 (HV1, 10.0.0.1) | VPC-1 ↔ VPC-1 | **Работает** |
| container-b1 (HV1, 10.0.0.3) | container-b2 (HV2, 10.0.0.4) | VPC-2 ↔ VPC-2 | **Работает** |
| container-b2 (HV2, 10.0.0.4) | container-b1 (HV1, 10.0.0.3) | VPC-2 ↔ VPC-2 | **Работает** |
| container-a1 (HV1, 10.0.0.1) | container-b1 (HV1, 10.0.0.3) | VPC-1 ✗ VPC-2 | **Изолировано** |
| container-a1 (HV1, 10.0.0.1) | container-b2 (HV2, 10.0.0.4) | VPC-1 ✗ VPC-2 | **Изолировано** |

---

## 8. Устойчивость к перезагрузке

Сетевые интерфейсы (VRF, VXLAN, bridge) не переживают reboot. FRR конфиг сохраняется через `write memory`, но ему нужны уже созданные интерфейсы при старте. nftables правила сохраняются отдельно (раздел 6.6).

Скрипт ниже выполняется **до FRR** (`Before=frr.service`): создаёт все интерфейсы, настраивает sysctl. После этого FRR стартует и обнаруживает VNI.

### 8.1 Скрипт (на каждом HV)

`$LOCAL_IP` подставляется из env при создании файла — в скрипт записывается фактический IP.

```bash
sudo tee /usr/local/bin/in-cloud-setup.sh <<SCRIPT
#!/bin/bash
set -e

sysctl -w net.bridge.bridge-nf-call-iptables=0
sysctl -w net.bridge.bridge-nf-call-ip6tables=0

# VRF
ip link add vrf-vpc-1 type vrf table 100 2>/dev/null || true
ip link set vrf-vpc-1 up
ip link add vrf-vpc-2 type vrf table 200 2>/dev/null || true
ip link set vrf-vpc-2 up

# VXLAN
ip link add vxlan-100001 type vxlan id 100001 local $LOCAL_IP dstport 4789 nolearning 2>/dev/null || true
ip link set vxlan-100001 up
ip link add vxlan-100002 type vxlan id 100002 local $LOCAL_IP dstport 4789 nolearning 2>/dev/null || true
ip link set vxlan-100002 up

# Bridge + привязка к VRF и VXLAN
ip link add br-100001 type bridge 2>/dev/null || true
ip link set br-100001 master vrf-vpc-1
ip link set br-100001 up
ip link set vxlan-100001 master br-100001

ip link add br-100002 type bridge 2>/dev/null || true
ip link set br-100002 master vrf-vpc-2
ip link set br-100002 up
ip link set vxlan-100002 master br-100002

# Gateway IP
ip addr add 10.0.0.254/24 dev br-100001 2>/dev/null || true
ip addr add 10.0.0.254/24 dev br-100002 2>/dev/null || true
SCRIPT

sudo chmod +x /usr/local/bin/in-cloud-setup.sh
```

### 8.2 Systemd unit (на каждом HV)

```bash
sudo tee /etc/systemd/system/in-cloud-setup.service <<'UNIT'
[Unit]
Description=in-cloud network setup
After=network.target
Before=frr.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/in-cloud-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable in-cloud-setup.service
```

---

## 9. Диагностика

### 9.1 BGP сессии

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show running-config"
```

### 9.2 EVPN маршруты

```bash
sudo vtysh -c "show bgp l2vpn evpn"
sudo vtysh -c "show evpn vni"
sudo vtysh -c "show evpn mac vni 100001"
sudo vtysh -c "show evpn mac vni 100002"
sudo vtysh -c "show vrf vni"
```

### 9.3 Сетевые интерфейсы

```bash
ip -d link show type vrf
ip -d link show type vxlan
bridge link show
bridge fdb show dev vxlan-100001
bridge fdb show dev vxlan-100002
```

### 9.4 Таблицы маршрутизации в VRF

```bash
ip route show vrf vrf-vpc-1
ip route show vrf vrf-vpc-2
```

### 9.5 VXLAN-трафик

```bash
sudo tcpdump -i eth0 -n 'udp port 4789' -c 10
```

### 9.6 Типичные проблемы

| Симптом | Причина | Решение |
|---|---|---|
| `Error: Unknown device type.` при создании VRF | Нет модуля VRF в ядре | `apt install linux-modules-extra-$(uname -r) && modprobe vrf` |
| BGP сессия не Established | Нет L3-связности underlay | `ping $RR1_IP` и `ping $RR2_IP` с HV |
| BGP `Idle`, nc:179 OK | `ttl-security hops 1` + облачные VM (TTL ~61) | Убрать `ttl-security`; для iBGP TTL=255 по умолчанию |
| BGP `Connect`, SYN/SYN-ACK ходят, ACK нет | TCP MD5 подпись не проходит проверку | Выровнять `password` на обоих HV; проверить `tcpdump -M secret` |
| VXLAN пакеты уходят, не доходят (tcpdump пуст на peer) | Облачный провайдер блокирует UDP:4789 | Открыть UDP:4789 в security group провайдера |
| `show evpn vni` показывает L3 вместо L2 | В FRR есть блоки `vrf / vni` | `vtysh`: `configure terminal` → `vrf vrf-vpc-1` → `no vni 100001` → `write memory` |
| VNI маппинг сдвинут (VNI в неправильном VRF) | FRR парсит `frr.conf` другой версии | Удалить `/etc/frr/frr.conf`, конфигурировать через `vtysh` + `write memory` |
| Ping не идёт, EVPN маршруты есть | Docker netfilter блокирует VXLAN | Создать таблицу `inet in-cloud-firewall` (раздел 2.6) + `sysctl net.bridge.bridge-nf-call-iptables=0` |
| veth `NO-CARRIER`, контейнер имеет другой eth0 | veth guest-end не перемещён в namespace | Пересоздать: удалить veth и контейнер, повторить секцию 5 |
| EVPN маршрутов нет | VNI не обнаружен FRR | Проверить: `show evpn vni`, VXLAN должен быть в bridge в VRF |
| Ping между VPC проходит (нет изоляции) | veth в неправильном bridge | `bridge link show`, проверить master |
| Remote VTEPs = 0 | BGP сессия не установлена | Сначала исправить BGP (`show bgp summary`) |

### 9.7 Cleanup

```bash
# Контейнеры
sudo docker rm -f container-a1 container-b1  # HV1
sudo docker rm -f container-a2 container-b2  # HV2

# veth (удаляются автоматически при удалении контейнера, но на всякий случай)
sudo ip link del veth-a1-h 2>/dev/null; sudo ip link del veth-b1-h 2>/dev/null  # HV1
sudo ip link del veth-a2-h 2>/dev/null; sudo ip link del veth-b2-h 2>/dev/null  # HV2

# Сетевые интерфейсы (удаление bridge автоматически освобождает порты)
sudo ip link del br-100001 2>/dev/null
sudo ip link del br-100002 2>/dev/null
sudo ip link del vxlan-100001 2>/dev/null
sudo ip link del vxlan-100002 2>/dev/null
sudo ip link del vrf-vpc-1 2>/dev/null
sudo ip link del vrf-vpc-2 2>/dev/null

# nftables
sudo nft delete table inet in-cloud-firewall 2>/dev/null
sudo nft delete table bridge antispoofing 2>/dev/null

# FRR конфиг
sudo vtysh -c "configure terminal" -c "no router bgp $BGP_ASN" -c "end" -c "write memory"
```
