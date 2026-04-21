# ZPE to ALM DCN Access Architecture

## Overview
This project demonstrates a real-world deployment scenario where a ZPE (Nodegrid) device is used as an out-of-band access gateway to reach an ALM (Adtran) system for centralized fiber monitoring.

The design enables secure remote access from external locations (e.g. Malaysia) into HQ infrastructure via IPsec VPN, traversing multiple security layers before reaching the ALM GUI.

---

## Architecture Diagram

![Architecture](High level Architecture.png)

---

## Key Components

- Remote Engineer (Malaysia)
- IPsec VPN Tunnel (AES-256)
- Check Point Security Gateway (Level 9)
- Check Point Security Gateway (Level 7)
- ZPE Nodegrid (Out-of-Band Management)
- ALM (Adtran) – Fiber Monitoring System

---

## Access Flow

1. Remote user connects via IPsec VPN to HQ
2. Traffic enters through Level 9 Checkpoint firewall
3. Routed internally to Level 7 network
4. Passed through Checkpoint L7
5. Reaches ZPE
6. ZPE performs NAT / routing into DCN network
7. User accesses ALM GUI

---

## Key Design Concepts

- Separation of WAN and DCN network
- ZPE acting as boundary gateway
- NAT (DNAT + MASQUERADE) for GUI exposure
- Secure layered firewall architecture
- Out-of-band management integration

---

## Example NAT Configuration (ZPE)
PREROUTING DNAT:
192.168.70.7:8443 → 10.10.10.55:443

POSTROUTING MASQUERADE:
10.10.10.0/24 → eth0


---

## Security Considerations

- Avoid exposing ALM directly to public internet
- Prefer VPN or controlled NAT access
- Implement ACLs on firewall
- Segment DCN network from corporate LAN

---

## Disclaimer

- This architecture diagram was generated using AI tools.
- The design and technical flow are fully conceptualized and engineered by the author.
- All IP addresses shown are **for illustration purposes only** and do NOT reflect any real production environment.

---

## Author

Hafiz – Network / DevOps Engineer  
Focus: Telecom Infrastructure, DCN, Automation, AI-assisted operations

---

## Purpose

This repository is intended for:
- Portfolio demonstration
- Architecture reference
- Knowledge sharing
