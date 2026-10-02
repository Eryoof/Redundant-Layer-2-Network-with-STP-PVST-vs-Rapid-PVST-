# STP-Redundancy-Lab
A Cisco Packet Tracer lab that builds a redundant Access/Distribution switch topology, places the STP root bridge manually, and compares link-failure behavior under classic PVST+ and Rapid PVST+.

## Objectives

- Build a Layer 2 network with intentional loops for redundancy.
- Control which switch becomes the root bridge (instead of leaving it to the lowest MAC address).
- Simulate a primary-link failure and verify the network recovers through the backup path.
- Compare lost pings under PVST+ and Rapid PVST+.

## Topology

- 2 Distribution switches: `DS1`, `DS2` (Cisco 2960-24TT)
- 2 Access switches: `AS1`, `AS2` (Cisco 2960-24TT)
- 4 PCs: PC0 and PC1 on AS1, PC2 and PC3 on AS2
- All PCs are in the same subnet (`192.168.1.0/24`, VLAN 1). No router and no ACLs: the lab focuses on Layer 2.

```
        DS1 ───────── DS2
        │ ╲         ╱ │
        │   ╲     ╱   │
        │     ╲ ╱     │
        │     ╱ ╲     │
        │   ╱     ╲   │
        AS1         AS2
       /  \        /  \
    PC0  PC1    PC3  PC2
```

### Cabling (copper straight-through)

| From | To |
|---|---|
| DS1 Fa0/1 | DS2 Fa0/1 |
| DS1 Fa0/2 | AS1 Fa0/1 |
| DS1 Fa0/3 | AS2 Fa0/1 |
| DS2 Fa0/2 | AS1 Fa0/2 |
| DS2 Fa0/3 | AS2 Fa0/2 |
| AS1 Fa0/3 | PC0 |
| AS1 Fa0/4 | PC1 |
| AS2 Fa0/3 | PC3 |
| AS2 Fa0/4 | PC2 |

## Configuration

**Root bridge placement** (VLAN 1):

```
! DS1 - primary root
hostname DS1
spanning-tree vlan 1 root primary

! DS2 - backup root
hostname DS2
spanning-tree vlan 1 root secondary
```

**Rapid PVST+** (applied on all four switches after the PVST+ baseline test):

```
configure terminal
spanning-tree mode rapid-pvst
end
```

Access switches only needed `hostname AS1` / `hostname AS2`.

## What I observed

1. **Default behavior (no root configured):** the root bridge was elected by lowest MAC address, and it turned out to be an access switch (AS1).
2. **After manual root placement:** DS1 became the root (`This bridge is the root`) and DS2 the backup (bridge priority 28673).
3. **Blocked ports:** on AS1 and AS2, `Fa0/2` (the link toward DS2) was in the `Altn BLK` state: the redundant path, kept ready but not forwarding.

## Failover test

Method:

1. Start a continuous ping from PC0 to PC2: `ping -n 100 192.168.1.3`
2. While the ping runs, shut down the primary uplink on AS1: `interface fastethernet0/1` then `shutdown`
3. Record the ping summary and check `show spanning-tree` on AS1.

Result of the shutdown: AS1's `Fa0/2` changed from `Altn BLK` to `Root FWD` with root path cost 38 (two links: AS1 to DS2, then DS2 to DS1). Traffic continued over the backup path without any manual change.

### Results

| STP mode | Pings sent | Pings lost (run 1) | Pings lost (run 2) |
|---|---|---|---|
| PVST+ (default) | 100 | 5 | 5 |
| Rapid PVST+ | 100 | 0 | 0 |

## Notes and limitations

- Results come from the Packet Tracer simulator, not physical hardware, so timings are only comparable inside the simulation.
- The test counts lost pings (about one per second), not exact seconds of downtime. `Lost = 0` means the outage was shorter than the ping interval, not that no failover happened.
- Only VLAN 1 was used. VLAN segmentation and ACLs are covered in my separate VLAN/ACL lab.
- Two runs per mode were measured.

## Files

- `STP-redundancy.pkt`: Packet Tracer project
- `images/`: topology and `show spanning-tree` screenshots (before and after failover)
