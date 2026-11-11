# Network security

Controls that protect networks and traffic: TLS encryption, firewalls, DMZ designs, VPNs and zero trust access.

## Map

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["SSL and TLS"]
  n1["Firewalls"]
  n0 --> n1
  n2["DMZ"]
  n1 --> n2
  n3["VPN"]
  n2 --> n3
  n4["ZTNA and zero trust"]
  n3 --> n4
```

## Notes in reading order

1. [SSL and TLS](ssl-and-tls.md): versions, handshake, cryptography and certificates.
2. [Firewalls](firewalls.md): types, policies, features and host firewalls.
3. [DMZ](dmz.md): architectures, mechanisms and attack mitigations.
4. [VPN](vpn.md): types, protocols, encryption and leaks.
5. [ZTNA and zero trust](ztna.md): microsegmentation and identity-based access.

## Planned topics

- Intrusion detection and prevention (Suricata, Zeek)
- DDoS protection
- Network access control and 802.1X
- Public key infrastructure for internal CAs
