# Cisco Nexus 1000V Port Channels and Trunking: Troubleshooting Guide

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Cisco / Nexus  
**Tags:** Cisco, Nexus 1000V, Port Channel, VLAN Trunking, LACP, APC, VMware, VEM, VSM, Network Troubleshooting

> **End-of-Life Notice:** Cisco announced End-of-Sale for the Nexus 1000V for VMware vSphere in 2019 and End-of-Support in 2025. This product should not be deployed in new environments. Existing deployments should have a migration plan to a supported virtual switching solution. This article is preserved as a reference for managing and troubleshooting legacy Nexus 1000V installations.

---

Port channel and trunking failures are among the most impactful issues in a virtual switching environment — they affect multiple VMs simultaneously, can cause subtle traffic imbalances that only surface under load, and are often misdiagnosed because the logical link appears up even when member interfaces have failed. This guide covers the Cisco Nexus 1000V port channel and trunking architecture, the complete diagnostic command set, asymmetric port channel (APC) troubleshooting, and a fault table for the most common failure scenarios.

---

## Port Channel Overview

A port channel aggregates multiple physical interfaces into a single logical interface. From the perspective of the upper-layer protocols, the spanning tree, and the MAC address table, a port channel is one link — regardless of how many physical members it contains.

| Function | Description |
|---|---|
| Bandwidth aggregation | Distributes traffic across all functional links in the channel, increasing effective throughput beyond what any single physical interface can provide. |
| Load balancing | Hashes traffic flows across member links based on configurable criteria (src/dst MAC, src/dst IP, src/dst port). Optimises bandwidth utilisation by preventing all traffic from traversing a single link. |
| Link redundancy | If one member link fails, traffic is redistributed to remaining links transparently. The upper-layer protocol sees an uninterrupted logical link — only reduced bandwidth, no link-down event, no MAC table flush. |

**Important:** When a member link fails, the upper protocol does not receive a link-down notification. Silent link failures within a port channel can go undetected for extended periods if monitoring is not explicitly configured for member interface state.

---

## Trunking Overview

VLAN trunking enables an interconnected port to transmit and receive frames across multiple VLANs over a single physical or logical link. Trunking and port channeling are complementary but independent features — a port channel can carry trunked VLAN traffic, or a single physical interface can trunk without being part of a channel.

On the Nexus 1000V, trunk configuration is managed through port profiles. The key operational distinction: the **configured allowed VLAN list** and the **active allowed VLAN list** can differ. A VLAN that is in the allowed list but has not been created on the switch will be pruned from the active trunking set. `show vlan internal trunk` reveals the active set; the running configuration shows only the configured allowed list.

---

## Diagnostic Commands

| Command | Description |
|---|---|
| `show port-channel summary` | Primary overview command. Shows all port channels, their state flags (Up/Down, LACP/NONE), protocol, and member port status. |
| `show port-channel internal event-history interface port-channel <n>` | Event history for a specific port channel interface. Shows state transitions, LACP PDU events, and error conditions — essential for diagnosing intermittent flapping or failed formation. |
| `show port-channel internal event-history interface ethernet <slot/port>` | Event history at the member interface level. Cross-reference with port channel event history to identify whether a failure originated at the member or channel level. |
| `show system internal ethpm event-history interface port-channel <n>` | Ethernet Port Manager (ethpm) event history for the logical port channel. Reveals internal state machine transitions not visible in the operational show commands. |
| `show system internal ethpm event-history interface ethernet <slot/port>` | ethpm event history at the physical member interface level. Critical for diagnosing interface-level failures that prevented bundle formation. |
| `show vlan internal trunk interface ethernet <slot/port>` | Shows the internal trunking state for a physical interface — allowed VLAN list, native VLAN, and active VLAN set. Use to verify whether VLANs are being blocked at the member level. |
| `show vlan internal trunk interface port-channel <n>` | Same as above but at the port channel (logical) level. Compare against the member-level output to identify VLAN inheritance or policy mismatches between the bundle and its members. |
| `module vem <n> execute vemcmd show port` | Runs directly on the VEM (Virtual Ethernet Module). Shows physical port state from the hypervisor module's perspective — useful when the VSM shows a port as up but traffic is not passing. |
| `module vem <n> execute vemcmd show pc` | Shows port channel state from the VEM perspective, including subgroup assignments for APC configurations. Use after resolving APC subgroup issues to confirm the VEM has applied the correct grouping. |
| `module vem <n> execute vemcmd show trunk` | Shows trunk state from the VEM module. Cross-reference with VSM trunk output to identify inconsistencies between the control plane and data plane. |
| `debug port-channel error` | Enables real-time error logging for port channel operations. Use in a maintenance window or lab — the output volume can be significant on a busy switch. |
| `debug port-channel trace` | Traces port channel state machine transitions in real time. Particularly useful when diagnosing APC subgroup assignment failures or LACP PDU negotiation issues. |

---

## Pre-Formation Checklist

Before troubleshooting a port channel failure, verify these prerequisites:

| Check | Detail |
|---|---|
| Run `show port-channel compatibility-parameters` | Before adding an interface to a port channel, verify the compatibility parameters. Mismatched speed, duplex, flow control, or port mode settings prevent bundle formation silently — the interface stays individual rather than joining the channel. |
| All LACP member interfaces go to the same destination device | For LACP channels, every member must connect to the same upstream device. LACP PDU exchange requires symmetric termination. Asymmetric Nexus 1000V deployments connecting to two different upstream switches require APC (ON mode only, not LACP). |
| Equal member count on both sides | Both ends of a port channel must have the same number of active member interfaces. A 4-member bundle on the Nexus 1000V connecting to a 2-member bundle on the upstream switch will not form correctly — verify symmetry before troubleshooting further. |
| Matching interface types on both sides | 1G interfaces cannot bundle with 10G interfaces. Each member on the Nexus 1000V must connect to a matching interface type on the upstream device. |
| All required VLANs in the trunk allowed list | A common trunking failure: the VLAN exists on both devices but is absent from the allowed VLAN list on the trunk. Use `show vlan internal trunk` to verify the effective allowed list — it may differ from the configured list if VLANs have not been created. |
| All member interfaces on the same module | On the Nexus 1000V, all member interfaces in a port channel must reside on the same VEM module. Interfaces across different modules cannot be bundled. |
| Port channel configuration present in the port profile | The Nexus 1000V uses port profiles to push configuration to interfaces. If the port channel configuration (channel-group, mode, etc.) is not in the port profile used by the physical ports, the interfaces will not join the bundle even if manually configured at the interface level. |
| Configure APC for ports connected to different upstream switches | If physical ports connect to two different upstream switches, standard port channel formation will fail. Configure Asymmetric Port Channel (APC) mode. APC is supported on ON mode channels only — not LACP. |
| APC: maximum two ports per group when upstream doesn't support port channels | If the upstream switch does not support port channels at all, configure APC with a maximum of two ports in the APC group. Exceeding this limit will prevent ports from coming online. |

---

## Troubleshooting Asymmetric Port Channels (APC)

Asymmetric Port Channel is a Nexus 1000V-specific feature that enables ports within a single port channel group to connect to two different upstream switches. Standard port channel operation requires all members to terminate on the same upstream device — APC removes this restriction for ON mode channels.

APC uses Cisco Discovery Protocol (CDP) to automatically assign subgroup IDs to physical ports based on which upstream switch they are connected to. This means CDP is not optional — it is a functional dependency. If CDP is disabled on the VSM, on any VEM module, or on the upstream switches, APC subgroup assignment will fail silently and ports will not come online.

---

## Fault Table

| Symptom | Cause | Solution |
|---|---|---|
| Cannot create port channel | Maximum port channel limit reached | The Nexus 1000V supports a maximum of 256 port channels (limit may vary by software release — verify with your release notes). Run `show port-channel summary` to count existing channels. Remove unused or orphaned port channels before creating new ones. |
| Newly added interface does not come online | Port channel configuration not in port profile | Verify the port channel configuration is present in the port profile (port group) used by the interface. Run `show port-profile name <profile>` and confirm channel-group settings are present. |
| Newly added interface does not come online | Port channel mode is ON and configurations differ | Check if a port channel already exists on the module using the same port profile. Compare running configuration on the port channel and the new interface. If configurations differ, apply the delta to the new interface, then remove and re-add it to the bundle. |
| Newly added interface does not come online | Interface parameters incompatible with existing bundle members | Use the `channel-group <n> force` command to force the interface to adopt the port channel's parameters. Use only when manually configuring — not when using port profiles. |
| VLAN traffic does not traverse trunk | VLAN not in the allowed VLAN list | Add the VLAN to the allowed list with: `switchport trunk allowed vlan add <vlan-id>`. Apply in the port profile used by the interface to ensure the configuration persists across module reloads. |

---

## Fixing VLAN Traffic Not Traversing a Trunk

The most frequent trunking issue is a VLAN being absent from the allowed list. The fix is straightforward, but the application point matters: the command must be applied in the port profile used by the interface, not just at the interface level. Interface-level changes on the Nexus 1000V are overridden by the port profile on the next reconciliation.

```
switchport trunk allowed vlan add <vlan-id>
```

Apply this in the port profile, then verify with `show vlan internal trunk interface ethernet <slot/port>` that the VLAN appears in both the configured and active allowed VLAN sets.
