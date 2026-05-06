# Networking Fundamentals Revision Notes

## From LAN to VXLAN

---

# 1. Core Mental Model

Networking becomes easier when you separate the layers:

```text
L1 = physical/signal world
L2 = MAC address / Ethernet frame world
L3 = IP address / routing world
```

A simple way to remember:

```text
L1: Can signals move?
L2: Can Ethernet frames move between MAC addresses?
L3: Can IP packets move between networks?
```

---

# 2. LAN

## What is a LAN?

**LAN = Local Area Network**

A LAN is a network of devices located in a limited physical area, such as:

```text
Home
Office
School
Data center
Lab room
```

A LAN does **not** always mean one IP subnet.

Example:

```text
Office LAN
├── 192.168.10.0/24
├── 192.168.20.0/24
└── 10.0.0.0/24
```

All of these can still be part of one LAN if they belong to the same local environment.

## Key point

```text
LAN is about locality.
Subnet is about IP addressing.
```

---

# 3. Network Link

## What is a Network Link?

A **network link** is a connection path between two or more network devices.

Examples:

```text
Laptop ─── Ethernet cable ─── Switch
Phone ))) Wi-Fi ((( Access Point
VM ─── veth pair ─── Linux bridge
```

A link can be:

```text
Physical cable
Fiber
Wi-Fi
Virtual Ethernet pair
Tunnel
```

## Key point

```text
A link is the path that allows devices to exchange data.
```

---

# 4. Network Segment

## What is a Network Segment?

A **network segment** is a portion of a network.

But the meaning depends on the layer.

```text
L1 segment = physical/signal segment
L2 segment = MAC/broadcast segment
L3 segment = IP subnet
```

Do not use the word “segment” alone without thinking about the layer.

---

# 5. L1 Segment

## What is an L1 Segment?

An **L1 segment** is a physical or signal-level connection.

Example:

```text
Computer A ─── cable ─── Computer B
```

In Linux lab:

```text
namespace A ─── veth pair ─── namespace B
```

A veth pair behaves like a virtual cable.

## L1 does not understand

```text
IP
MAC
Subnet
Routing
VLAN
```

L1 only carries signals or bits.

---

# 6. Evolution of Physical Segments

## Old Ethernet: Shared Coaxial Cable

Early Ethernet used a shared cable.

```text
PC1 ─┬──────── shared cable ────────┬─ PC4
PC2 ─┤                              │
PC3 ─┘                              ┘
```

All devices shared the same physical medium.

Problem:

```text
If two devices transmit at the same time,
their signals can collide.
```

---

## Hub-based Ethernet

A hub looks like a central device:

```text
        Hub
      /  |  \
    PC1 PC2 PC3
```

But a hub is not intelligent.

If PC1 sends a frame, the hub repeats it to everyone.

```text
PC1 → Hub → PC2, PC3, PC4
```

So hub networks still behave like shared media.

---

## Switch-based Ethernet

A switch is smarter.

```text
PC1 ─┐
PC2 ─┼── Switch
PC3 ─┘
```

A switch learns MAC addresses and forwards frames only where needed.

Modern Ethernet usually uses switches and full-duplex links, so classic collisions are mostly gone.

---

# 7. Collision Domain

## What is a Collision Domain?

A **collision domain** is an area where two devices transmitting at the same time can collide.

Common in:

```text
Old shared Ethernet
Hub networks
```

Less relevant in:

```text
Modern switched full-duplex Ethernet
```

## Example

```text
PC1 sends
PC2 sends
same shared cable
→ collision
```

## Key point

```text
Switches reduce or eliminate classic Ethernet collision domains.
```

---

# 8. L2 Segment

## What is an L2 Segment?

An **L2 segment** is a MAC-level network.

Devices in the same L2 segment can exchange Ethernet frames using MAC addresses.

Example:

```text
PC1 ─┐
PC2 ─┼── Switch
PC3 ─┘
```

All devices connected through the same switch can be part of the same L2 segment.

## Ethernet Frame

A simplified Ethernet frame:

```text
┌────────────────┬──────────────┬───────────┬─────────┬─────┐
│ Destination MAC│ Source MAC   │ EtherType │ Payload │ CRC │
└────────────────┴──────────────┴───────────┴─────────┴─────┘
```

## L2 uses

```text
MAC addresses
Ethernet frames
Broadcast
Switching
```

---

# 9. Broadcast Domain

## What is a Broadcast Domain?

A **broadcast domain** is the area where an Ethernet broadcast frame can reach.

Ethernet broadcast MAC:

```text
ff:ff:ff:ff:ff:ff
```

If a device sends to this MAC, all devices in the same L2 broadcast domain receive it.

## Example

```text
PC1 ─┐
PC2 ─┼── Switch
PC3 ─┘
```

If PC1 sends broadcast:

```text
PC2 receives it
PC3 receives it
```

## Important

A switch normally forwards broadcast within the same broadcast domain.

A router normally stops broadcast.

```text
Switch extends broadcast domain.
Router separates broadcast domains.
```

---

# 10. ARP and NDP

## Why ARP/NDP are needed

IP uses IP addresses.

Ethernet uses MAC addresses.

So when a device wants to send an IP packet on a local network, it needs to know the destination MAC address.

---

## ARP

**ARP = Address Resolution Protocol**

Used in IPv4.

Purpose:

```text
IPv4 address → MAC address
```

Example:

```text
Host A:
IP  = 192.168.1.10

Host B:
IP  = 192.168.1.20
```

Host A wants to send to Host B.

Host A asks:

```text
Who has 192.168.1.20?
Tell 192.168.1.10
```

This is sent as a broadcast.

Host B replies:

```text
192.168.1.20 is at BB:BB:BB:BB:BB:BB
```

Now Host A can send the Ethernet frame to Host B’s MAC.

---

## NDP

**NDP = Neighbor Discovery Protocol**

Used in IPv6.

It performs similar neighbor discovery tasks, but with IPv6 mechanisms.

NDP can help discover:

```text
IPv6 neighbor MAC address
Default router
Duplicate address detection
Neighbor reachability
```

---

# 11. VLAN

## What is a VLAN?

**VLAN = Virtual LAN**

A VLAN splits one L2 network into multiple isolated L2 broadcast domains.

Example:

```text
Same physical switch:

VLAN 10:
  HR devices

VLAN 20:
  Engineering devices
```

Devices in VLAN 10 do not receive VLAN 20 broadcasts.

## VLAN behavior

```text
VLAN 10 broadcast → only VLAN 10 devices
VLAN 20 broadcast → only VLAN 20 devices
```

## Important idea

A bridge or switch can merge physical links into one L2 segment.

A VLAN does the opposite:

```text
Bridge/switch = merge L2 links
VLAN          = split L2 network
```

## VLAN ID

Traditional VLAN uses a VLAN ID.

Common limitation:

```text
VLAN ID field = 12 bits
Maximum VLANs ≈ 4096
```

This is one reason large cloud environments often need VXLAN or similar overlay technologies.

---

# 12. L3 Segment

## What is an L3 Segment?

An **L3 segment** is an IP subnet.

Examples:

```text
192.168.1.0/24
10.0.0.0/24
172.16.0.0/16
```

A subnet defines which IP addresses are considered local.

---

# 13. Subnet

## What is a subnet?

A subnet is a smaller IP network.

Example:

```text
192.168.10.11/24
```

This means:

```text
IP address = 192.168.10.11
Subnet     = 192.168.10.0/24
```

`/24` means the first 24 bits are the network part.

Equivalent subnet mask:

```text
/24 = 255.255.255.0
```

So this subnet includes:

```text
192.168.10.1
192.168.10.2
...
192.168.10.254
```

## Where does subnet information come from?

A device learns its subnet from IP configuration.

Usually from:

```text
Manual configuration
DHCP
Cloud metadata/configuration
Container/network runtime
```

Example configuration:

```text
IP address:      192.168.10.11
Subnet mask:     255.255.255.0
Default gateway: 192.168.10.1
```

From this, the host knows:

```text
My local subnet is 192.168.10.0/24
```

---

# 14. Sending IP Packets Within the Same L3 Segment

## Scenario

```text
Host A:
IP  = 192.168.1.10
MAC = AA:AA:AA:AA:AA:AA

Host B:
IP  = 192.168.1.20
MAC = BB:BB:BB:BB:BB:BB

Subnet:
192.168.1.0/24
```

Host A wants to send to Host B.

Host A checks:

```text
Is 192.168.1.20 inside my subnet 192.168.1.0/24?
Yes.
```

So Host A sends directly at L2.

But first it needs Host B’s MAC.

Flow:

```text
1. Host A checks destination is same subnet.
2. Host A sends ARP request.
3. Host B replies with MAC.
4. Host A sends Ethernet frame to Host B MAC.
```

Frame:

```text
Ethernet:
  Source MAC      = Host A MAC
  Destination MAC = Host B MAC

IP:
  Source IP      = Host A IP
  Destination IP = Host B IP
```

## Key point

```text
Same subnet → ARP for destination MAC → send directly.
```

No router is needed.

---

# 15. Sending IP Packets Across Different L3 Segments

## Scenario

```text
Host A:
IP = 192.168.10.11/24

Host B:
IP = 192.168.20.21/24

Router:
192.168.10.1
192.168.20.1
```

Host A checks:

```text
Is 192.168.20.21 inside 192.168.10.0/24?
No.
```

So Host A sends the packet to its default gateway.

Important:

```text
L2 destination MAC = router MAC
L3 destination IP  = final Host B IP
```

Frame from Host A to router:

```text
Ethernet:
  Source MAC      = Host A MAC
  Destination MAC = Router MAC

IP:
  Source IP      = Host A IP
  Destination IP = Host B IP
```

Then router forwards the IP packet toward Host B.

On the next link, the router creates a new Ethernet frame.

```text
Ethernet header changes hop-by-hop.
IP source/destination stays end-to-end.
```

## Key point

```text
Different subnet → send to default gateway/router.
```

---

# 16. L2 and L3 Relationship

Common design:

```text
One VLAN / L2 broadcast domain ↔ One IP subnet / L3 segment
```

Example:

```text
VLAN 10 ↔ 192.168.10.0/24
VLAN 20 ↔ 192.168.20.0/24
```

This is common because it keeps isolation clean.

## Possible but messy

You can technically run multiple IP subnets on one L2 broadcast domain.

Example:

```text
Same L2 broadcast domain:
├── 192.168.10.0/24
└── 192.168.20.0/24
```

But isolation is weak because both subnets still share the same broadcast domain.

## Better design

```text
VLAN 10 → subnet 192.168.10.0/24
VLAN 20 → subnet 192.168.20.0/24
```

---

# 17. Router vs Bridge

## Bridge / Switch

A bridge or switch works at L2.

It forwards Ethernet frames based on MAC addresses.

```text
MAC address → port
```

## Router

A router works at L3.

It forwards IP packets based on destination IP and routing table.

```text
Destination IP → next hop/interface
```

## Difference

```text
Bridge/switch:
  Forwards frames inside an L2 domain.

Router:
  Moves packets between L3 networks/subnets.
```

---

# 18. VXLAN

## What is VXLAN?

**VXLAN = Virtual Extensible LAN**

VXLAN creates a virtual L2 network over an L3/IP network.

Simple definition:

```text
VXLAN carries Ethernet frames inside UDP/IP packets.
```

Another way:

```text
Inner world  = tenant Ethernet/L2 frame
Outer world  = provider IP/L3 packet
```

## Why VXLAN exists

Large environments need:

```text
Many isolated networks
Overlapping private IP ranges
L2-like behavior over routed networks
More scale than VLAN
Simple routed physical underlay
```

VLAN has around 4096 IDs.

VXLAN has a 24-bit VNI:

```text
VNI space ≈ 16 million segments
```

---

# 19. VXLAN Components

## Underlay

The real IP network.

Example:

```text
provider_edge_a ─── provider_router ─── provider_edge_b
```

The underlay only needs IP connectivity between VTEPs.

## Overlay

The virtual network built on top of the underlay.

Example:

```text
tenant_a_web ─── virtual L2 network ─── tenant_a_db
```

## VTEP

**VTEP = VXLAN Tunnel Endpoint**

A VTEP does:

```text
Encapsulation:
  Ethernet frame → VXLAN/UDP/IP packet

Decapsulation:
  VXLAN/UDP/IP packet → Ethernet frame
```

In a cloud-like setup, VTEPs live on provider edge hosts or hypervisors.

## VNI

**VNI = VXLAN Network Identifier**

It identifies the virtual L2 network.

Example:

```text
Tenant A = VNI 100
Tenant B = VNI 200
```

VNI separates traffic even if tenants use the same IP addresses.

---

# 20. VXLAN Packet Structure

A VXLAN packet has two worlds: inner and outer.

```text
Outer Ethernet
└── Outer IP
    └── UDP
        └── VXLAN Header
            └── Inner Ethernet Frame
                └── Inner IP Packet
```

## Example

Tenant A web sends to Tenant A database.

Inner traffic:

```text
Inner Ethernet:
  tenant_a_web MAC → tenant_a_db MAC

Inner IP:
  10.0.0.10 → 10.0.0.11
```

Outer traffic:

```text
Outer IP:
  provider_edge_a IP → provider_edge_b IP

UDP:
  destination port 4789

VXLAN:
  VNI 100
```

## Key point

Underlay routers only care about the outer IP packet.

```text
Router sees:
172.16.1.10 → 172.16.2.20

Router does not care about:
10.0.0.10
10.0.0.11
inner MACs
tenant identity
```

---

# 21. VLAN vs VXLAN

| Topic            | VLAN                       | VXLAN                      |
| ---------------- | -------------------------- | -------------------------- |
| Main purpose     | Split L2 network           | Build virtual L2 over L3   |
| Identifier       | VLAN ID                    | VNI                        |
| Scale            | Around 4096 IDs            | Around 16 million VNIs     |
| Depends on       | L2 switching               | L3/IP connectivity         |
| Common use       | Office/campus segmentation | Cloud/data center overlays |
| Broadcast domain | Per VLAN                   | Per VNI                    |

## Simple comparison

```text
VLAN:
  One L2 network split into many isolated L2 networks.

VXLAN:
  Many L3-connected locations joined into virtual L2 networks.
```

---

# 22. Finally

> A LAN is a local network that may contain many subnets. A network link is any path between devices — physical cable, Wi-Fi, or virtual pair. L1 deals with signals, L2 with MAC addresses and Ethernet frames, and L3 with IP addresses and routing. Switches extend L2 broadcast domains while routers separate them. When two hosts are in the same subnet, the sender ARPs for the destination MAC and sends directly; when they are in different subnets, the Ethernet frame goes to the router MAC while the IP destination stays the final host. VLANs split one L2 network into isolated broadcast domains using a 12-bit VLAN ID, limited to around 4096 IDs. VXLAN solves scale by carrying inner Ethernet frames inside UDP/IP packets, using a 24-bit VNI that supports around 16 million virtual networks. VTEPs encapsulate and decapsulate VXLAN traffic at the edge of the underlay network. The underlay routers see only the outer provider IPs and never the tenant addresses, which means overlapping tenant IP ranges can coexist without conflict.
