# Architecture Decision Records (ADRs)

This file documents key architectural decisions made in this design.

---

## ADR-001: ZPE Nodegrid as DCN Boundary Gateway

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Context** | Need a secure boundary between corporate LAN and DCN that supports NAT and out-of-band management |
| **Options** | (a) ZPE Nodegrid, (b) dedicated router, (c) direct L3 switch routing |
| **Decision** | ZPE Nodegrid |
| **Rationale** | Purpose-built OOB appliance with minimal attack surface; survives core network failure; provides serial console access to DCN gear |
| **Consequences** | Throughput limited vs dedicated router (acceptable for mgmt traffic); additional device to manage |

---

## ADR-002: DNAT + MASQUERADE vs Static Routes

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Context** | ALM management interface has no default gateway; needs bidirectional communication with remote users |
| **Options** | (a) DNAT + MASQUERADE on ZPE, (b) add default gateway to ALM, (c) static route + policy-based forwarding |
| **Decision** | DNAT + MASQUERADE on ZPE |
| **Rationale** | ALM is a monitoring appliance — modifying its routing table is unsupported. MASQUERADE guarantees symmetric routing without touching ALM config |
| **Consequences** | All ALM-originated traffic appears sourced from ZPE IP; connection tracking required |

---

## ADR-003: VPN Termination on Check Point (not ZPE)

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Context** | ZPE Nodegrid supports built-in IPsec/OpenVPN. Could terminate VPN on ZPE instead of Check Point |
| **Options** | (a) Terminate on Check Point L9, (b) terminate on ZPE |
| **Decision** | Terminate on Check Point L9 |
| **Rationale** | Centralised security policy; dedicated VPN licensing on the firewall; keeps ZPE CPU free for OOB functions; Check Point supports MFA out of the box |
| **Consequences** | VPN traffic must traverse two firewall layers; slight additional latency (acceptable) |

---

## ADR-004: Non-Standard Port 8443 for ALM GUI

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Context** | ALM GUI runs on standard HTTPS (443). Exposing 443 on the corporate LAN could conflict with other services |
| **Options** | (a) 8443 → DNAT to 443, (b) 443 with different VIP, (c) ALM on separate IP |
| **Decision** | 8443 external → DNAT to 443 internal |
| **Rationale** | eliminates port conflict without requiring additional IP addresses; port 8443 is commonly allowed in firewall policies |
| **Consequences** | Users must remember `:8443`; some corporate proxies may need explicit allow for non-standard ports |

---

## ADR-005: No Default Gateway on ALM

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Context** | ALM management network is isolated. Adding a default gateway could expose it to unintended traffic |
| **Options** | (a) No default gateway + ZPE MASQUERADE, (b) static default route to ZPE |
| **Decision** | No default gateway |
| **Rationale** | Zero-trust approach — ALM should not be able to initiate outbound connections. MASQUERADE handles return traffic |
| **Consequences** | ALM cannot initiate connections to syslog/NTP servers outside DCN (use ZPE as proxy if needed) |
