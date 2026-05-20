# Security Design Document

## Western Victoria Health Services — NIT1104 Group 9

---

## Security Architecture Overview

Security is implemented in four layers — each protecting against different threat vectors.

```
Layer 4 — Physical Security      (unused port shutdown)
Layer 3 — ACL Controls           (packet filtering)
Layer 2 — VLAN Segmentation      (traffic isolation)
Layer 1 — Device Hardening       (authentication & encryption)
```

Defence in depth: even if one layer is bypassed, the others provide continued protection.

---

## Layer 1: Device Hardening (All 11 Devices)

Applied uniformly to all 3 routers and 9 switches.

### Enable Secret

```
enable secret <password>
```

- Protects privileged EXEC mode (full device control)
- Uses MD5 hashing — cannot be read from config file in plain text
- `enable secret` supersedes `enable password` — always use secret

### Service Password Encryption

```
service password-encryption
```

- Encrypts all plain-text passwords stored in running-config
- Uses Cisco Type 7 encryption (weak cipher — deterrent only, not cryptographically secure)
- Prevents casual shoulder-surfing of configuration files

### Console Line Authentication

```
line console 0
 password <password>
 login
 exec-timeout 5 0
```

- Requires password for physical CLI access
- `exec-timeout 5 0` = auto-logout after 5 minutes idle — prevents unattended sessions
- Critical in a healthcare facility where console ports may be accessible to non-IT staff

### VTY Line Authentication

```
line vty 0 4
 password <password>
 login
 exec-timeout 5 0
```

- Protects Telnet/SSH remote access (5 simultaneous sessions)
- Same auto-logout policy as console

### MOTD Banner

```
banner motd # AUTHORISED ACCESS ONLY... #
```

- Displayed before login prompt
- Legal significance: establishes notice of restricted access
- Required for successful prosecution of unauthorised access in Australian law

---

## Layer 2: VLAN Segmentation

### Security Benefit

VLANs enforce Layer 2 isolation — a compromised device in VLAN 30 (Administration) cannot:

- Capture frames from VLAN 10 (Radiology) via packet sniffing
- Receive ARP broadcasts from Radiology devices
- Perform Layer 2 attacks (ARP spoofing, MAC flooding) against Radiology

This is enforced by switch hardware — not just configuration policy.

### VLAN Security Map

```
VLAN 10 (Radiology)     — Highest sensitivity: patient imaging data
VLAN 20 (Cardiology)    — Clinical data: cardiac records
VLAN 30 (Administration)— Administrative: billing, HR (BLOCKED from VLAN 10 via ACL)
VLAN 40 (IT)            — Full access: network management
```

---

## Layer 3: ACL Controls

### ACL 100 — Protecting Radiology Data

**Rule:**

```
access-list 100 deny   ip 10.7.9.32 0.0.0.15 10.7.9.0 0.0.0.15
access-list 100 permit ip any any
```

**Applied:**

```
interface GigabitEthernet0/0.30
 ip access-group 100 in
```

### Detailed Explanation

**What it does:**

- DENIES all IP traffic from the Administration subnet (10.7.9.32–10.7.9.47) to the Radiology subnet (10.7.9.0–10.7.9.15)
- PERMITS everything else

**Why extended ACL (100) not standard (1–99):**

- Standard ACLs can only filter by source IP — would block Admin from *everywhere*, not just Radiology
- Extended ACLs filter by source IP + destination IP + protocol + port
- We need to block a specific source→destination pair, not all traffic from a source

**Wildcard mask math:**

- /28 subnet mask: 255.255.255.240 = 11111111.11111111.11111111.11110000
- Wildcard inverse:   0.0.0.15        = 00000000.00000000.00000000.00001111
- Meaning: match any address where the first 28 bits equal 10.7.9.32's first 28 bits
- Matches: 10.7.9.32, 10.7.9.33, ... 10.7.9.46, 10.7.9.47 (all Admin hosts)

**Why applied INBOUND on VLAN 30 subinterface:**

- Best practice: filter as close to the source as possible
- Packet is dropped before router wastes CPU routing it
- Easier to audit: "traffic arriving FROM Admin cannot reach Radiology"
- If applied outbound on VLAN 10: router processes packet fully then drops — inefficient

**Healthcare compliance alignment:**

- Implements principle of least privilege: Admin staff have no clinical need to access patient imaging
- Aligns with Australian Privacy Act 1988 — patient data access must be restricted to authorised clinical staff
- ACSC Essential Eight: application control and access control requirements

---

## Layer 4: Physical Security

### Unused Port Shutdown

```
interface range FastEthernet0/X - 24
 shutdown
```

- Applied across all 9 switches
- Administratively down ports reject all physical connections
- Protects against:
  - Rogue device attachment in public clinic areas (waiting rooms)
  - Network reconnaissance by plugging in a laptop
  - Unauthorised access via unmonitored switch ports
- To re-enable: requires authenticated CLI access by an administrator

---

## Security Testing Results

| Test                           | Method                           | Expected                | Result      |
|--------------------------------|----------------------------------|-------------------------|-------------|
| Admin→Radiology blocked        | ping 10.7.9.5 from Admin PC      | Request timeout         | ✅ DENIED    |
| Cardiology→Radiology permitted | ping 10.7.9.5 from Cardiology PC | Reply received          | ✅ PERMITTED |
| Console requires password      | Physical console access          | Password prompt         | ✅ ENFORCED  |
| VTY requires password          | Telnet to device IP              | Password prompt         | ✅ ENFORCED  |
| Unused ports inactive          | Connect device to shut port      | No link                 | ✅ BLOCKED   |
| ACL match counter              | show access-lists                | Deny counter increments | ✅ CONFIRMED |
