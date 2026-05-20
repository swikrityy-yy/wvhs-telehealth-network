# Packet Tracer File
## Western Victoria Health Services — NIT1104 Group 9

---

## File

`Group9_NIT1104_Assesment3.pkt`

Add the `.pkt` file to this folder after downloading from the course submission.  
(The file is not included in this repository to respect academic integrity guidelines.)

---

## Requirements

- **Cisco Packet Tracer** version 8.x or later
- Free download with a NetAcad account: https://www.netacad.com/courses/packet-tracer

---

## How to Open and Explore

1. Download and install Packet Tracer
2. Open `Group9_NIT1104_Assesment3.pkt`
3. The network opens in **Logical View** — you'll see all three sites

---

## Key Things to Explore

### 1. Check router configurations
Click any router → CLI tab → type:
```
enable
show running-config
show ip route
show ip interface brief
```

### 2. Verify VLAN configuration
Click SW-Core-Ballarat → CLI tab:
```
enable
show vlan brief
show interfaces trunk
```

### 3. Test connectivity
Click any PC → Desktop → Command Prompt:
```
ping 10.7.9.5       (test to Ballarat Radiology from any device)
tracert 10.7.9.5    (see the full hop-by-hop path)
```

### 4. Test ACL block
Click Admin-PC1 (10.7.9.34) → Desktop → Command Prompt:
```
ping 10.7.9.5       (should FAIL — blocked by ACL 100)
ping 10.7.9.17      (should SUCCEED — Admin→Cardiology allowed)
```

### 5. Check ACL counters
Click R-Ballarat → CLI:
```
enable
show access-lists
```
After the ping test above, the deny rule should show match counters.

### 6. Use Simulation Mode
- Click the clock icon (bottom right) to enter Simulation Mode
- Add a PDU (Simple or Complex) between any two devices
- Click Play to watch packets travel hop by hop
- Inspect each packet's headers at each device

---

## Site Layout

| Area in Packet Tracer     | Location                           |
|---------------------------|------------------------------------|
| Left section              | Ararat Medical Clinic              |
| Centre section            | Ballarat Central Diagnostic Hub    |
| Right section             | Horsham Medical Clinic             |
| Red WAN links             | Serial connections between routers |
| Blue/green internal links | LAN Ethernet connections           |
