# Lab 3 — NTP, Syslog & SNMP Network Management

## Objective

Establish a network-wide time source via NTP, configure remote Syslog collection with severity-level control, and stand up SNMP for read-only polling and trap notifications — the three classic out-of-band network management protocols.

```mermaid
flowchart LR
    %% Define Devices
    PCA["🖥️ PC-A <br> 192.168.1.0/24"]
    R1(["🌐 R1 <br> (OSPF Area 0)"])
    R2(["🌐 R2"])
    SRV["🖧 Server0 <br> (Syslog & SNMP Host)"]

    %% Core Links
    PCA -- "G0/0/1" --- R1
    R1 -- "G0/0/0 <---> G0/0/0" --- R2
    R2 -- "G0/0/1" --- SRV
```

R1 and R2 connect over a point-to-point Ethernet link, each fronting its own LAN. PC-A acts as the SNMP manager (running iReasoning MIB Browser); Server0 acts as both the Syslog server (Tftpd32) and a generic LAN endpoint.

## Addressing Table

| Device | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| R1 | G0/0/1 | 192.168.1.1 | 255.255.255.0 |
| | G0/0/0 | 10.10.1.1 | 255.255.255.252 |
| R2 | G0/0/1 | 192.168.3.1 | 255.255.255.0 |
| | G0/0/0 | 10.10.1.2 | 255.255.255.252 |
| PC-A | NIC | 192.168.1.3 | 255.255.255.0 |
| Server0 | NIC | 192.168.3.3 | 255.255.255.0 |

OSPF process 1, area 0, connected networks only.

## Approach

### Part 1 — NTP (Network Time Protocol)

**Why time sync isn't a nice-to-have for a network admin.** The lab frames this directly: Syslog and debug output are only useful for diagnosing what happened, in what order, across multiple devices, if every device's clock agrees. A log entry timestamped 14:32:01 on R1 and 14:31:58 on R2 for what was actually the same event makes correlating cause and effect across devices needlessly hard — or actively misleading — during an actual incident.

**Stratum is a distance metric, not a quality rating in itself.** NTP stratum indicates how many hops a device is from an authoritative reference clock (stratum 0 — typically a GPS receiver or atomic clock source). A stratum 1 server is directly synced to that reference; stratum 2 syncs from a stratum 1 server; and so on. In this lab, R2 was deliberately configured as `ntp master 4` — a self-declared stratum 4 master, since there's no real external time source available in the lab environment, and R1 then becomes a stratum 5 NTP *client* of R2. The stratum number itself doesn't measure accuracy directly, but it does indicate how many potential points of drift/error sit between a device and the original authoritative source.

**Master/client roles, configured asymmetrically on purpose.** R2 gets `ntp master 4` (declares itself an authoritative time source at stratum 4); R1 gets `ntp server <R2's IP>` (points explicitly at R2 as its time source). This isn't a peer relationship — it's a deliberate hierarchy, which is the normal pattern for a small network: one or two routers act as the internal time reference, and everything else syncs from them rather than each device trying to independently source time from the wider internet.

### Part 2 — Syslog

**Severity levels run from most to least severe, numerically ascending.** Severity 0 (Emergency) is the most critical; severity 7 (Debugging) is the least. This numbering is easy to get backwards intuitively — a *lower* number means a *more severe* event, the opposite of how many people would guess "level 7" sounds more serious than "level 0."

**Setting the trap level controls what gets sent to the remote server, not what's logged locally.** `logging trap notifications` (or equivalently `logging trap 5`) sets the *threshold* — any message at that severity or more severe gets forwarded to the configured Syslog host; less severe messages stay local (console/buffer) only. This is the practical lever for noise control: set the trap level too low (numerically too high, i.e. too permissive) and the Syslog server gets flooded with routine informational chatter; set it too restrictive and genuinely important events (like a flapping link or a security-relevant access list hit) might never make it off the device.

**Local Syslog destination, before remote forwarding is even configured.** Network devices and most operating systems log locally first by default — to a console buffer, an internal memory buffer, or a local file — regardless of whether a remote Syslog server is configured at all. Remote forwarding (`logging host <ip>`) is an addition on top of that local logging, not a replacement for it.

**UDP, not TCP — and the implication that follows.** Syslog uses UDP port 514 by default. Because UDP doesn't guarantee delivery, there's no acknowledgment that a Syslog message ever actually arrived at the server — which is part of why correlating timestamps from NTP-synced devices matters even more: if a message is dropped in transit, the surrounding messages (with consistent timestamps) are often the only way to reconstruct what happened.

**Observed directly: link-state changes propagate to Syslog faster than routing-protocol state changes.** Bringing an interface down and back up on R2 produced an immediate local link-state Syslog message, while the corresponding OSPF adjacency change message appeared with a visible delay — a direct, observed illustration of the layering between Layer 1/2 link detection (essentially instantaneous) and Layer 3 routing protocol convergence (which needs to run through Hello/Dead timers and re-establish adjacency before it has anything to report).

### Part 3 — SNMP

**Read-only by default, write access scoped tightly when present.** R1's SNMP agent configuration grants read-only (`RO`) access to the community string `public` for general polling, while write access (`RW`) requires the separate `admin` community string and — critically — is restricted to a single source host via a standard ACL. This mirrors a real production stance: broad read access for monitoring tools is low-risk and commonly necessary, but the ability to *change* device configuration via SNMP write should be locked down to as few authorized management stations as technically possible.

**An ACL on the community string, not just the string itself.** The community string alone is a shared secret, but in SNMPv1/v2c it travels across the network in clear text — meaning the string alone is a weak control if the network itself isn't otherwise trusted. Binding the community string to a named/standard ACL that only permits one specific host adds a second, independent control: even a captured community string is useless to an attacker whose source IP doesn't match the ACL.

**Traps exist because polling alone misses the moment something happens.** SNMP GET lets a manager pull the current state of a Managed Object on demand, but that only reflects whatever the value happens to be *at the moment you ask*. A trap is the device proactively pushing a notification the instant a defined event occurs (a link transitioning to up, a configuration change, a threshold breach) — without traps, a manager would have to poll constantly and still might miss a transient event that resolved itself between polling intervals.

**Confirmed in Wireshark: SNMPv2c is not encrypted.** A packet capture during a live SNMP GET request showed the community string in cleartext, directly readable in the captured packet — concrete, hands-on confirmation of why SNMPv2c's "security" is really just an unencrypted shared secret, and part of the motivation for SNMPv3, which adds genuine authentication and encryption on top of the same basic protocol mechanics.

**OID structure is a tree, and the numeric path tells you exactly where an object lives.** Every Managed Object has both an alphanumeric path (`iso.org.dod.internet.mgmt.mib-2.ip.ipDefaultTTL`) and an equivalent fully-numeric OID (`1.3.6.1.2.1.4.2`). The numeric form is what actually travels in SNMP protocol messages — the alphanumeric form exists purely for human readability — and tools like the MIB browser or oid-base.com exist specifically to translate between the two without requiring memorization of the entire MIB tree.

## Verification & key findings

- `show ntp associations` on R1 confirmed an active NTP association with R2 and showed a non-zero poll count, confirming R1 was actively and repeatedly syncing — not just configured to sync once and forget.
- `show ntp status` on both routers showed each device's reported clock precision and drift figures, and confirmed R1's synchronization state explicitly (rather than assuming sync had occurred just because the command didn't error).
- `show logging` on R2 confirmed the configured Syslog server IP, current message count, and trap logging level all matched what had been configured — verifying configuration intent against actual running state rather than trusting the config alone.
- Creating a loopback interface on R2 while watching both the local console and the remote Tftpd32 Syslog window simultaneously confirmed both destinations received a `LINEPROTO-5-UPDOWN` message — but the timestamps differed slightly between the two views, a small but concrete illustration of why synchronized time across every device in the chain (router, transport, server) genuinely matters for precise correlation.
- The MIB Browser successfully read `sysName`, `sysDescr` (containing the embedded IOS version string), and `ipInReceives` from R1 — confirming the SNMP agent, community string, and ACL were all correctly aligned end-to-end, not just individually configured.
- Creating a new loopback interface on R1 generated exactly two SNMP traps, visible in the MIB Browser's Trap Receiver — both Cisco-enterprise (ciscoConfigMan / ccmCLIRunningConfigChanged) notification types tied to the configuration change itself, distinct from the standard `linkUp` trap from the generic SNMP MIB.

## Reflection

- **SNMP vs. Syslog monitoring benefit:** SNMP supports structured, queryable state (you can ask "what is the current value of X" at any time, and receive proactive traps on defined events) whereas Syslog is purely a stream of human-readable text messages the device chooses to emit — SNMP's structured Managed Objects are far easier to consume programmatically or feed into automated alerting/dashboarding than parsing free-text log lines.
- **Why read-only is the safer default for SNMPv1/v2c:** with no real encryption or strong authentication protecting the community string, granting write access over a protocol whose "security" is an unencrypted shared secret travelling across the network is a meaningfully larger risk than read access — a captured "public" read-only string only leaks information, while a captured write-capable string lets an attacker reconfigure the device outright.
- **More structured alternatives to classic SNMP MIB data for automation:** modern network automation increasingly favors structured data formats like JSON or YAML (often delivered over gNMI/NETCONF/RESTCONF rather than raw SNMP polling), since these formats are natively machine-parseable without needing a separate MIB-to-meaning translation step the way numeric OIDs require.

## Files

- [`r1-config.txt`](./r1-config.txt) — Router R1 (OSPF, NTP client, SNMP agent + ACL-restricted write access)
- [`r2-config.txt`](./r2-config.txt) — Router R2 (OSPF, NTP master, Syslog remote logging + severity tuning)
