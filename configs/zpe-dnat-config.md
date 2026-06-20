# DNAT + MASQUERADE Configuration Reference

## Overview

The ZPE Nodegrid uses two complementary NAT rules to bridge between the
corporate LAN and the isolated DCN network.

---

## DNAT Rule — ALM GUI Forwarding

### Purpose

Forward incoming HTTPS connections on the corporate-facing interface to the ALM
inside the DCN.

### ZPE Web UI Path

```
Network → NAT → DNAT Rules → Add Rule
```

### Parameters

| Field | Value | Notes |
|---|---|---|
| Rule Name | `ALM-GUI-DNAT` | Must be unique |
| Inbound Interface | `eth0` | Corporate LAN side |
| Protocol | `TCP` | HTTPS runs over TCP |
| Destination IP | `192.168.70.7` | ZPE's corporate LAN IP |
| Destination Port | `8443` | Non-standard port to avoid conflicts |
| Forward To IP | `10.10.10.55` | ALM management IP |
| Forward To Port | `443` | ALM native HTTPS port |
| State | Enabled | |

### iptables Equivalent

```bash
iptables -t nat -A PREROUTING \
  -i eth0 \
  -d 192.168.70.7 \
  -p tcp --dport 8443 \
  -j DNAT --to-destination 10.10.10.55:443
```

### Verification

```bash
iptables -t nat -L PREROUTING -n -v
# Chain PREROUTING (policy ACCEPT)
#  pkts bytes target     prot opt in     out     source    destination
#     0     0 DNAT       tcp  --  eth0   *       0.0.0.0/0 192.168.70.7
#                          tcp dpt:8443 to:10.10.10.55:443
```

---

## MASQUERADE Rule — DCN Return Traffic

### Purpose

Rewrite the source IP of packets originating from the DCN network so that ALM
(and other DCN devices without a default gateway) receive return traffic
correctly.

### ZPE Web UI Path

```
Network → NAT → MASQUERADE Rules → Add Rule
```

### Parameters

| Field | Value | Notes |
|---|---|---|
| Rule Name | `DCN-MASQ` | Must be unique |
| Source Network | `10.10.10.0/24` | Entire DCN management segment |
| Outbound Interface | `eth0` | Exit to corporate LAN |
| State | Enabled | |

### iptables Equivalent

```bash
iptables -t nat -A POSTROUTING \
  -s 10.10.10.0/24 \
  -o eth0 \
  -j MASQUERADE
```

### Verification

```bash
iptables -t nat -L POSTROUTING -n -v
# Chain POSTROUTING (policy ACCEPT)
#  pkts bytes target     prot opt in     out     source         destination
#     0     0 MASQUERADE  all  --  *     eth0   10.10.10.0/24  0.0.0.0/0
```

---

## Packet Walk

```
Packet from Remote User > CP L9 > CP L7 > ZPE eth0 :8443

1. PREROUTING DNAT:  dst=192.168.70.7:8443 → 10.10.10.55:443
2. Route lookup → eth1 (DCN)
3. ALM receives packet src=<remote_user_ip> dst=10.10.10.55:443
4. ALM replies dst=<remote_user_ip> src=10.10.10.55:443
   → sent to default gateway (NONE — ALM has no default route)
   → or sent to 10.10.10.1 (ZPE eth1) if directly connected

5. ZPE receives ALM reply on eth1
6. FORWARD chain checks state — packet is part of established DNAT session
7. POSTROUTING MASQUERADE:  src=10.10.10.55 → 192.168.70.7
8. Packet leaves ZPE eth0, traverses firewalls, reaches remote user
```

---

## Troubleshooting

| Symptom | Likely Cause | Check |
|---|---|---|
| Connection timeout | Firewall blocking 8443 | Check Point logs for drops |
| Connection refused | ALM service not running | `curl https://10.10.10.55:443` from ZPE CLI |
| GUI loads but no data | MASQUERADE missing | Verify `iptables -t nat -L POSTROUTING` |
| Only works locally | VPN route missing | Ensure corporate LAN subnet is in VPN ACL |
| DNAT not matching | IP/port mismatch | Double-check DNAT rule parameters |
