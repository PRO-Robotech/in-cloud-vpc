# Настройка Route Reflector: практическое руководство

| | |
|---|---|
| **Тип** | Руководство по развёртыванию |
| **Статус** | Draft |
| **Создан** | 2026-03-09 |
| **Область** | Route Reflector / HA-пара |

---

## Оглавление

0. [Переменные стенда](#0-переменные-стенда)
1. [Топология](#1-топология)
2. [Установка компонентов](#2-установка-компонентов)
3. [FRR конфигурация](#3-frr-конфигурация)
4. [nftables: BGP firewall](#4-nftables-bgp-firewall)
5. [Проверка](#5-проверка)
6. [Устойчивость к перезагрузке](#6-устойчивость-к-перезагрузке)
7. [Диагностика](#7-диагностика)

---

## 0. Переменные стенда

При пересоздании VM меняются только **underlay IP**. Задайте переменные **один раз** на каждом RR перед выполнением команд.

```bash
# ─── Underlay IP (менять при каждом пересоздании VM) ───
RR1_IP=10.123.0.23
RR2_IP=10.123.0.6
HV1_IP=10.123.0.25
HV2_IP=10.123.0.20

# ─── Роль этого узла (выбрать ОДНО значение) ───
NODE=rr2        # ← на RR1
# NODE=rr2      # ← на RR2

# ─── Вычисляемые переменные (не менять) ───
if [ "$NODE" = "rr1" ]; then
  export LOCAL_IP=$RR1_IP  PEER_RR_IP=$RR2_IP
elif [ "$NODE" = "rr2" ]; then
  export LOCAL_IP=$RR2_IP  PEER_RR_IP=$RR1_IP
else
  echo "ОШИБКА: NODE должен быть rr1 или rr2" >&2
fi

export HV1_IP HV2_IP
export BGP_ASN=65000
export BGP_PASSWORD="in-cloud-bgp-secret"
export CLUSTER_ID="10.255.255.1"

echo "[$NODE] LOCAL_IP=$LOCAL_IP  PEER_RR_IP=$PEER_RR_IP"
```

> **CLUSTER_ID** — одинаковый на обоих RR. Это логический идентификатор кластера
> Route Reflector. Два RR с одним cluster-id образуют HA-пару: оба отражают
> одни и те же маршруты, HV получает маршруты от любого из них.

---

## 1. Топология

Route Reflector (RR) — **не маршрутизатор**. Через RR не идёт data-plane трафик. RR — оптимизация BGP-топологии: вместо full-mesh O(N²) между HV — 2×N сессий через RR-пару.

### 1.1 Схема

```
┌───────────────────────────────────────────────────────────────────────┐
│                         BGP CONTROL PLANE                              │
│                                                                         │
│    ┌────────────┐                         ┌────────────┐               │
│    │    RR1     │◄───────iBGP────────────►│    RR2     │               │
│    │ ${RR1_IP}  │                         │ ${RR2_IP}  │               │
│    └─────┬──────┘                         └──────┬─────┘               │
│          │                                       │                      │
│    ┌─────┴──────────────────┬────────────────────┴─────┐               │
│    │                        │                          │                │
│    ▼                        ▼                          ▼                │
│  ┌──────┐              ┌──────┐                   ┌──────┐             │
│  │ HV1  │              │ HV2  │        ...        │ HV N │             │
│  └──┬───┘              └──┬───┘                   └──┬───┘             │
│     │                     │                          │                  │
├─────┼─────────────────────┼──────────────────────────┼──────────────────┤
│     │              DATA PLANE (VXLAN)                │                  │
│     │                     │                          │                  │
│     └─────────────────────┴──────────────────────────┘                  │
│            VXLAN-трафик идёт напрямую HV ↔ HV, минуя RR                │
└───────────────────────────────────────────────────────────────────────┘
```

### 1.2 Параметры

| Параметр | RR1 | RR2 |
|---|---|---|
| Underlay IP | `$RR1_IP` | `$RR2_IP` |
| BGP ASN | `$BGP_ASN` | `$BGP_ASN` (iBGP) |
| BGP Router ID | `$LOCAL_IP` | `$LOCAL_IP` |
| Cluster ID | `$CLUSTER_ID` | `$CLUSTER_ID` (одинаковый) |
| Роль | Route Reflector | Route Reflector |

### 1.3 Отличия RR от HV

| Компонент | HV | RR |
|---|---|---|
| FRR (BGP/EVPN) | Да | Да |
| VXLAN | Да (data-plane) | **Нет** |
| VRF | Да (изоляция VPC) | **Нет** |
| Bridge | Да (L2-коммутация) | **Нет** |
| Docker / контейнеры | Да (PoC VM) | **Нет** |
| nftables (bridge) | Да (anti-spoofing) | **Нет** |
| nftables (inet) | Да (VXLAN firewall) | Да (BGP firewall) |
| Модули ядра (vrf, vxlan) | Да | **Нет** |

RR — чисто control-plane компонент. Минимальная VM: 1 vCPU, 512 MB RAM достаточно.

### 1.4 Что проверяем

- RR1 ↔ RR2 — BGP-сессия Established (обмен маршрутами между рефлекторами)
- RR1 ↔ HV1, RR1 ↔ HV2 — BGP-сессии Established (клиенты)
- RR2 ↔ HV1, RR2 ↔ HV2 — BGP-сессии Established (резервные)
- Маршруты от HV1 видны на HV2 через RR (type-2, type-3 EVPN)

---

## 2. Установка компонентов

Выполнить на **обоих RR**.

### 2.1 FRR (BGP/EVPN daemon)

```bash
curl -s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null
FRRVER="frr-stable"
echo "deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER" | \
  sudo tee /etc/apt/sources.list.d/frr.list

sudo apt update
sudo apt install -y frr frr-pythontools
```

### 2.2 Включить нужные демоны FRR

- **zebra** — взаимодействует с ядром (необходим для bgpd)
- **bgpd** — BGP-сессии, обмен EVPN маршрутами

```bash
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^zebra=no/zebra=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

### 2.3 Системные утилиты

```bash
sudo apt install -y iproute2 jq nftables
```

> **Не нужно** на RR: `linux-modules-extra`, `docker.io`, `bridge-utils`, модули ядра
> (vrf, vxlan, br_netfilter), sysctl для ip_forward и bridge-nf-call.
> RR не маршрутизирует и не коммутирует трафик.

### 2.4 Проверка готовности

```bash
echo "=== FRR ==="
sudo systemctl is-active frr && echo "OK" || echo "FRR не запущен"

echo "=== nftables ==="
sudo systemctl is-active nftables && echo "OK" || echo "nftables не активен"
```

---

## 3. FRR конфигурация

> **Важно**: конфигурируем через `vtysh`, а не через запись в `/etc/frr/frr.conf`.
> Команда `vtysh` + `write memory` гарантирует корректный формат.

### 3.1 BGP (на каждом RR)

Одна и та же конфигурация на обоих RR — `$LOCAL_IP`, `$PEER_RR_IP` и IP гипервизоров подставляются из переменных раздела 0.

```bash
sudo vtysh <<VTYSH
configure terminal
!
router bgp $BGP_ASN
 bgp router-id $LOCAL_IP
 !
 ! Cluster ID — одинаковый на обоих RR.
 ! Два RR с одним cluster-id = HA-пара: маршрут, полученный от HV1
 ! через RR1, не будет повторно отражён RR2 (loop prevention).
 bgp cluster-id $CLUSTER_ID
 !
 no bgp default ipv4-unicast
 no bgp ebgp-requires-policy
 !
 ! --- Peer RR (iBGP между рефлекторами) ---
 neighbor $PEER_RR_IP remote-as $BGP_ASN
 neighbor $PEER_RR_IP update-source $LOCAL_IP
 neighbor $PEER_RR_IP password $BGP_PASSWORD
 !
 ! --- HV clients ---
 neighbor $HV1_IP remote-as $BGP_ASN
 neighbor $HV1_IP update-source $LOCAL_IP
 neighbor $HV1_IP password $BGP_PASSWORD
 !
 neighbor $HV2_IP remote-as $BGP_ASN
 neighbor $HV2_IP update-source $LOCAL_IP
 neighbor $HV2_IP password $BGP_PASSWORD
 !
 address-family l2vpn evpn
  ! Peer RR — activate, но НЕ route-reflector-client
  neighbor $PEER_RR_IP activate
  !
  ! HV — route-reflector-client: RR отражает маршруты между клиентами.
  ! Без этой строки RR ведёт себя как обычный iBGP peer и не пересылает
  ! маршруты от одного клиента другому (iBGP split-horizon).
  neighbor $HV1_IP activate
  neighbor $HV1_IP route-reflector-client
  !
  neighbor $HV2_IP activate
  neighbor $HV2_IP route-reflector-client
 exit-address-family
exit
!
end
write memory
VTYSH
```

### 3.2 Добавление нового HV

Route Reflector должен знать о каждом HV-клиенте: без явного `neighbor` и `route-reflector-client` RR не установит BGP-сессию с новым HV и не будет отражать его маршруты остальным. Процедура выполняется на **обоих RR** — иначе новый HV потеряет HA (будет получать маршруты только от одного рефлектора).

При добавлении HV3 — выполнить на **обоих RR**:

```bash
NEW_HV_IP=$HV2_IP

sudo vtysh <<VTYSH
configure terminal
router bgp $BGP_ASN
 neighbor $NEW_HV_IP remote-as $BGP_ASN
 neighbor $NEW_HV_IP update-source $LOCAL_IP
 neighbor $NEW_HV_IP password $BGP_PASSWORD
 address-family l2vpn evpn
  neighbor $NEW_HV_IP activate
  neighbor $NEW_HV_IP route-reflector-client
 exit-address-family
exit
end
write memory
VTYSH
```

Также добавить `$NEW_HV_IP` в nftables `allowed_bgp_peers` (раздел 4).

### 3.3 Почему нет `advertise-all-vni` и VRF

На RR нет VXLAN-интерфейсов и VRF. Команда `advertise-all-vni` обнаруживает локальные VNI — на RR их нет. VRF-процессы (`router bgp 65000 vrf ...`) тоже не нужны — RR не участвует в data-plane, он только отражает маршруты.

### 3.4 Security: BGP MD5 и TTL security

Аналогично HV: MD5 auth включена, `ttl-security hops 1` не используется на облачных VM (см. HV-SETUP.md раздел 4.2).

---

## 4. nftables: BGP firewall

RR принимает только BGP-трафик (TCP:179). VXLAN UDP:4789 на RR **не нужен** — data-plane трафик идёт напрямую между HV.

**На каждом RR:**

```bash
sudo nft -f - <<NFT
table inet rr-firewall {
  set allowed_bgp_peers {
    type ipv4_addr
    elements = { $PEER_RR_IP, $HV1_IP, $HV2_IP }
  }

  chain input {
    type filter hook input priority 0; policy accept;

    # BGP TCP:179 — только от известных peer'ов (RR и HV)
    tcp dport 179 ip saddr @allowed_bgp_peers accept
    tcp dport 179 drop

    # Ответный BGP-трафик (source port 179) — только от известных peer'ов
    tcp sport 179 ip saddr @allowed_bgp_peers accept
    tcp sport 179 drop
  }
}
NFT
```

> При добавлении нового HV:
> `sudo nft add element inet rr-firewall allowed_bgp_peers { 10.123.0.24 }`

### 4.1 Persistence

```bash
sudo nft list table inet rr-firewall > /etc/nftables-rr-firewall.conf
echo 'include "/etc/nftables-rr-firewall.conf"' | sudo tee -a /etc/nftables.conf
```

---

## 5. Проверка

### 5.1 BGP сессии (на каждом RR)

```bash
sudo vtysh -c "show bgp summary"
```

Ожидаемый результат — все peer'ы Established:

```
Neighbor        V   AS   MsgRcvd MsgSent   Up/Down  State/PfxRcd
<PEER_RR_IP>    4   65000  ...     ...     00:05:xx  N
<HV1_IP>        4   65000  ...     ...     00:05:xx  N
<HV2_IP>        4   65000  ...     ...     00:05:xx  N
```

### 5.2 EVPN маршруты

```bash
sudo vtysh -c "show bgp l2vpn evpn"
```

RR должен видеть type-2 (MAC/IP) и type-3 (BUM) маршруты от каждого HV. Маршруты, полученные от HV1, должны быть отражены к HV2 и наоборот.

### 5.3 Route-reflector-client статус

```bash
sudo vtysh -c "show bgp neighbors $HV1_IP" | grep -i "route-reflector"
```

Ожидаемый вывод:

```
  Route-Reflector Client
```

### 5.4 Cluster ID

```bash
sudo vtysh -c "show bgp neighbors $HV1_IP" | grep -i "cluster"
```

### 5.5 Проверка с HV

На **любом HV** проверить, что маршруты приходят через RR:

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show evpn vni"
sudo vtysh -c "show bgp l2vpn evpn"
```

Должно быть: 2 BGP-сессии Established (к RR1 и RR2), EVPN маршруты от remote HV.

---

## 6. Устойчивость к перезагрузке

FRR конфиг сохраняется через `write memory`. nftables правила сохраняются в раздел 4.1. На RR нет сетевых интерфейсов (VRF/VXLAN/bridge), поэтому persistence-скрипт для интерфейсов **не нужен**.

Единственное, что нужно восстановить после reboot — nftables-таблица (через systemd nftables.service + include).

---

## 7. Диагностика

### 7.1 BGP сессии

```bash
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show running-config"
```

### 7.2 EVPN маршруты на RR

```bash
sudo vtysh -c "show bgp l2vpn evpn"
sudo vtysh -c "show bgp l2vpn evpn summary"
```

### 7.3 Детали peer'а

```bash
sudo vtysh -c "show bgp neighbors $HV1_IP"
```

### 7.4 Типичные проблемы

| Симптом | Причина | Решение |
|---|---|---|
| BGP сессия не Established | Нет L3-связности underlay | `ping $HV1_IP` с RR |
| BGP `Idle`, nc:179 OK | `ttl-security hops 1` на облачных VM | Убрать `ttl-security`; для iBGP TTL=255 по умолчанию |
| BGP `Connect`, SYN/ACK не проходит | TCP MD5 пароль не совпадает | Выровнять `password` на RR и HV |
| HV не получает маршруты от другого HV | Забыли `route-reflector-client` | Добавить `neighbor $HV_IP route-reflector-client` в `address-family l2vpn evpn` |
| HV получает маршруты от одного RR, но не от другого | Разный `cluster-id` на RR1 и RR2 | Выровнять `bgp cluster-id` — должен быть одинаковым |
| RR показывает маршруты, HV — нет | `neighbor ... activate` не выполнен в `address-family l2vpn evpn` | Проверить `show running-config` на RR |
| TCP:179 заблокирован | nftables `rr-firewall` не содержит IP HV | `nft add element inet rr-firewall allowed_bgp_peers { $HV_IP }` |
| Remote VTEPs = 0 на HV после перехода на RR | HV всё ещё пирятся напрямую, а не через RR | Обновить FRR на HV: заменить `neighbor $PEER_IP` на `neighbor $RR1_IP` + `neighbor $RR2_IP` |
| RR1 упал — маршруты есть | Штатная работа: HV получает маршруты от RR2 | Починить RR1, убедиться что BGP восстановился |
| Оба RR упали | Существующие маршруты закешированы в FRR на HV | Трафик идёт; не работает: новые HV, миграция VM |

### 7.5 Cleanup

```bash
sudo vtysh -c "configure terminal" -c "no router bgp $BGP_ASN" -c "end" -c "write memory"
sudo nft delete table inet rr-firewall 2>/dev/null
```
