# Troubleshooting Log

## Western Victoria Health Services — NIT1104 Group 9

All issues encountered during implementation, with root cause analysis and resolution.

---

## Issue 1 — Router Interface "Invalid Input" Errors

**Severity:** High — blocked all router LAN configuration  
**Phase:** Initial router configuration

### Symptom

When entering interface configuration commands on R-Ballarat:

```
R-Ballarat(config)# interface FastEthernet0/0
                               ^
% Invalid input detected at '^' marker.
```

### Root Cause

The Cisco 2911 router uses **GigabitEthernet** interfaces, not FastEthernet. Interface naming is hardware-specific:

| Router Model   | LAN Interfaces                   | WAN Interfaces     |
|----------------|----------------------------------|--------------------|
| Cisco 1841     | FastEthernet0/0, FastEthernet0/1 | Serial0/0/0        |
| Cisco 2811     | FastEthernet0/0, FastEthernet0/1 | Serial0/0/0        |
| **Cisco 2911** | **GigabitEthernet0/0, 0/1, 0/2** | Serial0/0/0, 0/0/1 |
| Cisco 4321     | GigabitEthernet0/0/0, 0/0/1      | Serial0/1/0        |

Cisco IOS is case-sensitive and name-sensitive for interface references. An incorrect name is treated as an unknown command.

### Resolution

```
R-Ballarat(config)# interface GigabitEthernet0/0
R-Ballarat(config-if)# no ip address
R-Ballarat(config-if)# no shutdown
```

### Verification

```
R-Ballarat# show ip interface brief
GigabitEthernet0/0   unassigned   YES unset   up   up
```

### Lesson Learned

Always verify hardware model interface naming before beginning configuration. Use `show interfaces` or `show ip interface brief` to list all available interfaces on the device.

---

## Issue 2 — Serial WAN Links Stayed Down

**Severity:** Critical — no inter-site connectivity possible  
**Phase:** WAN configuration

### Symptom

After cabling serial connections and configuring IP addresses on all serial interfaces, links remained down:

```
R-Ballarat# show ip interface brief
Serial0/0/0   10.7.9.113   YES manual   down   down
Serial0/0/1   10.7.9.117   YES manual   down   down
```

Both sites showed same result. Ping between routers failed.

### Root Cause

Serial WAN links require a **clock signal** to synchronise bit transmission. On real WAN circuits, the telecommunications provider's DCE equipment generates this clock. The two ends of a serial link are:

- **DCE (Data Circuit-terminating Equipment):** Provides the clock signal — in real networks, the telco's gear
- **DTE (Data Terminal Equipment):** Receives the clock signal — the customer's router

In Cisco Packet Tracer, there is no real telco. Packet Tracer designates one end of each serial cable as DCE. That end **must** be manually configured to generate a clock rate, or neither router can synchronise transmission and the link stays permanently down.

### Diagnosis

```
R-Ballarat# show controllers Serial0/0/0
...
DCE V.35, clock rate 0
```

The `clock rate 0` confirmed the DCE end had no clock configured.

### Resolution

On R-Ballarat's serial interfaces (both are DCE ends):

```
R-Ballarat(config)# interface Serial0/0/0
R-Ballarat(config-if)# clock rate 64000
R-Ballarat(config-if)# no shutdown

R-Ballarat(config)# interface Serial0/0/1
R-Ballarat(config-if)# clock rate 64000
R-Ballarat(config-if)# no shutdown
```

### Verification

```
R-Ballarat# show ip interface brief
Serial0/0/0   10.7.9.113   YES manual   up   up
Serial0/0/1   10.7.9.117   YES manual   up   up

R-Ballarat# ping 10.7.9.114
!!!!!   (5/5 success)
```

### Lesson Learned

For any serial WAN link in Packet Tracer:

1. Use `show controllers Serial0/0/0` to identify DCE vs DTE
2. Always configure `clock rate` on the DCE end
3. In production, the service provider handles this — `clock rate` is only needed in lab environments

---

## Issue 3 — VLAN Subinterface IPs Would Not Apply

**Severity:** High — inter-VLAN routing non-functional  
**Phase:** Router-on-a-stick configuration

### Symptom

When configuring subinterface IP addresses on R-Ballarat, IPs appeared to apply but inter-VLAN routing did not work. On closer inspection, the subinterfaces were not responding.

### Root Cause

An IP address had been previously assigned to the **physical interface** GigabitEthernet0/0:

```
interface GigabitEthernet0/0
 ip address 10.7.9.1 255.255.255.240   ← This was the problem
```

Cisco IOS does not permit subinterfaces to have IP addresses when the parent physical interface also has an IP address. The physical interface "owns" the address space, conflicting with subinterface addressing.

For router-on-a-stick to work, the **physical interface must have no IP address** — it is purely a trunk carrier. Only the subinterfaces get IP addresses.

### Resolution

```
R-Ballarat(config)# interface GigabitEthernet0/0
R-Ballarat(config-if)# no ip address
R-Ballarat(config-if)# no shutdown

R-Ballarat(config)# interface GigabitEthernet0/0.10
R-Ballarat(config-subif)# encapsulation dot1Q 10
R-Ballarat(config-subif)# ip address 10.7.9.1 255.255.255.240
```

### Verification

```
R-Ballarat# show ip interface brief
GigabitEthernet0/0         unassigned   YES unset    up   up
GigabitEthernet0/0.10      10.7.9.1     YES manual   up   up
GigabitEthernet0/0.20      10.7.9.17    YES manual   up   up
GigabitEthernet0/0.30      10.7.9.33    YES manual   up   up
GigabitEthernet0/0.40      10.7.9.49    YES manual   up   up
```

### Lesson Learned

Router-on-a-stick requires: **physical interface = no IP, subinterfaces = one IP each**.
Always remove the physical interface IP before creating subinterfaces.

---

## Issue 4 — Duplicate Management IP on SW-Core-Ballarat

**Severity:** Medium — network management disrupted  
**Phase:** Switch management configuration

### Symptom

Duplicate IP address conflict detected. Two devices responded to the same IP address, causing intermittent connectivity and Cisco IOS warnings:

```
%IP-4-DUPADDR: Duplicate address 10.7.9.X on GigabitEthernet...
```

### Root Cause

SW-Core-Ballarat was initially assigned a management IP that was already in use by another device on the same subnet. IP address assignment was not tracked centrally during initial configuration.

### Resolution

Reassigned SW-Core-Ballarat management IP to `10.7.9.46` — an unused address within the Administration /28 subnet range (10.7.9.32–10.7.9.47).

```
SW-Core-Ballarat(config)# interface vlan 40
SW-Core-Ballarat(config-if)# ip address 10.7.9.46 255.255.255.248
```

### Lesson Learned

Maintain a complete IP address allocation spreadsheet before and during deployment. Every address assignment must be recorded centrally. IP Address Management (IPAM) is a critical discipline in enterprise networking.

---

## Final Verification — 10/10 Connectivity Tests Passed

After all issues resolved:

| Test           | From → To                    | Result   |
|----------------|------------------------------|----------|
| Same-VLAN      | Rad WS → Rad Server          | ✅        |
| Inter-VLAN     | Cardiology → Radiology       | ✅        |
| Inter-VLAN     | Admin → Cardiology           | ✅        |
| ACL Block      | Admin → Radiology            | ✅ DENIED |
| WAN            | R-Ballarat → R-Ararat        | ✅        |
| WAN            | R-Ballarat → R-Horsham       | ✅        |
| Cross-site     | Ararat → Ballarat Radiology  | ✅        |
| Cross-site     | Horsham → Ballarat Radiology | ✅        |
| Spoke-to-spoke | Ararat → Horsham             | ✅        |
| Management     | IT → All subnets             | ✅        |
