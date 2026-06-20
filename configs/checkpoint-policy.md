# Check Point Firewall Policy (Pseudocode)

## Overview

The remote access path traverses two Check Point security gateways in series.
This document describes the firewall policy rules that permit the ALM access
flow.

> **Note:** These rules are shown in pseudocode format for architecture
> reference. Actual rule syntax depends on the Check Point version and
> management console (e.g. R81.20 SmartConsole / CLI).

---

## Scope

Only rules relevant to the **ALM access flow** are listed. Pre-existing rules
(policy base, clean-up, logging) are omitted.

---

## Level 9 — WAN Edge Gateway

### Inbound (External → DMZ)

| # | Name | Source | Destination | Service | Action | Log |
|---|---|---|---|---|---|---|
| 10 | IPsec-IKE | Any | L9 Ext IP | UDP 500 (ISAKMP) | Accept | On |
| 11 | IPsec-NAT-T | Any | L9 Ext IP | UDP 4500 (IPsec NAT-T) | Accept | On |
| 12 | ESP | Any | L9 Ext IP | ESP (IP protocol 50) | Accept | On |
| 99 | Drop-All | Any | Any | Any | Drop | On |

### Internal (DMZ → Corporate LAN)

| # | Name | Source | Destination | Service | Action | Log |
|---|---|---|---|---|---|---|
| 20 | VPN-Pool-to-L7 | VPN_Pool | L7_Gateway_Int | Any | Accept | On |
| 21 | VPN-Pool-to-ZPE | VPN_Pool | 192.168.70.7 | HTTPS (8443) | Accept | On |
| 99 | Drop-All | Any | Any | Any | Drop | On |

### Objects

```
VPN_Pool:        10.0.0.0/8         (IPsec tunnel pool)
L9_Ext_IP:       a.b.c.d            (public IP of L9 gateway)
L7_Gateway_Int:  192.168.70.1       (L7's internal interface)
ZPE_Corp_IP:     192.168.70.7/32
```

---

## Level 7 — Corporate LAN Gateway

### Inbound (from Level 9)

| # | Name | Source | Destination | Service | Action | Log |
|---|---|---|---|---|---|---|
| 30 | VPN-to-ZPE | VPN_Pool | ZPE_Corp_IP | HTTPS (8443) | Accept | On |
| 31 | ZPE-Mgmt | ZPE_Corp_IP | Corp_DNS, Corp_NTP | DNS, NTP | Accept | On |
| 99 | Drop-All | Any | Any | Any | Drop | On |

### Outbound (to WAN / Internet) — ZPE management traffic

| # | Name | Source | Destination | Service | Action | Log |
|---|---|---|---|---|---|---|
| 40 | ZPE-to-Ext | ZPE_Corp_IP | Any | DNS, NTP, HTTPS | Accept | On |
| 99 | Drop-All | Any | Any | Any | Drop | On |

### Objects

```
ZPE_Corp_IP:     192.168.70.7/32
Corp_DNS:        <corporate DNS server IP>
Corp_NTP:        <corporate NTP server IP>
```

---

## Implicit Rules

- Traffic from DCN (`10.10.10.0/24`) is never routed through the firewalls
  — the ZPE handles NAT for DCN-originated flows
- ALM (`10.10.10.55`) is not directly reachable from any firewall-policy
  segment; it is only reachable via ZPE's DNAT
- All inter-zone traffic is logged, with a retention period of 90 days

---

## Rule Ordering

Rules are evaluated top-down with an **implicit drop** at the end. All
ALM-access rules are placed before the general Drop-All rules in each section.

---

## Audit Commands

```bash
# Show installed policy
fw stat
fw print

# Show active connections related to ALM flow
fw tab -t connections | grep 10.10.10.55

# Test rule match (simulate)
fw policy_verify
```
