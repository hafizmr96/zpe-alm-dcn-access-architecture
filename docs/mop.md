# Method of Procedure (MOP)

**Objective:** Enable remote access to ALM GUI via ZPE Nodegrid through the DCN network.

**Scope:** First-time deployment. Assumes ZPE is physically installed and
network cabling is complete.

**Risk:** Low — ALM is not disrupted during configuration; only access path changes.

---

## Pre-Requisites

- [ ] ZPE Nodegrid appliance — powered, network cables connected (eth0 → corporate LAN, eth1 → DCN switch)
- [ ] ALM (Adtran) — powered, configured with management IP `10.10.10.55/24`
- [ ] Laptop with console cable or SSH access to ZPE
- [ ] L3 connectivity from ZPE eth0 to corporate LAN gateway
- [ ] IPsec VPN profile (for remote testing)
- [ ] Change request approved

---

## Procedure

### Step 1 — Configure ALM Management IP

```bash
# On ALM console or web GUI
# Set management interface
ip address 10.10.10.55/24
# No default gateway needed — traffic returns via ZPE MASQUERADE
```

**Verify:**

```bash
ping 10.10.10.1    # ALM → ZPE eth1 (DCN side)
# Expected: replies received
```

### Step 2 — Configure ZPE LAN Interface (eth0)

```
From ZPE Web GUI or CLI:
Network → Interfaces → eth0
  IP Address:    192.168.70.7/24
  Default GW:    <corporate gateway IP>
  DNS:           <corporate DNS>
```

**Verify:**

```bash
ping 192.168.70.1   # ZPE → corporate gateway (adjust GW IP)
ping 8.8.8.8        # ZPE → internet (if permitted by firewall)
```

### Step 3 — Configure ZPE DCN Interface (eth1)

```
Network → Interfaces → eth1
  IP Address:    10.10.10.1/24
  Do NOT set a default gateway on this interface
```

**Verify:**

```bash
ping 10.10.10.55   # ZPE → ALM across DCN network
# Expected: replies received
```

### Step 4 — Configure DNAT (Port Forwarding)

```
Network → NAT → DNAT Rules → Add Rule

  Rule Name:      ALM-GUI
  Inbound Intf:   eth0
  Protocol:       TCP
  Destination IP: 192.168.70.7
  Dest. Port:     8443
  Forward To:     10.10.10.55
  Forward Port:   443
```

Equivalent iptables command (verification):

```bash
iptables -t nat -L PREROUTING
# Look for DNAT rule matching 192.168.70.7:8443 → 10.10.10.55:443
```

### Step 5 — Configure MASQUERADE (Source NAT)

```
Network → NAT → MASQUERADE Rules → Add Rule

  Source Network: 10.10.10.0/24
  Outbound Intf:  eth0
```

Equivalent iptables:

```bash
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
```

**Why:** ALM has no default gateway. Without MASQUERADE, return packets from
ALM would be sent to the remote user's IP directly and dropped. MASQUERADE
rewrites the source to ZPE's eth0 IP, guaranteeing symmetric routing.

### Step 6 — Validate Access

**Local test (from corporate LAN):**

```bash
curl -k https://192.168.70.7:8443
# Expected: ALM login page HTML
```

**Remote test (via IPsec VPN):**

1. Connect IPsec VPN client
2. Open browser → `https://192.168.70.7:8443`
3. ALM login page should load

### Step 7 — Lock Down

Ensure Check Point firewall rules only permit:

- IPsec UDP 500/4500 to L9
- HTTPS traffic from VPN pool to `192.168.70.7:8443` through L7
- Block all other DCN-bound traffic

---

## Validation Checklist

| Check | Expected | Result |
|---|---|---|
| ZPE → ALM ping | Replies received | |
| curl to ALM via DNAT | HTTP 200 / login page | |
| GUI access via VPN | Page loads | |
| MASQUERADE working | src=192.168.70.7 in ALM logs | |
| Firewall logs | No denied traffic for ALM flow | |

---

## Rollback Plan

```bash
# Remove DNAT rule
iptables -t nat -D PREROUTING -d 192.168.70.7 -p tcp --dport 8443 -j DNAT --to-destination 10.10.10.55:443

# Remove MASQUERADE rule
iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE

# Or via ZPE GUI: disable the NAT rules and save
```

Post-rollback validation:

- ALM GUI is not accessible from corporate LAN or VPN
- ZPE → ALM ping still works (management connectivity preserved)
- No stale NAT connections remain (flush conntrack if needed)
