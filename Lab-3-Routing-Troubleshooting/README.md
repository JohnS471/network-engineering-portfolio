# Lab 3 — Routing Failure Troubleshooting

**Skill focus:** Diagnosing routing failures using a structured bottom-up (OSI) method
**Environment:** Cisco CML
**Protocols/concepts:** Static routing, next-hop logic, interface states, bidirectional traffic

---

## Topology

```
PC1 (Alpine) 192.168.1.10
   |
R1   G0/0 192.168.1.1  |  G0/1 10.10.10.1
   |   (10.10.10.0/30 point-to-point link)
R2   G0/0 10.10.10.2   |  G0/1 192.168.2.1
   |
PC2 (Alpine) 192.168.2.10
```

---

## Baseline Configuration

Each router only knows about its directly-connected networks, so static routes were needed to reach the far-side network on each side.

**R1** — route to PC2's network:
```
ip route 192.168.2.0 255.255.255.0 10.10.10.2
```

**R2** — route back to PC1's network:
```
ip route 192.168.1.0 255.255.255.0 10.10.10.1
```

**Next-hop logic:** The next-hop on R1's route is `10.10.10.2` (not `192.168.2.1`) because you direct traffic to the other end of the link you're directly connected to. The next-hop has to be an address the router can already reach — its neighbor on the shared link. You hand the packet to your neighbor and trust them to know the next step.

**End hosts** — default gateways set on each PC:
```
PC1:  sudo ip route add default via 192.168.1.1
PC2:  sudo ip route add default via 192.168.2.1
```

Baseline verified: PC1 successfully pings PC2.

---

## Scenario 1 — Missing Static Route

**Symptom:** PC1 can't reach PC2. Worked earlier.

**Diagnosis (bottom-up):**
Pinging outward from PC1, the default gateway responds and the 10.10.10.0/30 link responds, but the 192.168.2.0 network does not. Checking R1's routing table with `show ip route`, the 192.168.2.0/24 network isn't listed — R1 has no way to reach that network, so it drops the packet.

**Fix:**
```
ip route 192.168.2.0 255.255.255.0 10.10.10.2
```

**Verification:** `show ip route` on R1 confirms the route is back (control plane), then PC1 pings PC2 successfully (data plane).

---

## Scenario 2 — Administratively Down Interface

**Symptom:** PC1 can't reach PC2. Same symptom as before — but not the same cause.

**Diagnosis (bottom-up):**
Pinging outward, traffic reaches the default gateway but dies at R1's G0/1 interface. Running `show ip int brief` shows G0/1 as **administratively down / down** — meaning it was manually shut down, not a routing problem.

This also explains a knock-on effect: with G0/1 down, the 10.10.10.0/30 connected route disappears from the table, *and* the static route to 192.168.2.0 drops with it (its next-hop is no longer reachable). A single Layer 1 failure cascaded upward and looked like a Layer 3 problem in the routing table — which is exactly why checking the interface status first matters.

**Fix:**
```
interface g0/1
 no shutdown
```

**Verification:** `show ip int brief` shows G0/1 **up / up**, `show ip route` shows both routes restored, PC1 pings PC2 successfully.

---

## Scenario 3 — Missing Return Route (One-Way Failure)

**Symptom:** PC1 can't ping PC2 — but R1 can ping R2 fine and all interfaces are up. Routing looks fine from R1's side.

**Diagnosis (bottom-up):**
The forward path is intact — the echo request reaches PC2 without issue. The problem is the return: PC2 sends its reply to R2, but R2 has no route back to the 192.168.1.0 network in its routing table, and it isn't directly connected to that network either. So the reply gets dropped at R2.

**Key takeaway:** A ping has to succeed in *both* directions. The forward path working tells you nothing about whether the reply can get home. Every conversation across a network needs a route in both directions.

**Fix:**
```
ip route 192.168.1.0 255.255.255.0 10.10.10.1
```

**Verification:** `show ip route` on R2 confirms the 192.168.1.0/24 route, then PC1 pings PC2 successfully both ways.

---

## What I Learned

Three structurally different faults — a missing route, a downed interface, and a broken return path — all produced the *same* symptom from PC1's point of view: "can't reach PC2." The bottom-up method worked every time because it doesn't assume where the problem is. Isolating by distance (how far does the ping get?) points to the layer, then the right `show` command confirms the cause. The biggest lesson was Scenario 3: traffic is bidirectional, and a route only fixes one direction.
