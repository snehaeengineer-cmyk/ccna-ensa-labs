# CCNA ENSA Labs
![CCNA Progress](https://img.shields.io/badge/CCNA-ITN%20%E2%9C%93%20SRWE%20%E2%9C%93%20ENSA-blue)

![Cisco IOS](https://img.shields.io/badge/Cisco-IOS-blue)
![Routing](https://img.shields.io/badge/Routing-OSPF-success)
![Security](https://img.shields.io/badge/Security-ACLs-orange)
![WAN](https://img.shields.io/badge/WAN-GRE%20%26%20PPP-purple)
![Monitoring](https://img.shields.io/badge/Monitoring-SNMP%20%26%20Syslog-blue)

Hands-on routing, WAN, and network-management labs completed for **Enterprise Networking, Security, and Automation (ENSA)**, the second course in the CCNA curriculum (following SRWE).

All configurations were built and verified on physical Cisco gear (2960 switches, ISR4321 routers) in a lab environment, console-cable access via PuTTY, with traffic verification via Wireshark and SNMP MIB browser tooling where applicable.

## What's in here

| Lab | Topics | Devices |
|---|---|---|
| [Lab 1](./lab1-ospf-acl) | Single-area OSPFv2, OSPF metric tuning, OSPF authentication, passive interfaces, standard & extended ACLs | 3x ISR4321 routers, 2x switches |
| [Lab 2](./lab2-nat-ppp-gre) | Static & dynamic NAT/NAPT, PPP encapsulation with CHAP authentication, GRE VPN tunneling, OSPF over tunnel | 3x ISR4321 routers (WEST/ISP/EAST), 2x switches |
| [Lab 3](./lab3-ntp-syslog-snmp) | NTP master/client time sync, Syslog severity levels and remote logging, SNMP agent/manager configuration, MIB/OID traps | 2x ISR4321 routers, 1x switch |

Each lab folder contains:
- `README.md` — topology, addressing table, and a walkthrough of the configuration with reasoning
- `*.txt` config files — full device configuration sets, ready to paste into a CLI or Packet Tracer

## Core skills demonstrated

- Single-area OSPFv2: router ID assignment, wildcard masks, network advertisement, default-route redistribution, passive interfaces
- OSPF metric tuning: reference bandwidth adjustment, per-interface cost manipulation, path selection verification
- OSPF MD5 authentication between neighbors
- Standard and extended ACLs: numbered and named, source/destination/port filtering, placement strategy (close to source vs. close to destination), in-place ACL editing with sequence numbers
- NAT/NAPT: static one-to-one NAT, dynamic NAT with overload (port translation), NAT translation table inspection
- WAN encapsulation: HDLC vs. PPP, PPP authentication with CHAP (and why it beats PAP), debug-level inspection of PPP link negotiation phases
- GRE tunneling: tunnel interface configuration, routing across a tunnel (static and OSPF), traceroute behavior across an encapsulated path
- Network management protocols: NTP stratum hierarchy and client/master sync, Syslog severity levels and remote log collection, SNMP community strings, ACL-restricted SNMP access, MIB/OID structure and trap notifications

## Tools

Cisco IOS CLI (console access via PuTTY), Cisco Packet Tracer / physical lab equipment (2960-24TT switches, ISR4321 routers), Wireshark (SNMP packet inspection), iReasoning MIB Browser (SNMP manager), Tftpd32 (Syslog server).

---
*Coursework for the CCNA track (ENSA module), Cologne University of Applied Sciences — Prof. Dr. A. Grebe.*
