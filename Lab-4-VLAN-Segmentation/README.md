# Lab 4 — VLAN Segmentation (School District Scenario)

**Skill focus:** Layer 2 segmentation with VLANs and inter-VLAN routing (router-on-a-stick)
**Environment:** Cisco CML
**Concepts:** VLANs, access ports, trunking (802.1Q), subinterfaces, inter-VLAN routing

---

## Topology

```
   VLAN 10 (Staff)              VLAN 20 (Students)
   192.168.10.0/24              192.168.20.0/24
        |                            |
   [PC-Staff]                   [PC-Student]
   .10  |                       .10  |
        |  Gi0/1          Gi0/2  |
        +-----------[ SW1 ]------+
                       | Gi0/0 (802.1Q trunk)
                       |
                    [ R1 ]  router-on-a-stick
                 Gi0/0.10  -> 192.168.10.1 (Staff gateway)
                 Gi0/0.20  -> 192.168.20.1 (Students gateway)
```

*(Replace with Lucidchart diagram before publishing.)*

---

## The Scenario

VLANs are used to segment broadcast domains logically, so you don't need multiple physical switches to separate LAN networks. A school would want this to keep student and admin/staff traffic separate — especially for security, so a rogue student can't tap into the network with malicious intent and reach sensitive systems.

**Design:** one switch (SW1) carrying two VLANs — Staff (VLAN 10) and Students (VLAN 20) — with R1 providing controlled routing between them.

---

## Step 1 — Create the VLANs on SW1

```
vlan 10
 name Staff
vlan 20
 name Students
```

Verified with `show vlan brief` — VLANs 10 and 20 exist, but no access ports assigned yet (all ports still in default VLAN 1).

---

## Step 2 — Assign Access Ports

```
interface g0/1
 switchport mode access
 switchport access vlan 10
interface g0/2
 switchport mode access
 switchport access vlan 20
```

- `switchport mode access` sets the port to carry a single VLAN and connect to an end device.
- `switchport access vlan X` drops the port into that VLAN.

`show vlan brief` now shows Gi0/1 in VLAN 10 and Gi0/2 in VLAN 20.

---

## Step 3 — Prove Isolation (the key moment)

From PC-Staff, pinged PC-Student (192.168.20.10). **The ping failed.**

This failure proved that the two VLANs do not allow communication between them directly. Even though both PCs are on the same physical switch, being in different VLANs isolates them at Layer 2. This is the whole point of segmentation — the isolation is working as intended.

---

## Step 4 — Configure the Trunk (SW1)

The link from SW1 to R1 (Gi0/0) must carry both VLANs to the router, so it becomes a trunk.

```
interface g0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

**Gotcha caught during config:** trying to set trunk mode first returned *"An interface whose trunk encapsulation is Auto cannot be configured to trunk mode."* The fix is to set `switchport trunk encapsulation dot1q` **before** setting trunk mode. Scoping the trunk with `allowed vlan 10,20` is a security habit — only the VLANs that need to cross are permitted.

Verified with `show interface g0/0 trunk` — trunking, 802.1Q, VLANs 10 and 20 allowed and active.

---

## Step 5 — Router-on-a-Stick (R1)

R1 has one physical interface facing the switch, but needs to be the gateway for both VLANs. This is done with one subinterface per VLAN.

```
interface g0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface g0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface g0/0
 no shutdown
```

- Each subinterface is tagged with its VLAN via `encapsulation dot1Q`.
- Each subinterface IP is the default gateway for that VLAN's subnet.
- The single physical link (the "stick") carries all VLAN traffic to the router.

Verified with `show ip int brief` — both subinterfaces up/up with their gateway IPs.

---

## Step 6 — Controlled Communication

With the PCs pointing to their subinterface gateways (192.168.10.1 and 192.168.20.1), PC-Staff and PC-Student can now ping each other successfully. Router-on-a-stick gave each host its default gateway and performs the inter-VLAN routing between the two subnets.

**No static routes needed.** `show ip route` on R1 shows both networks as directly connected (C), each via its subinterface:

```
C   192.168.10.0/24 is directly connected, GigabitEthernet0/0.10
C   192.168.20.0/24 is directly connected, GigabitEthernet0/0.20
```

Because R1 is directly connected to both subnets through its subinterfaces, it already knows how to reach both — so it bridges them with no static routing required.

---

## What I Learned

VLANs segment one physical switch into separate broadcast domains, isolating traffic at Layer 2 for security and efficiency. The failed ping proved the isolation was real; router-on-a-stick then provided controlled routing between the VLANs, using subinterfaces as per-VLAN gateways over a single trunk link. The contrast with Lab 3 stuck with me: Lab 3 needed static routes to reach remote networks, but here the router is directly connected to both VLAN subnets, so inter-VLAN routing works automatically. This is the foundation of the school district design — separate networks on shared hardware, with the router controlling what crosses between them.
