# Northbridge Savings & Finance — Network Security Upgrade

**Module:** IE3122 Network Security
**Institution:** Sri Lanka Institute of Information Technology (SLIIT)
**Tool:** Cisco Packet Tracer

## Overview

Northbridge Savings & Finance is a small regional financial institution whose network had serious security gaps: no network segmentation, weak passwords, no multi-factor authentication, and poor access control. This project identifies realistic threats to the bank's network, designs a single layered security plan to address them, and implements and tests that plan in Cisco Packet Tracer.

Instead of four separate, unrelated fixes, the group designed one combined security architecture where each member's control contributes to an overall defense-in-depth strategy.

## Threat Model

Eight realistic threats were identified across the network, split two per group member. Each threat was analyzed using the asset → vulnerability → threat → exploit → attack → impact → likelihood/risk chain, with risk ratings justified specifically for Northbridge rather than generically.

Example threats covered include ARP spoofing, VLAN hopping, unauthorized access due to missing MFA, rogue DHCP servers, ransomware spreading through a flat network, and third-party payment partner risk.

## Security Controls

Seven layered security controls were designed and mapped against the eight threats, classified as Preventive, Detective, Deterrent, or Corrective:

- Port security (sticky MAC, violation shutdown)
- DHCP snooping
- Dynamic ARP Inspection (DAI)
- Trunk hardening (unused native VLAN, disabled DTP)
- VLAN segmentation with extended ACLs
- Zone-based firewall
- IPsec VPN with per-user AAA authentication

## Technology Selections

Each of the four group members evaluated and selected one technology area, with technical justification:

| Area | Technology chosen | Reason |
|---|---|---|
| AAA | Cisco ACS | Native integration with existing Cisco equipment; detailed per-command logging |
| IDS/IPS | Cisco IPS module | Plugs directly into existing equipment; no extra management overhead |
| VPN | Site-to-client IPsec VPN | Encrypts remote access traffic; per-user authentication via AAA |
| Network architecture | Dedicated firewall + internal segmentation | Deep traffic inspection needed for card payment processing; room to scale |

## Network Topology

The full network consists of 24 devices across head office, branch, and partner sites:

- **Routers:** ISP, R-HO (head office), R-BRANCH, PARTNER
- **Switches:** SW-CORE, SW-HO, SW-SERVER, SW-BRANCH, SW-DMZ, SW-INTERNET
- **Servers:** SRV-Core, SRV-AAA, SRV-Web, SRV-PayGW, SRV-Syslog
- **End devices:** PCs across head office, branch, management, and guest segments

## Security Configurations

### VLAN Segmentation + ACL (SW-CORE)

Blocks guest/branch traffic from reaching the CORE and STAFF networks:

```
ip access-list extended BLOCK_GUEST_TO_CORE
 deny ip 10.20.30.0 0.0.0.255 10.10.10.0 0.0.0.255
 deny ip 10.20.30.0 0.0.0.255 10.10.20.0 0.0.0.255
 permit ip any any
!
interface GigabitEthernet0/1
 ip access-group BLOCK_GUEST_TO_CORE in
```

Verified live: ping from the guest subnet to a core server failed (100% loss, destination unreachable), while the same ping from a staff PC succeeded.

### Zone-Based Firewall (R-HO)

Separates the network into INSIDE and OUTSIDE security zones and inspects traffic crossing between them:

```
class-map type inspect match-any INSIDE-TRAFFIC
 match protocol tcp
 match protocol udp
 match protocol icmp
!
policy-map type inspect INSIDE-TO-OUTSIDE
 class type inspect INSIDE-TRAFFIC
  inspect
 class type inspect class-default
  drop
!
zone security INSIDE
zone security OUTSIDE
zone-pair security IN-TO-OUT source INSIDE destination OUTSIDE
 service-policy type inspect INSIDE-TO-OUTSIDE
!
interface GigabitEthernet0/0
 zone-member security OUTSIDE
interface GigabitEthernet0/1
 zone-member security INSIDE
interface GigabitEthernet0/2
 zone-member security INSIDE
```

### IPsec VPN with AAA (R-HO)

Encrypts remote access traffic and authenticates each VPN user individually rather than relying on a single shared credential:

```
aaa new-model
aaa authentication login VPN-AUTH local
aaa authorization network VPN-GROUP local
!
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
 group 5
!
crypto isakmp client configuration group NB-REMOTE
 key NBgroupKey123
 pool VPN-POOL
!
crypto ipsec transform-set VPN-TS esp-3des esp-sha-hmac
crypto dynamic-map VPN-DYN 10
 set transform-set VPN-TS
 reverse-route
!
crypto map VPN-MAP client authentication list VPN-AUTH
crypto map VPN-MAP isakmp authorization list VPN-GROUP
crypto map VPN-MAP client configuration address respond
crypto map VPN-MAP 10 ipsec-isakmp dynamic VPN-DYN
!
ip local pool VPN-POOL 10.10.50.1 10.10.50.20
!
interface GigabitEthernet0/0
 crypto map VPN-MAP
```

## Verification

All configurations were tested live in Packet Tracer rather than only written on paper:

- `show access-lists` — confirms ACL rules and live hit counters
- `show zone security` / `show zone-pair security` — confirms firewall zone assignment and policy
- `show crypto isakmp sa` / `show crypto ipsec sa` — confirms active VPN tunnel state
- Live ping tests contrasting blocked (guest) and allowed (staff) traffic

## Known Limitation

Cisco Packet Tracer does not correctly re-inspect traffic through the zone-based firewall after it has been decrypted off an IPsec VPN tunnel terminating on the same physical interface. This was confirmed through testing (`show crypto ipsec sa` showed successful encryption/decryption counts with zero matching firewall zone-pair sessions) and a real-Cisco VTI workaround was attempted and found unsupported in this simulator. This is documented here as a genuine platform limitation rather than worked around with an unverifiable claim of success. The VPN path is instead secured by per-user AAA authentication and the ACL, independent of the firewall zone.

## Screenshots

All evidence screenshots from configuration and verification are in the [`screenshots/`](./screenshots) folder, covering device configuration, `show` command verification, and live ping tests for each member's security control.

## Group Members

| Member | Role | Technology | Threats covered |
|---|---|---|---|
| Member A | AAA | Cisco ACS | Threats 1–2 |
| Member B | IDS/IPS | Cisco IPS module | Threats 3–4 |
| Member C | VPN | Site-to-client IPsec VPN | Threats 5–6 |
| Member D | ACL + Firewall + Architecture | Zone-based firewall, segmentation | Threats 7–8 |
