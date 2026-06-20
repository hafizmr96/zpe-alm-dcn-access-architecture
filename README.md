# ZPE Nodegrid — Out-of-Band DCN Access Architecture

[![IPsec](https://img.shields.io/badge/VPN-IPsec_AES256-003E7E?logo=checkpoint)](https://www.checkpoint.com/)
[![ZPE](https://img.shields.io/badge/OOB-ZPE_Nodegrid-00A86B)](https://www.zpesystems.com/)
[![OS](https://img.shields.io/badge/OS-Linux_CentOS-262577?logo=linux)](https://www.centos.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A production-grade **out-of-band access architecture** that enables secure remote
access to an ALM (Adtran) fiber monitoring system inside a DCN network — through
multi-layer firewalls, IPsec VPN, and ZPE Nodegrid as the boundary gateway.

---

## Problem

Field engineers and NOC teams in a different country (Malaysia) need to access
the ALM fiber monitoring GUI at HQ. The ALM lives inside a closed **DCN
(Datacenter Network)** segment with no direct internet exposure. Direct
connectivity is blocked by:

- Multiple firewall zones (Level 9 WAN → Level 7 corporate LAN → DCN)
- No route from corporate LAN into the DCN management plane
- Security policy requiring all management traffic to traverse an
  **out-of-band (OOB) management gateway**

---

## Solution

Deploy a **ZPE Nodegrid** device as the OOB boundary gateway between the
corporate LAN and the DCN network. Remote users connect via IPsec VPN through
two Check Point firewall layers, then the ZPE performs DNAT + MASQUERADE to
expose the ALM GUI on a controlled port.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Remote Engineer (Malaysia)                                             │
│  ┌──────────────┐                                                       │
│  │  VPN Client  │  IPsec AES-256                                        │
│  └──────┬───────┘                                                       │
│         │                                                               │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │  Check Point L9 (WAN Edge)         Zone: External → DMZ         │   │
│  │  ─ Allow IPsec UDP 500/4500                                      │   │
│  │  ─ Allow HTTPS to ZPE WAN IP:8443                                │   │
│  └──────┬──────────────────────────────────────────────────────────┘   │
│         │                                                               │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │  Check Point L7 (Corporate LAN)    Zone: DMZ → Internal         │   │
│  │  ─ Permit established VPN traffic                                │   │
│  │  ─ Allow ZPE-bound sessions                                      │   │
│  └──────┬──────────────────────────────────────────────────────────┘   │
│         │                                                               │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │  ZPE Nodegrid                     OOB Management Gateway        │   │
│  │  ─ eth0: 192.168.70.7/24         (corporate LAN side)          │   │
│  │  ─ eth1: 10.10.10.1/24           (DCN side)                    │   │
│  │                                                                  │   │
│  │  DNAT:  192.168.70.7:8443  →  10.10.10.55:443                   │   │
│  │  MASQ:  10.10.10.0/24      →  eth0                              │   │
│  └──────┬──────────────────────────────────────────────────────────┘   │
│         │                                                               │
│  ┌──────▼──────────────────────────────────────────────────────────┐   │
│  │  ALM (Adtran)                     Fiber Monitoring System       │   │
│  │  ─ MGMT IP: 10.10.10.55/24                                      │   │
│  │  ─ GUI: HTTPS :443                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Traffic Flow

```
Remote User                  CP L9              CP L7              ZPE               ALM
    │                          │                  │                  │                │
    │  IPsec VPN (AES-256)     │                  │                  │                │
    ├─────────────────────────►│                  │                  │                │
    │                          │  Decrypt & route │                  │                │
    │                          ├─────────────────►│                  │                │
    │                          │                  │  Forward to ZPE  │                │
    │                          │                  ├─────────────────►│                │
    │                          │                  │                  │ DNAT:8443→443  │
    │                          │                  │                  ├───────────────►│
    │                          │                  │                  │  HTTPS :443    │
    │                          │                  │◄─────────────────┤                │
    │                          │◄─────────────────┤                  │                │
    │◄─────────────────────────┤                  │                  │                │
    │  GUI reaches user        │                  │                  │                │
```

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| OOB Gateway | ZPE Nodegrid | Purpose-built OOB appliance; Linux-based; survives core network failure |
| VPN termination | Check Point L9 (WAN edge) | Industry-standard security gateway; IPSec termination at perimeter |
| NAT type | DNAT + MASQUERADE (ZPE) | ALM has no default gateway; ZPE handles bidirectional translation |
| DCN addressing | RFC 1918 /24 per site | Predictable mgmt addressing; no overlap with corporate space |
| GUI access port | 8443 (ZPE) → 443 (ALM) | Non-standard port on corporate side to avoid conflict; DNAT rewrites to ALM native |

---

## Repository Structure

```
zpe-alm-dcn-access-architecture/
├── README.md                    # This file — architecture overview
├── ARCHITECTURE.md              # Architecture Decision Records (ADRs)
├── docs/
│   ├── architecture-deep-dive.md  # Detailed technical breakdown
│   ├── MOP.md                     # Method of Procedure
│   ├── images/                    # Architecture diagrams & screenshots
│   │   ├── High Level Architecture.png
│   │   ├── ALM GUI.jpeg
│   │   ├── DNAT Configuration in ZPE.jpeg
│   │   └── LAN Gateway configuration in ZPE.jpeg
│   └── diagram/
│       └── source-notes.md        # Styling source
├── configs/
│   ├── zpe-dnat-config.md         # DNAT + MASQUERADE config reference
│   ├── zpe-lan-gateway-config.md  # LAN interface config reference
│   └── checkpoint-policy.md       # Firewall policy pseudocode
├── .gitignore
└── LICENSE
```

---

## Technologies

| Layer | Technology | Role |
|---|---|---|
| WAN connectivity | IPsec VPN (AES-256) | Encrypted tunnel from remote to HQ |
| Security gateway | Check Point (L9 / L7) | Multi-zone firewall, traffic inspection |
| OOB management | ZPE Nodegrid (CentOS-based) | Boundary gateway, NAT, access control |
| Fiber monitoring | ALM (Adtran) | Passive optical network monitoring |
| DCN | Switched Ethernet / RFC 1918 | Isolated management plane |

---

## Security Posture

- **Defence in depth**: 3 security zones (WAN, corporate LAN, DCN)
- **Least privilege**: ALM accessible only via ZPE on a single port (8443)
- **No direct internet**: DCN has no default gateway to WAN
- **OOB isolation**: ZPE on separate management VLAN; does not transit production data
- **Encryption in transit**: IPsec AES-256 from remote to HQ perimeter

---

## Author

**Hafiz** — Network / Platform Engineer  
Telecom Infrastructure · DCN · Automation · AI-assisted Operations

---

## License

MIT
