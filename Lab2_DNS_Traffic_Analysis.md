# Lab 2 — DNS Traffic Analysis
**Engineer:** Johnathan Sanchez  
**Certification Path:** CCNA | CCNP ENCOR (In Progress)  
**Platform:** Cisco Modeling Labs (CML)  
**Date:** May 2026

---

## Objective

Configure a Cisco router as a DNS server, configure a Linux host to use that DNS server, generate DNS queries for multiple hostnames, and capture and analyze the DNS traffic in Wireshark. The goal is to observe and understand the complete DNS query and response cycle at the packet level.

---

## Topology

```
PC1 (Linux) -------- Router 1 -------- Router 2 -------- PC2 (Linux)
192.168.1.10         G0/0: 192.168.1.1  G0/0: 10.10.10.2  192.168.2.10
                     G0/1: 10.10.10.1   G0/1: 192.168.2.1
```

**Networks:**
- LAN 1: 192.168.1.0/24 (PC1 and Router 1 G0/0)
- WAN Link: 10.10.10.0/30 (Router 1 G0/1 and Router 2 G0/0)
- LAN 2: 192.168.2.0/24 (Router 2 G0/1 and PC2)

---

## Configuration

### Router 1 — DNS Server Setup
```
R1(config)# ip dns server
R1(config)# ip domain-lookup
R1(config)# ip host PC2.lab 192.168.2.10
R1(config)# ip host testsite.lab 192.168.2.10
R1(config)# ip host router2.lab 10.10.10.2
```

### PC1 Linux — DNS Client Configuration
```bash
# Set Router 1 as DNS server
echo "nameserver 192.168.1.1" | sudo tee /etc/resolv.conf

# Verify configuration saved
cat /etc/resolv.conf

# Test DNS resolution
nslookup PC2.lab 192.168.1.1
nslookup testsite.lab 192.168.1.1
nslookup router2.lab 192.168.1.1
```

### Verification Output
```
Server:         192.168.1.1
Address:        192.168.1.1:53
Non-authoritative answer:
Name:   PC2.lab
Address: 192.168.2.10
```

---

## Capture Point

Packet capture performed on the **LAN link between PC1 and Router 1 G0/0** to capture DNS queries leaving PC1 and responses returning from Router 1.

Wireshark filter used: `dns`

---

## Findings

### DNS Query and Response Pairs Observed

| Packet | Source | Destination | Type | Info |
|---|---|---|---|---|
| 5 | 192.168.1.10 | 192.168.1.1 | DNS Query | A record for PC2.lab |
| 6 | 192.168.1.10 | 192.168.1.1 | DNS Query | AAAA record for PC2.lab |
| 7 | 192.168.1.1 | 192.168.1.10 | DNS Response | PC2.lab = 192.168.2.10 |
| 15 | 192.168.1.10 | 192.168.1.1 | DNS Query | A record for testsite.lab |
| 16 | 192.168.1.10 | 192.168.1.1 | DNS Query | AAAA record for testsite.lab |
| 17 | 192.168.1.1 | 192.168.1.10 | DNS Response | testsite.lab = 192.168.2.10 |
| 22 | 192.168.1.10 | 192.168.1.1 | DNS Query | A record for router2.lab |
| 23 | 192.168.1.10 | 192.168.1.1 | DNS Query | AAAA record for router2.lab |
| 24 | 192.168.1.1 | 192.168.1.10 | DNS Response | router2.lab = 10.10.10.2 |

---

### Detailed Packet 7 Analysis — DNS Response for PC2.lab

**Layer 3 — IP Header:**
- Source: 192.168.1.1 (Router 1 DNS server)
- Destination: 192.168.1.10 (PC1)

**Layer 4 — UDP Header:**
- Source Port: 53 (DNS server)
- Destination Port: 40556 (ephemeral port chosen by PC1)

**DNS Response Fields:**
- Transaction ID: 0xf00d — matches the original query in packet 5
- Flags: Standard query response, No error
- Questions: 1
- Answer RRs: 1
- Authority RRs: 0
- Additional RRs: 0

**Query Section:**
- PC2.lab: type A, class IN

**Answer Section:**
- PC2.lab: type A, class IN, addr 192.168.2.10

**Resolution Time: 8.007 milliseconds**

---

## Key Concepts Demonstrated

| Concept | Observed Behavior |
|---|---|
| DNS uses UDP port 53 | Source port 53 on all DNS responses |
| Transaction IDs | Each query/response pair shares a unique ID (0xf00d, 0xf1bd, 0x5cd0) |
| A record query | PC1 asked for IPv4 address of each hostname |
| AAAA record query | Alpine Linux simultaneously queried for IPv6 addresses |
| No error response | Router 1 successfully resolved all hostnames |
| Unanswered AAAA queries | Router 1 only had IPv4 host entries — IPv6 queries got no response |
| Ephemeral ports | Client used random high port (40556) for its side of the DNS transaction |

---

## Linux Commands Used

| Command | Purpose |
|---|---|
| `echo "nameserver X.X.X.X" \| sudo tee /etc/resolv.conf` | Set DNS server on Alpine Linux |
| `cat /etc/resolv.conf` | Verify DNS configuration |
| `nslookup hostname DNS-server-IP` | Query DNS server directly |
| `ping IP -c 1` | Send single ping to verify connectivity |

---

## Troubleshooting Insights

**If DNS queries go out but no responses return:**
- DNS server may be unreachable — check routing and firewall rules
- DNS server may not be configured — verify `ip dns server` on Cisco router
- Wrong DNS server IP configured on client

**If DNS response shows NXDOMAIN:**
- Hostname does not exist in DNS
- Hostname is misspelled
- Wrong DNS server is being queried

**How to identify DNS issues in Wireshark:**
- Filter: `dns`
- Look for queries without matching responses
- Look for responses with error flags instead of "No error"
- Check transaction IDs to match queries to responses
- Check that responses come from port 53

**The key diagnostic flow:**
1. Can the user ping an external IP (8.8.8.8)? If yes — internet routing works
2. Can the user ping a domain name (google.com)? If no — DNS is broken
3. Open Wireshark, filter by dns — are queries going out? Are responses coming back?

---

## Lessons Learned

1. DNS uses UDP port 53 — both queries and responses use this port on the server side
2. Modern operating systems query for both A (IPv4) and AAAA (IPv6) records simultaneously
3. Transaction IDs link queries to responses — critical for matching pairs in a busy capture
4. NXDOMAIN means the hostname doesn't exist in DNS — not a connectivity issue
5. DNS resolution time is visible in Wireshark — useful for diagnosing slow name resolution
6. Linux DNS configuration lives in /etc/resolv.conf — the nameserver line points to the DNS server
7. nslookup with explicit server IP bypasses /etc/resolv.conf — useful for testing specific DNS servers directly

---

## Next Steps

- Capture a failed DNS lookup and observe the NXDOMAIN response
- Simulate a DNS server outage and observe what happens in the capture
- Capture DNS traffic on a real network and observe recursive resolution
- Learn how DHCP automatically distributes DNS server information to clients

---

*Lab documented as part of a self-directed networking portfolio. Built in Cisco Modeling Labs (CML) using IOS routers and Alpine Linux nodes.*
