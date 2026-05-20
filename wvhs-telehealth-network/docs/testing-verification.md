# Testing & Verification Report

## Western Victoria Health Services — NIT1104 Group 9

---

## Test Methodology

All tests performed using Cisco Packet Tracer simulation.  
Testing sequence follows OSI model — Layer 1 verified before Layer 3 tests.

**Verification commands used:**

- `ping` — basic connectivity (5 ICMP echo requests, reports success rate)
- `traceroute` — path verification (shows each hop)
- `show ip interface brief` — interface status
- `show ip route` — routing table verification
- `show vlan brief` — VLAN membership
- `show interfaces trunk` — trunk configuration
- `show access-lists` — ACL match counters

---

## Test Results

### Test 1 — Same-VLAN Communication (Radiology)

**Source:** Rad-WS1 (10.7.9.2, VLAN 10)  
**Destination:** Rad-ImagingServer1 (10.7.9.5, VLAN 10)  
**Expected:** Success — same VLAN, Layer 2 forwarding only  
**Result:** ✅ PASS — 5/5 packets received

```
Rad-WS1> ping 10.7.9.5
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/1 ms
```

### Test 2 — Inter-VLAN Routing (Cardiology → Radiology)

**Source:** Card-WS1 (10.7.9.18, VLAN 20)  
**Destination:** Rad-ImagingServer1 (10.7.9.5, VLAN 10)  
**Expected:** Success — different VLANs, routed via R-Ballarat subinterfaces  
**Result:** ✅ PASS — 5/5 packets received

```
Card-WS1> ping 10.7.9.5
!!!!!
```

**Traceroute confirms routing via R-Ballarat:**

```
Card-WS1> tracert 10.7.9.5
1   10.7.9.17  (R-Ballarat VLAN20 gateway)   1ms
2   10.7.9.5   (Radiology server)             1ms
```

### Test 3 — Inter-VLAN Routing (Admin → Cardiology)

**Source:** Admin-PC1 (10.7.9.34, VLAN 30)  
**Destination:** Card-ECGServer (10.7.9.20, VLAN 20)  
**Expected:** Success — ACL 100 only blocks Admin→Radiology, not Admin→Cardiology  
**Result:** ✅ PASS — 5/5 packets received

### Test 4 — ACL Block Test (Admin → Radiology) ⭐ Security Test

**Source:** Admin-PC1 (10.7.9.34, VLAN 30)  
**Destination:** Rad-ImagingServer1 (10.7.9.5, VLAN 10)  
**Expected:** DENIED — ACL 100 rule 1 blocks 10.7.9.32/28 → 10.7.9.0/28  
**Result:** ✅ PASS (correctly denied)

```
Admin-PC1> ping 10.7.9.5
.....
Success rate is 0 percent (0/5)
```

**ACL counter confirmed:**

```
R-Ballarat# show access-lists
Extended IP access list 100
    10 deny ip 10.7.9.32 0.0.0.15 10.7.9.0 0.0.0.15 (5 matches)
    20 permit ip any any (...)
```

### Test 5 — WAN Link: Ballarat to Ararat

**Source:** R-Ballarat (10.7.9.113)  
**Destination:** R-Ararat (10.7.9.114)  
**Expected:** Success — WAN serial link operational with clock rate  
**Result:** ✅ PASS

```
R-Ballarat# ping 10.7.9.114
!!!!!
```

### Test 6 — WAN Link: Ballarat to Horsham

**Source:** R-Ballarat (10.7.9.117)  
**Destination:** R-Horsham (10.7.9.118)  
**Expected:** Success  
**Result:** ✅ PASS

### Test 7 — Cross-Site: Ararat Clinical → Ballarat Radiology

**Source:** Ararat-Clinical-PC1 (10.7.9.66)  
**Destination:** Rad-ImagingServer1 (10.7.9.5)  
**Expected:** Success — static routes via R-Ballarat hub  
**Result:** ✅ PASS

```
Traceroute:
1  10.7.9.65   (R-Ararat LAN gateway)     1ms
2  10.7.9.113  (R-Ballarat WAN end)        2ms
3  10.7.9.5    (Radiology server)          2ms
```

### Test 8 — Cross-Site: Horsham Clinical → Ballarat Radiology

**Source:** Horsham-Clinical-PC1 (10.7.9.82)  
**Destination:** Rad-ImagingServer1 (10.7.9.5)  
**Expected:** Success  
**Result:** ✅ PASS — confirms telehealth imaging access works end-to-end

### Test 9 — Spoke-to-Spoke: Ararat → Horsham (via Ballarat hub)

**Source:** Ararat-Clinical-PC1 (10.7.9.66)  
**Destination:** Horsham-Clinical-PC1 (10.7.9.82)  
**Expected:** Success — routed through Ballarat (no direct Ararat–Horsham link)  
**Result:** ✅ PASS

```
Traceroute:
1  10.7.9.65   (R-Ararat)
2  10.7.9.113  (R-Ballarat WAN - Ararat end)
3  10.7.9.118  (R-Horsham WAN end) ← packet handed off
4  10.7.9.82   (Horsham PC)
```

Confirms hub-and-spoke routing works correctly — Ararat–Horsham traffic routes via Ballarat.

### Test 10 — IT Infrastructure: Network-Wide Management Access

**Source:** IT-WS1 (10.7.9.50, VLAN 40)  
**Destinations:** All 10 subnets  
**Expected:** Success — IT has no ACL restrictions, full access for management  
**Result:** ✅ PASS — all 10 destination subnets reachable from IT workstation

---

## Summary

| Category             | Tests  | Passed | Failed |
|----------------------|--------|--------|--------|
| Same-site same-VLAN  | 1      | 1      | 0      |
| Inter-VLAN routing   | 2      | 2      | 0      |
| ACL security (block) | 1      | 1      | 0      |
| WAN connectivity     | 2      | 2      | 0      |
| Cross-site routing   | 2      | 2      | 0      |
| Spoke-to-spoke       | 1      | 1      | 0      |
| Management access    | 1      | 1      | 0      |
| **TOTAL**            | **10** | **10** | **0**  |

**All 10/10 connectivity tests passed. Network is fully operational.**
