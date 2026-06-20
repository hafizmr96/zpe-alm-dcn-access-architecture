# Architecture Deep Dive

## Network Topology

### Layer 1: WAN Edge (Check Point L9)

The outermost security layer. Termination point for all remote-access IPsec VPN
tunnels. Only three services are permitted inbound:

| Service | Port | Purpose |
|---|---|---|
| IPsec ISAKMP | UDP 500 | IKE key exchange |
| IPsec NAT-T | UDP 4500 | NAT traversal for VPN clients |
| HTTPS | TCP 8443 | ZPE GUI access (conditional, typically VPN-only) |

All traffic passing L9 is decrypted, inspected, and forwarded to the internal
routing domain (Level 7).

### Layer 2: Corporate LAN (Check Point L7)

The internal security layer. Separates the corporate IT network from the DCN
management plane. Policy:

- Permit decrypted VPN traffic sourced from L9 → ZPE IP
- Block all direct traffic to DCN subnets
- Allow DNS, NTP, and syslog from ZPE to corporate servers

### Layer 3: DCN Boundary (ZPE Nodegrid)

The ZPE Nodegrid bridges between the corporate LAN (eth0) and the DCN management
segment (eth1). It is the **only** gateway between these two domains.

| Interface | Network | Role |
|---|---|---|
| eth0 | 192.168.70.0/24 | Corporate LAN side — default gateway to upstream |
| eth1 | 10.10.10.0/24 | DCN side — management network for ALM and other elements |

### Layer 4: DCN (ALM + Managed Infrastructure)

The innermost network. Contains the ALM (Adtran) fiber monitoring system and
other DCN-managed elements. These devices have:

- No default gateway to the corporate LAN
- Management access only via the ZPE
- RFC 1918 addressing, one /24 per site

---

## NAT Flow (Detailed)

### DNAT (Destination NAT) — Forwarding GUI access

```
Rule: PREROUTING DNAT
Match:  dst=192.168.70.7  dport=8443  proto=tcp
Action: DNAT → 10.10.10.55:443
```

When a remote user hits `https://192.168.70.7:8443`:

1. ZPE receives the packet on eth0
2. PREROUTING DNAT rewrites destination to `10.10.10.55:443`
3. Route lookup sends the packet out eth1 (DCN)
4. ALM receives a standard HTTPS request on port 443

### MASQUERADE (Source NAT) — Return traffic

```
Rule: POSTROUTING MASQUERADE
Match:  src=10.10.10.0/24  oif=eth0
Action: src NAT to eth0 IP (192.168.70.7)
```

Because the ALM's default gateway does not point to ZPE, return packets from
ALM would be dropped. MASQUERADE rewrites the source IP to ZPE's eth0 address,
ensuring:

1. ALM sends response to `192.168.70.7` (matching its directly connected route)
2. ZPE receives, unwraps NAT, forwards to the original remote user

Without MASQUERADE, the ALM would try to send replies directly to the remote
user's IP — which it has no route for.

---

## Why ZPE Nodegrid?

| Requirement | ZPE Capability |
|---|---|
| Out-of-band access | CentOS-based appliance with serial + ethernet mgmt ports |
| Multi-homing | Dual NIC — physically separates WAN/LAN from DCN |
| NAT | Full iptables — DNAT, MASQUERADE, port forwarding |
| Remote power control | Optional PDU integration for power-cycling remote gear |
| VPN termination | Built-in OpenVPN / IPsec (though we terminate at Check Point) |
| Logging | Local syslog, remote syslog forwarding |

ZPE is not a router. It is an **OOB management appliance** that happens to do
routing between its interfaces as a secondary function. This is deliberate:
the DCN network has no dedicated router, and the ZPE's minimal attack surface
makes it a safer gateway than a full router.

---

## IPsec VPN Design

| Parameter | Value |
|---|---|
| Protocol | IKEv2 (preferred) / IKEv1 |
| Encryption | AES-256-GCM |
| Integrity | SHA-256 |
| DH Group | 14 (2048-bit MODP) |
| Perfect Forward Secrecy | Enabled |
| NAT Traversal | Enabled (clients behind CGNAT) |
| Authentication | Certificate + username/password (MFA-ready) |

The VPN terminates at **Check Point L9**, not on the ZPE. This keeps:

- VPN licensing on the dedicated security appliance
- ZPE CPU free for its primary OOB function
- Security policy centralized in the Check Point management console

---

## Security Zone Mapping

| Zone | Network Segment | Trust Level | Internet Access |
|---|---|---|---|
| External / WAN | Internet | None | N/A |
| DMZ (L9 outbound) | IPsec tunnel pool | Low | No |
| Corporate LAN (L7) | 192.168.70.0/24 | Medium | Yes (restricted) |
| DCN | 10.10.10.0/24 | High | No |
| ALM | 10.10.10.55/32 | Critical | No |

Access is permitted strictly from lower trust → higher trust only via
explicit rules (VPN → CP L9 → CP L7 → ZPE → ALM).

---

## Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| ZPE power loss | ALM inaccessible remotely | OOB serial console + PDU auto-reboot |
| IPsec tunnel drop | No remote access | Auto-reconnect configured on client; DPD enabled |
| ALM unreachable | GUI down | ZPE alerting via ping monitor |
| eth0 link down | No corporate connectivity | Dual-homed ZPE (bonding) in production |
