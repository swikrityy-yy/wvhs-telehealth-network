# Routing Design Document

## Western Victoria Health Services — NIT1104 Group 9

---

## Routing Protocol Decision: Static Routing

### Why Not Dynamic Routing (OSPF / EIGRP / RIP)?

| Criterion                | Static Routing     | OSPF                        | RIP                    |
|--------------------------|--------------------|-----------------------------|------------------------|
| Network size suitability | 2–10 routers       | 10–1000+ routers            | 2–15 routers           |
| Topology changes         | Manual update      | Auto-discovers              | Auto-discovers         |
| WAN bandwidth overhead   | Zero               | ~10% (Hello + LSA)          | ~5% (periodic updates) |
| Convergence time         | Instant (manual)   | Seconds                     | 30–180 seconds         |
| CPU/Memory usage         | Negligible         | Moderate (SPF calc)         | Low                    |
| Predictability           | 100% deterministic | May reroute                 | May loop               |
| Troubleshooting          | Read the table     | Requires protocol knowledge | Simple but limited     |

**Decision:** Static routing. Rationale:

1. Exactly 3 routers in a fixed topology — complexity of dynamic protocols adds zero value
2. Hub-and-spoke means routes are entirely predictable at design time
3. Zero protocol overhead on WAN serial links — important for bandwidth-constrained circuits
4. Healthcare requires deterministic routing — a routing protocol reconvergence during a topology change could briefly route patient data unexpectedly
5. Static routes are instantly verifiable with `show ip route`

---

## Routing Tables

### R-Ballarat Routing Table (expected output of `show ip route`)

```
Codes: C - connected, S - static

C    10.7.9.0/28    is directly connected, GigabitEthernet0/0.10
C    10.7.9.16/29   is directly connected, GigabitEthernet0/0.20
C    10.7.9.32/28   is directly connected, GigabitEthernet0/0.30
C    10.7.9.48/29   is directly connected, GigabitEthernet0/0.40
C    10.7.9.112/30  is directly connected, Serial0/0/0
C    10.7.9.116/30  is directly connected, Serial0/0/1
S    10.7.9.64/29   [1/0] via 10.7.9.114
S    10.7.9.72/29   [1/0] via 10.7.9.114
S    10.7.9.80/28   [1/0] via 10.7.9.118
S    10.7.9.96/29   [1/0] via 10.7.9.118
```

### R-Ararat Routing Table

```
C    10.7.9.64/29   is directly connected, GigabitEthernet0/0
C    10.7.9.72/29   is directly connected, GigabitEthernet0/1
C    10.7.9.112/30  is directly connected, Serial0/0/0
S    10.7.9.0/28    [1/0] via 10.7.9.113
S    10.7.9.16/29   [1/0] via 10.7.9.113
S    10.7.9.32/28   [1/0] via 10.7.9.113
S    10.7.9.48/29   [1/0] via 10.7.9.113
S    10.7.9.80/28   [1/0] via 10.7.9.113
S    10.7.9.96/29   [1/0] via 10.7.9.113
S    10.7.9.116/30  [1/0] via 10.7.9.113
```

### R-Horsham Routing Table

```
C    10.7.9.80/28   is directly connected, GigabitEthernet0/0
C    10.7.9.96/29   is directly connected, GigabitEthernet0/1
C    10.7.9.116/30  is directly connected, Serial0/0/0
S    10.7.9.0/28    [1/0] via 10.7.9.117
S    10.7.9.16/29   [1/0] via 10.7.9.117
S    10.7.9.32/28   [1/0] via 10.7.9.117
S    10.7.9.48/29   [1/0] via 10.7.9.117
S    10.7.9.64/29   [1/0] via 10.7.9.117
S    10.7.9.72/29   [1/0] via 10.7.9.117
S    10.7.9.112/30  [1/0] via 10.7.9.117
```

---

## Cross-Site Packet Path (Horsham Clinical → Ballarat Radiology)

```
Horsham PC (10.7.9.82)
    └→ default gateway 10.7.9.81 (R-Horsham Gi0/0)
         └→ static route: 10.7.9.0/28 via 10.7.9.117
              └→ WAN Serial link (10.7.9.116/30)
                   └→ R-Ballarat Serial0/0/1 (10.7.9.117)
                        └→ directly connected: Gi0/0.10 → VLAN 10
                             └→ trunk → SW-Core-Ballarat → SW-Radiology
                                  └→ Radiology Server (10.7.9.5) ✓
```

Return path is symmetric — R-Ballarat has static route to 10.7.9.80/28 via 10.7.9.118.
