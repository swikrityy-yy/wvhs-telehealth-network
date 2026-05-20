# Network Design Document

## Western Victoria Health Services — Regional Telehealth Network

**Project:** NIT1104 Computer Networks — Group 9  
**Version:** 1.0  
**Sites:** Ballarat (hub) · Ararat (spoke) · Horsham (spoke)

---

## 1. Business Requirements

### Problem Statement

Western Victoria Health Services operated three geographically dispersed medical facilities with no shared network infrastructure. This created:

- **No specialist access:** Patients in Horsham or Ararat requiring specialist opinions from Ballarat-based specialists faced days-long waits
- **Physical data transport:** Diagnostic results were transported by courier or sent via basic email — no real-time imaging sharing
- **Patient safety risk:** Treatment delays caused by communication gaps between sites
- **No centralised records:** Electronic health records could not be accessed across sites

### Solution Requirements

1. Connect all three sites with reliable inter-site communication
2. Enable real-time sharing of diagnostic imaging (CT, X-Ray, MRI)
3. Centralise electronic health records at Ballarat hub
4. Support live telehealth video consultations (minimum 1–2 Mbps sustained)
5. Enforce departmental security — patient imaging data must be isolated
6. Scalable design with 30% growth capacity per subnet

---

## 2. Topology Design Decision

### Selected Topology: Hub-and-Spoke

```
         Ararat ──── Ballarat ──── Horsham
         (Spoke)     (Hub)        (Spoke)
```

**Why hub-and-spoke over full mesh:**

| Factor               | Hub-and-Spoke                     | Full Mesh                                     |
|----------------------|-----------------------------------|-----------------------------------------------|
| WAN links required   | 2                                 | 3                                             |
| Routing complexity   | Low — spokes default-route to hub | Higher — each site needs routes to all others |
| Centralised services | Natural — all services at hub     | No natural centre                             |
| Management           | Single hub to manage              | Distributed management                        |
| Cost                 | Lower (fewer WAN circuits)        | Higher                                        |

Hub-and-spoke is ideal here because all specialist services (Radiology, Cardiology) are located at Ballarat. Ararat and Horsham do not need a direct path between them — all their traffic targets Ballarat resources anyway.

### Sites

**Ballarat — Central Diagnostic Hub**

- Departments: Radiology, Cardiology, Administration, IT Infrastructure
- 4 VLANs for departmental isolation
- Hosts the specialist services all remote sites depend on
- Hub router (R-Ballarat) handles: inter-VLAN routing, WAN routing, ACL enforcement

**Ararat — Spoke Clinic**

- Departments: Clinical Area, Reception
- Smaller site — no VLAN separation needed (separate router interfaces serve each subnet)
- All inter-site traffic routes to Ballarat

**Horsham — Spoke Clinic**

- Departments: Clinical Area (larger — /28), Reception (/29)
- Same architecture as Ararat
- All inter-site traffic routes to Ballarat

---

## 3. Layer Design (OSI Model Applied)

### Layer 1 — Physical

- Copper straight-through cables: PC-to-switch, switch-to-router
- Serial cables: router-to-router WAN links (DCE/DTE)
- All unused switch ports administratively shut down

### Layer 2 — Data Link

- VLANs: 4 VLANs at Ballarat (VLAN 10/20/30/40)
- 802.1Q trunking: SW-Core-Ballarat ↔ R-Ballarat; SW-Core-Ballarat ↔ departmental switches
- MAC address learning on all switches

### Layer 3 — Network

- IP addressing: VLSM from 10.7.9.0/24
- Routing: Static routing on all 3 routers
- ACLs: Extended ACL 100 on R-Ballarat

---

## 4. Device Selection

### Cisco 2911 Router (×3)

- Supports GigabitEthernet LAN interfaces (required for subinterfaces)
- Supports Serial WAN interfaces with DCE/DTE clock configuration
- Full Cisco IOS feature set: ACLs, subinterfaces, static routing, security hardening
- Selected over 1841: 1841 uses FastEthernet — insufficient for modern healthcare bandwidth requirements

### Cisco 2960 Switch (×9 — 1 core + 4 departmental at Ballarat; 1 each at Ararat, Horsham)

- Layer 2 managed switch with VLAN and trunking support
- Supports 802.1Q VLAN tagging
- 24 FastEthernet ports + 2 GigabitEthernet uplinks
- Departmental separation: dedicated switch per department at Ballarat provides fault isolation

---

## 5. Healthcare-Specific Design Considerations

### Bandwidth Requirements

| Traffic Type     | Bandwidth                  | Implication                        |
|------------------|----------------------------|------------------------------------|
| CT Scan transfer | 100–500 MB per study       | Requires high-throughput WAN links |
| X-Ray series     | 10–50 MB                   | Moderate bandwidth                 |
| Telehealth video | 1–2 Mbps minimum sustained | Low latency, consistent bandwidth  |
| EHR access       | Low bandwidth              | Latency critical — must be < 50ms  |

### Quality of Service (QoS) — Future Implementation

In a production deployment, QoS policies would:

1. Prioritise telehealth video (Class: Real-time)
2. Prioritise EHR transactions (Class: Interactive)
3. Best-effort for imaging file transfers (Class: Bulk data)
4. Lowest priority for administrative email/printing

### Fault Tolerance — Current and Planned

| Failure Scenario          | Current Design                        | Recommended Mitigation                                |
|---------------------------|---------------------------------------|-------------------------------------------------------|
| R-Ballarat fails          | Full outage — single point of failure | Redundant router + HSRP/VRRP + floating static routes |
| WAN link fails            | Site loses all connectivity           | Secondary WAN circuit or backup VPN                   |
| Core switch fails         | Full site outage                      | Redundant core switch + STP + EtherChannel            |
| Departmental switch fails | Only that department affected ✓       | Already isolated by design                            |

The dedicated departmental switch architecture at Ballarat provides inherent fault isolation — a Radiology switch failure does not affect Cardiology or Administration operations.
