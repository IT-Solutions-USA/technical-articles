# VMware Infrastructure Architecture Overview

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / Infrastructure  
**Tags:** VMware, vSphere, ESXi, VMotion, DRS, HA, VMFS, vCenter, Storage, Virtualisation, Data Centre

---

**Historical reference:** This article documents the VMware Infrastructure 3 / early vSphere architecture. Components referenced here — ESX Server, VirtualCenter Management Server, VI Client, VI Web Access, and VMware Consolidated Backup — are legacy products no longer available in current vSphere releases. ESX Server was replaced by ESXi, VirtualCenter by vCenter Server Appliance (VCSA), and VI Client by the HTML5-based vSphere Client. Consolidated Backup was removed in vSphere 6.0 (2015). This article is preserved as a reference for understanding pre-vSphere 5 environments and architecture concepts.

---

VMware Infrastructure is the foundational architecture upon which modern enterprise virtualisation is built. Understanding its physical and logical layers — how compute, storage, and networking resources are abstracted, pooled, and presented to virtual machines — is essential for anyone designing, securing, or auditing a virtualised data centre. This article walks through the complete architecture: from the physical building blocks of the data centre floor to the VirtualCenter management plane, covering every major component along the way.

---

## Physical Topology of the VMware Data Centre

A VMware Infrastructure data centre uses standard x86 hardware for its compute servers — no specialised server equipment required. Note that Fibre Channel SAN deployments do require specialised components: FC HBAs in each server, FC switches, and FC-capable storage arrays. The physical building blocks are consistent across deployments of any size, from a three-host lab to a multi-rack enterprise installation.

| Component | Description |
|---|---|
| Computing Servers | Industry-standard x86 servers running ESX Server on bare metal. Each server is a standalone Host in the virtual environment. Groups of similarly configured servers sharing the same network and storage subsystems form a Cluster — an aggregate pool of compute and memory resources presented as a single logical unit. |
| Storage Networks & Arrays | VMware Infrastructure supports Fibre Channel SAN, iSCSI SAN, and NAS arrays. Connecting storage to groups of servers through a storage area network allows aggregation of storage resources and flexible provisioning to virtual machines — independent of which physical host a VM runs on. |
| IP Networks | Each computing server can have multiple gigabit Ethernet NICs to provide high bandwidth, redundancy, and logical separation between management traffic, VM data traffic, storage traffic, and VMotion traffic. Network segmentation at this layer is a foundational security control. |
| VirtualCenter Management Server | The centralised management plane. VirtualCenter provides a single point of control for access management, performance monitoring, resource assignment, and configuration across all ESX Server installations in the data centre. Computing servers continue to run assigned VMs independently if VirtualCenter becomes unreachable. |

---

## Virtual Data Centre Architecture

VMware Infrastructure virtualises the entire IT infrastructure — servers, storage, and networks — and aggregates heterogeneous physical resources into a simple, uniform set of virtual elements. IT resources become a shared utility: dynamically provisioned to business units and projects without concern for the underlying hardware differences or physical locations.

The virtual data centre is built from six core elements:

| Element | Definition |
|---|---|
| Host | The virtual representation of a single physical machine's compute and memory resources. A server with four dual-core CPUs at 4 GHz each and 32 GB RAM presents as a Host with 32 GHz compute and 32 GB memory available for virtual machine allocation. |
| Cluster | The aggregate compute and memory resources of a group of physical servers sharing the same network and storage. Eight servers each with 32 GHz and 32 GB RAM form a Cluster with 256 GHz and 256 GB of pooled resources — managed as a single unit. |
| Resource Pool | A hierarchical mechanism for partitioning compute and memory resources from a Host or Cluster. Resource Pools can be nested — a Cluster can have a 'Finance Department' pool, which itself contains an 'Accounting' pool. Reservations are dynamic: unused reserved resources are available to sibling pools and reclaimed on demand. |
| Datastore | The virtual representation of physical storage resources — VMFS volumes spanning DAS, FC SAN, iSCSI SAN, or NAS. Each VM is stored as a set of files in its own Datastore directory. Virtual disks are files and can be copied, moved, backed up, and hot-added without powering down the VM. |
| Network | Virtual networks connect VMs to each other and to the physical network. A VM on Host1 and a VM on Host2 assigned to the same Port Group are on the same logical network regardless of which physical server they run on — enabling VMotion without network reconfiguration. |
| Virtual Machine | The fundamental workload unit. A VM consumes zero resources when powered off; dynamically scales resource consumption as workload increases; and can be provisioned in seconds — no purchase order, no physical constraints, no waiting. |

---

## Hosts, Clusters, and Resource Pools in Practice

Resource Pools are one of the most practically powerful — and most frequently misconfigured — elements of the virtual data centre.

Consider a Cluster of three servers, each with 4 GHz CPU and 16 GB RAM: the Cluster has 12 GHz and 48 GB available. A 'Finance Department' Resource Pool reserves 8 GHz and 32 GB. Within Finance, an 'Accounting' pool reserves 4 GHz and 16 GB, leaving the remainder for 'Payroll'. At year end, when Accounting's workload spikes, the Accounting pool's reservation can be raised to 6 GHz dynamically — no VM downtime, no hardware change.

The key behaviour: reservations are not wasted capacity. If Accounting is not using its 4 GHz reservation, Payroll can consume that headroom. When Accounting demands it back, Payroll releases it. This is resource pooling working correctly — dedicated allocation with shared utilisation efficiency.

---

## Distributed Services

Three distributed services build on the shared infrastructure foundation:

| Service | Tagline |
|---|---|
| VMotion | Live migration with zero downtime |
| DRS (Distributed Resource Scheduler) | Policy-driven automated workload balancing |
| HA (High Availability) | Automatic VM restart on host failure |

---

## Virtual Networking Architecture

VMware Infrastructure replicates the physical networking model in software — virtual NICs (vNICs), virtual switches (vSwitches), and Port Groups — and extends it with capabilities that are physically cost-prohibitive or impossible in traditional infrastructure.

Each virtual machine has one or more vNICs. To the guest OS, the vNIC appears identical to a physical NIC — it has a MAC address, one or more IP addresses, and responds to Ethernet protocol exactly as physical hardware does.

Port Groups are the key abstraction. A Port Group is a policy container — not just a network port. All VMs connected to the same Port Group are on the same logical network, regardless of which physical host they run on. This is what makes VMotion transparent: the VM reconnects to the same Port Group on the destination host and its network identity is unchanged.

| Policy | Description |
|---|---|
| Layer 2 security options | Promiscuous mode, MAC address changes, and forged transmits can be disabled per Port Group to isolate compromised or malicious VMs and prevent them from affecting other workloads on the same virtual switch. |
| VLAN segmentation | Port Groups can be configured with 802.1Q VLAN tagging to logically segment traffic across a shared physical switch infrastructure — enabling separate network segments for management, storage, production, and DMZ traffic on the same physical adapters. |
| NIC teaming policies | Each Port Group can specify its own NIC teaming policy — active/active load balancing, active/standby failover, or load-based teaming — independent of other Port Groups on the same vSwitch. |
| Traffic shaping | Average bandwidth, peak bandwidth, and burst size policies can be configured per Port Group to ensure that no single workload monopolises available physical bandwidth. |

---

## Storage Architecture and VMFS

VMware Infrastructure's storage architecture provides enterprise-class performance and availability by inserting an abstraction layer between the physical storage subsystem and the virtual machines. Applications and guest operating systems see only a standard SCSI disk — the complexity of the underlying storage technology (FC SAN, iSCSI, NAS, DAS) is invisible to the workload.

The central element is VMFS — the Virtual Machine File System. VMFS is a clustered file system designed specifically for shared storage environments.

| Feature | Description |
|---|---|
| Clustered file system | Multiple physical hosts can read and write to the same VMFS volume simultaneously. On-disk distributed locking prevents the same VM from being powered on by two hosts at the same time. |
| Distributed journaling | VMFS maintains enterprise-class crash consistency through distributed journaling, crash-consistent I/O paths, and machine state snapshots — enabling rapid root-cause analysis and recovery from host, storage, or VM failures. |
| Dynamic expansion | New LUNs added to any physical storage subsystem are automatically discovered and can be added to an existing Datastore without powering down servers or storage arrays. |
| Fault isolation | If a LUN within a Datastore fails, only VMs residing on that LUN are affected. All other VMs in the Datastore continue operating normally — a critical advantage over traditional volume-based storage allocation. |
| Raw Device Mapping (RDM) | Provides a mechanism for a VM to have direct access to a FC or iSCSI LUN. Required for SAN snapshot integrations and Microsoft Clustering Services (MSCS) spanning physical servers — where cluster data and quorum disks must be RDMs rather than VMFS files. |

---

## VMware Consolidated Backup

> **Deprecated:** VMware Consolidated Backup was deprecated in vSphere 5.0 and removed entirely in vSphere 6.0 (2015). Modern backup integration uses the vStorage APIs for Data Protection (VADP), which is supported by all current enterprise backup solutions. The architecture below is preserved for historical reference.

Consolidated Backup was VMware's agent-less backup solution for virtual machines. It operates without requiring a backup agent inside each virtual machine — instead, a third-party backup agent runs on a dedicated backup proxy server outside the ESX Server host.

The backup workflow: the third-party agent triggers Consolidated Backup, which runs pre-backup scripts to quiesce virtual disks and capture snapshots. It then mounts the snapshot to the backup proxy server. The third-party agent backs up the mounted snapshot to its backup targets (disk or tape). Post-thaw scripts restore the VM to normal operation. The result is a low-overhead, non-intrusive backup that does not require backup windows or in-guest quiescing.

For large environments, SAN-offloaded backup leverages RDM (Raw Device Mapping) to move data directly over the SAN fabric to backup targets — bypassing the LAN entirely and eliminating backup traffic from the production network.

---

## VirtualCenter Management Server Core Services

| Service | Description |
|---|---|
| VM Provisioning | Automates and guides the creation and deployment of virtual machines, including template-based deployments. |
| Host & VM Configuration | Centralised management of ESX Server host configuration and individual VM settings — eliminating the need to manage each host directly. |
| Resource & VM Inventory Management | Organises all virtual machines, hosts, clusters, datastores, and networks in a logical hierarchy — the foundation for access control and policy assignment. |
| Statistics & Logging | Logs and reports performance statistics and resource utilisation for VMs, hosts, and clusters — stored in Oracle or Microsoft SQL Server for historical analysis in legacy Windows-based VirtualCenter deployments. Modern vCenter Server Appliance (VCSA) uses an embedded PostgreSQL database; external Oracle/SQL Server databases are no longer supported as of vCenter 7.0. |
| Alarms & Event Management | Tracks conditions such as resource over-utilisation, host failures, datastore capacity thresholds, and generates alerts for configured escalation paths. |
| Task Scheduler | Schedules administrative actions — including VMotion migrations, snapshots, and policy changes — for execution at defined times or intervals. |

---

## Accessing the Virtual Data Centre

VirtualCenter Management Server supports three access methods, each suited to different administrative use cases. All three authenticate through VirtualCenter's User Access Control layer, which maps Active Directory identities to role-based permissions over specific resource scopes.

| Access Method | Description |
|---|---|
| VI Client | A Windows desktop application that connects to either VirtualCenter Management Server or directly to individual ESX Server hosts. Provides full management functionality — host and VM configuration, resource management, console access, and performance monitoring. The primary interface for vSphere administrators. |
| VI Web Access | Browser-based interface for virtual machine management and console access. VI Web Access mediates communication between the browser and VirtualCenter through the VI API and resolves the physical location of the target VM to establish a direct console connection to the hosting ESX Server. |
| Terminal Services | Standard terminal access tools (Windows Terminal Services, Xterm) provide VM console access through the same channel used to access physical machines — no VMware-specific client required for basic console connectivity. |
