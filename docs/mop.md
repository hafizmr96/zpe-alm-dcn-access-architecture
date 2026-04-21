# Method of Procedure (MOP)

## Objective
Enable remote access to ALM GUI via ZPE through DCN network.

---

## Steps

1. Configure ALM management IP
2. Configure ZPE LAN interface
3. Verify connectivity (ping)
4. Configure NAT (DNAT + MASQUERADE)
5. Validate access via browser
6. Secure access via VPN or firewall rules

---

## Validation

- Ping from ZPE → ALM
- Access GUI via forwarded port
- Monitor logs for traffic flow

---

## Rollback

- Remove NAT rules
- Revert ALM IP if needed
