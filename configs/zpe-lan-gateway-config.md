# ZPE LAN Gateway Configuration

## Overview

The ZPE Nodegrid has two L3 interfaces that bridge the corporate LAN and DCN
networks. This document describes the interface configuration.

---

## Interface Layout

| Interface | Network | Purpose | VLAN |
|---|---|---|---|
| `eth0` | 192.168.70.7/24 | Corporate LAN connectivity | Access VLAN 70 |
| `eth1` | 10.10.10.1/24 | DCN management network | Access VLAN 10 |

Both interfaces are configured as **access ports** (untagged). In production
deployments, these may be changed to trunk ports with subinterfaces.

---

## eth0 — Corporate LAN

### Web UI Path

```
Network → Interfaces → eth0
```

### Parameters

| Field | Value |
|---|---|
| IPv4 Address | `192.168.70.7/24` |
| Default Gateway | `192.168.70.1` (corporate LAN gateway) |
| DNS Servers | Corporate DNS (e.g. 8.8.8.8 as secondary) |
| MTU | 1500 |
| Management Access | Enable SSH, HTTPS |

### Post-Config Verification

```bash
ip addr show eth0
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     inet 192.168.70.7/24 brd 192.168.70.255 scope global eth0

ip route show default
# default via 192.168.70.1 dev eth0

ping 192.168.70.1 -c 3   # gateway reachable
ping 8.8.8.8 -c 3        # internet reachable (if firewall permits)
```

---

## eth1 — DCN Network

### Web UI Path

```
Network → Interfaces → eth1
```

### Parameters

| Field | Value |
|---|---|
| IPv4 Address | `10.10.10.1/24` |
| Default Gateway | (none — leave blank) |
| MTU | 1500 |
| Management Access | Disable SSH, HTTPS (DCN is isolated) |

### Important

Do **not** set a default gateway on eth1. The corporate gateway on eth0
is the single default route. This guarantees:

- All internet-bound traffic from ZPE itself uses eth0
- DCN-bound traffic uses the directly connected route
- No asymmetric routing

### Post-Config Verification

```bash
ip addr show eth1
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     inet 10.10.10.1/24 brd 10.10.10.255 scope global eth1

ping 10.10.10.55 -c 3   # ALM reachable
```

---

## Routing Table (Expected)

```bash
ip route show
# default via 192.168.70.1 dev eth0
# 10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.1
# 192.168.70.0/24 dev eth0 proto kernel scope link src 192.168.70.7
```

---

## Persistence

ZPE Nodegrid saves network configuration automatically when applied through
the Web UI. If configuring via CLI, run after changes:

```bash
config-save
```

To reload saved config:

```bash
config-reload
```
