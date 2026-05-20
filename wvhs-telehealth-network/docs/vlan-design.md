# VLAN Design Document

## Western Victoria Health Services — NIT1104 Group 9

---

## Why VLANs at Ballarat?

Ballarat hosts four distinct departments sharing physical switch infrastructure. Without VLANs:

- All broadcast traffic (ARP, DHCP, discovery) reaches every device across all departments
- Radiology imaging data is visible to Administration workstations at Layer 2
- A compromised Admin PC could sniff Radiology frames — a healthcare compliance violation
- No traffic isolation between clinical and administrative systems

VLANs provide Layer 2 segmentation — logical separation without physical separation. Each department behaves as if it has its own dedicated switch.

### Why no VLANs at Ararat/Horsham?

The spoke sites have only two departments each, and each department connects to a separate router interface (separate subnet). The physical separation of router interfaces provides equivalent isolation without VLAN configuration overhead.

---

## VLAN Assignments

| VLAN ID | Name              | Subnet       | Purpose                                    |
|---------|-------------------|--------------|--------------------------------------------|
| 10      | Radiology         | 10.7.9.0/28  | Patient imaging — highest sensitivity data |
| 20      | Cardiology        | 10.7.9.16/29 | Cardiac specialist systems                 |
| 30      | Administration    | 10.7.9.32/28 | Billing, scheduling, HR                    |
| 40      | IT Infrastructure | 10.7.9.48/29 | Network management — full access           |

---

## 802.1Q Trunking

### What is 802.1Q?

IEEE 802.1Q inserts a 4-byte tag into Ethernet frames containing:

- TPID (Tag Protocol Identifier): 0x8100 — identifies the frame as VLAN-tagged
- PCP (Priority Code Point): 3 bits — used by QoS
- DEI (Drop Eligible Indicator): 1 bit
- VID (VLAN Identifier): 12 bits — allows up to 4094 VLANs (0 and 4095 reserved)

### Trunk Links in this Network

| Link                             | Port Type | VLANs Carried          |
|----------------------------------|-----------|------------------------|
| SW-Core-Ballarat ↔ R-Ballarat    | Trunk     | 10, 20, 30, 40         |
| SW-Core-Ballarat ↔ SW-Radiology  | Trunk     | 10 only                |
| SW-Core-Ballarat ↔ SW-Cardiology | Trunk     | 20 only                |
| SW-Core-Ballarat ↔ SW-Admin      | Trunk     | 30 only                |
| SW-Core-Ballarat ↔ SW-IT         | Trunk     | 40 only                |
| All end-device ports             | Access    | Single VLAN (untagged) |

---

## Router-on-a-Stick Configuration

R-Ballarat GigabitEthernet0/0 is divided into 4 subinterfaces:

```
R-Ballarat
GigabitEthernet0/0 (no IP — trunk carrier)
├── GigabitEthernet0/0.10  encapsulation dot1Q 10  IP: 10.7.9.1/28
├── GigabitEthernet0/0.20  encapsulation dot1Q 20  IP: 10.7.9.17/29
├── GigabitEthernet0/0.30  encapsulation dot1Q 30  IP: 10.7.9.33/28
└── GigabitEthernet0/0.40  encapsulation dot1Q 40  IP: 10.7.9.49/29
```

Each subinterface serves as the default gateway for its VLAN. When a device sends a packet to a different subnet, it sends it to its gateway IP — the router subinterface — which routes it to the destination VLAN.

### Inter-VLAN Packet Flow Example (VLAN 20 → VLAN 30)

1. Cardiology PC (10.7.9.18) sends to Admin server (10.7.9.35)
2. Destination is different subnet → send to gateway 10.7.9.17
3. Frame tagged VLAN 20 travels: SW-Cardiology → trunk → SW-Core-Ballarat → trunk → R-Ballarat .20 subinterface
4. Router strips frame, reads IP destination 10.7.9.35 → matches 10.7.9.32/28 (VLAN 30 subinterface)
5. Router forwards frame back down trunk tagged VLAN 30
6. SW-Core-Ballarat → SW-Admin → Admin server
