# Lab 1 — Wireshark Packet Analysis
**Engineer:** Johnathan Sanchez  
**Certification Path:** CCNA | CCNP ENCOR (In Progress)  
**Platform:** Cisco Modeling Labs (CML)  
**Date:** May 2026

---

## Objective

Build a multi-router topology in Cisco CML, generate network traffic between two end hosts, and capture and analyze packets using Wireshark. The goal is to observe and understand real network behavior at the packet level — including ICMP, ARP, DHCP, and CDP traffic — and correlate findings to OSI model concepts.

---

## Topology

```
PC1 -------- Router 1 -------- Router 2 -------- PC2
192.168.1.10  G0/0: 192.168.1.1  G0/0: 10.0.0.2  192.168.2.10
              G0/1: 10.0.0.1     G0/1: 192.168.2.1
```

**Networks:**
- LAN 1: 192.168.1.0/24 (PC1 and Router 1 G0/0)
- WAN Link: 10.0.0.0/30 (Router 1 G0/1 and Router 2 G0/0)
- LAN 2: 192.168.2.0/24 (Router 2 G0/1 and PC2)

**Static Routes Configured:**
- Router 1: `ip route 192.168.2.0 255.255.255.0 10.0.0.2`
- Router 2: `ip route 192.168.1.0 255.255.255.0 10.0.0.1`

---

## Capture Points

Two separate captures were performed:

1. **WAN link between Router 1 and Router 2** — to observe routed traffic and TTL behavior
2. **LAN link between PC1 and Router 1** — to observe ARP, DHCP, and local segment traffic

---

## Findings

### Capture 1 — WAN Link (R1 G0/1 to R2 G0/0)

**ICMP Traffic:**
- PC1 (192.168.1.10) successfully pinged PC2 (192.168.2.10)
- ICMP echo requests and echo replies were visible in alternating pairs
- TTL on ICMP requests: 63 — decremented once by Router 1 from the original TTL of 64
- TTL on ICMP replies: 63 — decremented once by Router 2 from PC2's original TTL of 64
- Round trip time: approximately 3-4ms

**CDP Traffic:**
- Cisco Discovery Protocol packets observed from Router 1
- Device ID: R1, Port ID: GigabitEthernet0/1
- CDP advertises automatically between Cisco devices — confirms routers are Cisco IOS devices

**Key Observation — MAC Address Behavior:**
- Source MAC on WAN link: Router 1's G0/1 interface MAC
- Destination MAC on WAN link: Router 2's G0/0 interface MAC
- PC1 and PC2 MAC addresses were not visible on this link
- Confirms that MAC addresses are local to each network segment and are replaced at each router hop — only IP addresses remain constant end to end

---

### Capture 2 — LAN Link (PC1 to Router 1 G0/0)

**Gratuitous ARP (Packet 1):**
- Source: Router 1's MAC address
- Destination: Broadcast (FF:FF:FF:FF:FF:FF)
- Router 1 announced its own IP-to-MAC mapping without being asked
- Gratuitous ARPs occur when a device comes online or when the ARP cache is cleared
- Purpose: Update all devices on the segment with the router's current MAC address

**ARP Request (Packet 2):**
- Source: Router 1's MAC
- Info: "Who has 192.168.1.10? Tell 192.168.1.1"
- Router 1 needed PC1's MAC address to forward return traffic
- Sent as a broadcast — all devices on the segment receive it

**ARP Reply (Packet 3):**
- Source: PC1's MAC (52:54:00:d2:ef:46)
- Destination: Router 1's MAC
- Info: "192.168.1.10 is at 52:54:00:d2:ef:46"
- PC1 responded with its MAC address — unicast reply directly to Router 1
- Router 1 stored this mapping in its ARP cache

**DHCP Discover (Packet 9):**
- Source IP: 0.0.0.0 — PC1 had no IP address at this point
- Destination IP: 255.255.255.255 — broadcast to entire network
- This is the D in the DORA process (Discover, Offer, Request, Acknowledge)
- Confirms a device with no IP address must broadcast to find a DHCP server

**TTL Behavior on LAN Link:**
- ICMP requests from PC1: TTL=64 (PC1 is the originator, full TTL)
- ICMP replies returning from PC2: TTL=62 (decremented twice — once by Router 2, once by Router 1)
- By comparing TTL at different capture points, the number of hops a packet has traversed can be calculated

---

## Key Concepts Demonstrated

| Concept | Observed Behavior |
|---|---|
| ARP Request/Reply | Router asked for PC1's MAC, PC1 replied with its MAC address |
| Gratuitous ARP | Router announced its IP-MAC mapping without being prompted |
| DHCP Discover | PC1 sent broadcast from 0.0.0.0 to 255.255.255.255 |
| TTL Decrement | TTL=64 at source, TTL=63 after one hop, TTL=62 after two hops |
| MAC Address Scope | MAC addresses changed at each router — IP addresses stayed the same end to end |
| ICMP Echo Request/Reply | Ping packets matched by sequence number in Wireshark |
| CDP | Cisco routers automatically advertise to neighbors |

---

## Wireshark Filters Used

| Filter | Purpose |
|---|---|
| `icmp` | Show only ping traffic |
| `arp` | Show only ARP traffic |
| `dns` | Show only DNS traffic |
| `icmp and ip.ttl < 64` | Show ICMP packets that have already passed through at least one router |

---

## Troubleshooting Insights Gained

**If ICMP replies are missing in a capture:**
- Destination may be unreachable
- Firewall may be blocking ICMP replies
- Return path may have a routing issue

**If a user can ping an IP but not load websites:**
- Layer 3 routing is working (proven by successful IP ping)
- DNS is likely broken — domain names are not resolving to IP addresses
- Check DNS server settings and connectivity to DNS server

**How to calculate hops from TTL:**
- Default TTL = 64 (Cisco) or 128 (Windows)
- Hops traversed = Default TTL minus observed TTL
- Example: TTL=62 means 2 router hops from the source

---

## Lessons Learned

1. Wireshark captures on different links in the same topology show different information — always capture at the right point for the problem you are diagnosing
2. ARP must resolve before any IP traffic can flow on a segment — if ARP fails, nothing works
3. MAC addresses are strictly local — you will never see the original sender's MAC after the first router hop
4. TTL is a powerful diagnostic tool — it reveals how many hops a packet has taken without running traceroute
5. CDP and LOOP packets are normal background traffic on Cisco links — not a sign of a problem
6. DHCP Discover always comes from 0.0.0.0 because the device has no IP yet — this is a broadcast by design

---

## Next Steps

- Repeat this lab generating DNS traffic and capturing DNS queries and responses
- Introduce a routing failure and observe what changes in the packet capture
- Add a third router and observe TTL decrement across three hops
- Practice Wireshark filter syntax for common protocols: TCP, UDP, HTTP, DNS, DHCP

---

*Lab documented as part of a self-directed networking portfolio. Built in Cisco Modeling Labs (CML) using IOS routers and virtual PCs.*
