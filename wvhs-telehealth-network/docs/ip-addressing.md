# IP Addressing & VLSM Design
## Western Victoria Health Services — NIT1104 Group 9

---

## Base Network

**Address Space:** `10.7.9.0/24`  
**Total Addresses:** 256 (254 usable)  
**Address Family:** Private IPv4 (RFC 1918 — 10.0.0.0/8)  
**Method:** Variable Length Subnet Masking (VLSM)

### Why 10.x.x.x private addressing?
RFC 1918 defines three private address ranges:
- `10.0.0.0/8` — up to 16.7 million addresses
- `172.16.0.0/12` — up to 1 million addresses
- `192.168.0.0/16` — up to 65,534 addresses

We use `10.7.9.0/24` (derived from course/group number: NIT**1**1**0**4, Group **9**) because:
1. Private addresses are never routed on the public internet — no external conflicts
2. No cost — unlike public IP addresses
3. 10.x.x.x provides the largest available range for future expansion

---

## VLSM Design Process

### Why VLSM over Fixed-Length Subnetting (FLSM)?

**FLSM example (what we did NOT do):**
Give every subnet a /27 (30 usable hosts). The Cardiology subnet needs 4 hosts — wastes 26 addresses. The WAN links need 2 hosts — wastes 28 addresses each. Total waste: hundreds of addresses.

**VLSM approach (what we did):**
Give each subnet exactly the right size. Cardiology needs 4 hosts → /29 (6 usable). WAN links need 2 → /30 (2 usable). Zero unnecessary waste.

### VLSM Calculation Method

For each subnet:
1. Count required hosts (actual devices)
2. Add 30% growth headroom: `required × 1.3`, round up
3. Add 2 (network address + broadcast)
4. Find smallest power of 2 ≥ that total
5. CIDR = 32 − log₂(block size)

**Example — Radiology:**
- Devices: 5 (2 workstations + 2 servers + 1 PACS)
- With 30% growth: 5 × 1.3 = 6.5 → 7 hosts required
- Add 2: need 9 total addresses
- Next power of 2 ≥ 9 = 16 → block size 16
- CIDR: 32 − 4 = /28 ✓

**Example — WAN link:**
- Devices: 2 (one IP per router end)
- No growth needed for point-to-point link
- Add 2: need 4 total addresses
- Block size 4 → /30 ✓

---

## Complete VLSM Allocation Table

Subnets assigned largest-to-smallest from 10.7.9.0, sequentially:

| #  | Subnet Name                 | Network    | CIDR | Subnet Mask     | Gateway   | Host Range | Broadcast  | Required | Usable |
|----|-----------------------------|------------|------|-----------------|-----------|------------|------------|----------|--------|
| 1  | Radiology (VLAN 10)         | 10.7.9.0   | /28  | 255.255.255.240 | 10.7.9.1  | .2 – .14   | 10.7.9.15  | 7        | 14     |
| 2  | Cardiology (VLAN 20)        | 10.7.9.16  | /29  | 255.255.255.248 | 10.7.9.17 | .18 – .22  | 10.7.9.23  | 4        | 6      |
| 3  | Administration (VLAN 30)    | 10.7.9.32  | /28  | 255.255.255.240 | 10.7.9.33 | .34 – .46  | 10.7.9.47  | 8        | 14     |
| 4  | IT Infrastructure (VLAN 40) | 10.7.9.48  | /29  | 255.255.255.248 | 10.7.9.49 | .50 – .54  | 10.7.9.55  | 3        | 6      |
| 5  | Ararat Clinical             | 10.7.9.64  | /29  | 255.255.255.248 | 10.7.9.65 | .66 – .70  | 10.7.9.71  | 6        | 6      |
| 6  | Ararat Reception            | 10.7.9.72  | /29  | 255.255.255.248 | 10.7.9.73 | .74 – .78  | 10.7.9.79  | 4        | 6      |
| 7  | Horsham Clinical            | 10.7.9.80  | /28  | 255.255.255.240 | 10.7.9.81 | .82 – .94  | 10.7.9.95  | 8        | 14     |
| 8  | Horsham Reception           | 10.7.9.96  | /29  | 255.255.255.248 | 10.7.9.97 | .98 – .102 | 10.7.9.103 | 4        | 6      |
| 9  | WAN Ballarat–Ararat         | 10.7.9.112 | /30  | 255.255.255.252 | N/A       | .113–.114  | 10.7.9.115 | 2        | 2      |
| 10 | WAN Ballarat–Horsham        | 10.7.9.116 | /30  | 255.255.255.252 | N/A       | .117–.118  | 10.7.9.119 | 2        | 2      |

**Total addresses used:** 10×(16+8+16+8+8+8+16+8+4+4) = 96 out of 254 usable  
**Address utilisation:** 37.8% — leaves ample room for future subnets

---

## Address Gaps Explained

You'll notice gaps between subnets (e.g., .55 to .64, .103 to .112). These are due to **subnet boundary alignment**:
- A /29 (block of 8) must start at multiples of 8: 0, 8, 16, 24, 32, 40, 48, 56, 64...
- A /28 (block of 16) must start at multiples of 16: 0, 16, 32, 48, 64, 80, 96...
- A /30 (block of 4) must start at multiples of 4: 0, 4, 8, 112, 116...

Misaligned subnets cause routing ambiguity — always align to block boundaries.

---

## WAN Link Design (/30 justification)

A WAN serial point-to-point link connects exactly two endpoints — one IP per router.
- Network address: 10.7.9.112 (not assignable)
- R-Ballarat Serial0/0/0: 10.7.9.113
- R-Ararat Serial0/0/0: 10.7.9.114
- Broadcast: 10.7.9.115 (not assignable)

Using /29 would provide 6 usable addresses — wasting 4. /30 is the industry-standard choice for point-to-point WAN links.
