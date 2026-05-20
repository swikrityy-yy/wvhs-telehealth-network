# Network Topology Documentation
## Western Victoria Health Services — NIT1104 Group 9

---

## Logical Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    BALLARAT — Central Diagnostic Hub            │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ VLAN 10  │  │ VLAN 20  │  │ VLAN 30  │  │ VLAN 40  │       │
│  │Radiology │  │Cardiology│  │  Admin   │  │    IT    │       │
│  │10.7.9.0/2│  │10.7.9.16 │  │10.7.9.32 │  │10.7.9.48 │       │
│  │    8     │  │   /29    │  │   /28    │  │   /29    │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │              │              │             │
│  ┌────▼─────────────▼──────────────▼──────────────▼─────┐      │
│  │              SW-Core-Ballarat (trunk)                 │      │
│  └─────────────────────────┬─────────────────────────────┘      │
│                            │ 802.1Q trunk (VLAN 10,20,30,40)   │
│                     ┌──────▼──────┐                            │
│                     │  R-Ballarat  │ ← ACL 100 (Admin→Rad DENY) │
│                     │  Cisco 2911  │                            │
│                     └──────┬──┬───┘                            │
└────────────────────────────┼──┼────────────────────────────────┘
                             │  │
          Serial0/0/0        │  │ Serial0/0/1
          10.7.9.112/30      │  │  10.7.9.116/30
                    ┌────────┘  └────────┐
                    │                    │
          ┌─────────▼──────┐   ┌─────────▼──────┐
          │   R-Ararat      │   │   R-Horsham     │
          │   Cisco 2911    │   │   Cisco 2911    │
          └─────────┬──┬───┘   └─────────┬──┬───┘
                    │  │                  │  │
               ┌────┘  └───┐        ┌────┘  └───┐
               ▼           ▼        ▼           ▼
          Clinical     Reception  Clinical    Reception
         10.7.9.64/29 10.7.9.72/29 10.7.9.80/28 10.7.9.96/29
```

---

## Physical Layer Topology

### Ballarat — Central Diagnostic Hub

```
[Rad-WS1]──────┐
[Rad-WS2]──────┤
[Rad-Server1]──┤──[SW-Radiology]──Gi──[SW-Core-Ballarat]
[Rad-Server2]──┤                              │
[PACS-System]──┘                              │ (trunk: VLANs 10,20,30,40)
                                              │
[Card-WS1]─────┐                             │
[Card-WS2]─────┤──[SW-Cardiology]──Gi─────────┤
[Card-ECGSrv]──┘                             │
                                              │
[Admin-PC1]────┐                             │
[Admin-PC2]────┤                             │
[Admin-PC3]────┤──[SW-Admin]──────Gi─────────┤
[Admin-Print1]─┤                             │
[Admin-Print2]─┤                             │
[Admin-FileSrv]┘                             │
                                              │
[IT-WS1]───────┐                             │
[IT-WS2]───────┴──[SW-IT]────────Gi─────────┘
                                              │
                                      [R-Ballarat]
                                      Gi0/0 (trunk, no IP)
                                      ├── .10  10.7.9.1/28
                                      ├── .20  10.7.9.17/29
                                      ├── .30  10.7.9.33/28  ← ACL 100 in
                                      └── .40  10.7.9.49/29
                                      Se0/0/0 10.7.9.113/30 (DCE, clk 64000)
                                      Se0/0/1 10.7.9.117/30 (DCE, clk 64000)
```

### Ararat — Medical Clinic

```
[Ararat-Clin-PC1]──┐
[Ararat-Clin-PC2]──┤──[SW-Core-Ararat]──Gi──[R-Ararat]
[Ararat-NurseStn]──┘              │           Gi0/0: 10.7.9.65/29 (Clinical)
                                  │           Gi0/1: 10.7.9.73/29 (Reception)
[Ararat-Rec-PC1]───┐              │           Se0/0/0: 10.7.9.114/30 (DTE)
[Ararat-Rec-PC2]───┴──────────────┘
```

### Horsham — Medical Clinic

```
[Horsham-Clin-PC1]──┐
[Horsham-Clin-PC2]──┤──[SW-Core-Horsham]──Gi──[R-Horsham]
[Horsham-NurseStn]──┘               │           Gi0/0: 10.7.9.81/28 (Clinical)
                                    │           Gi0/1: 10.7.9.97/29 (Reception)
[Horsham-Rec-PC1]───┐               │           Se0/0/0: 10.7.9.118/30 (DTE)
[Horsham-Rec-PC2]───┴───────────────┘
```

---

## OSI Layer Mapping

| Layer | Technology | Implementation |
|-------|-----------|----------------|
| 7 Application | Healthcare applications | EHR systems, PACS, telehealth video |
| 4 Transport | TCP/UDP | Patient data sessions |
| 3 Network | IPv4, Static Routing, ACLs | R-Ballarat, R-Ararat, R-Horsham |
| 2 Data Link | Ethernet, 802.1Q VLANs | All 9 switches, trunk links |
| 1 Physical | Copper, Serial | Straight-through, crossover, serial cables |
