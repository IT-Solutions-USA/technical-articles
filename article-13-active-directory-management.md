# Active Directory Management: Domain Controllers, FSMO Roles, Replication, DNS & Best Practices

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Microsoft / Active Directory  
**Tags:** Active Directory, Domain Controller, FSMO, DNS, DHCP, Kerberos, Group Policy, RODC, Replication, KCC, SYSVOL, Microsoft, Windows Server, Identity

---

Active Directory is the identity and access backbone of nearly every Windows-based enterprise network. Understanding its architecture — how Domain Controllers work, how the directory replicates, what the FSMO roles do, and how DNS underpins the entire system — is a prerequisite for designing it correctly, troubleshooting it effectively, and securing it against the identity-based attacks that target it relentlessly.

---

## Domain Controller Types

| Type | Description |
|---|---|
| **Read/Write Domain Controller** | Accepts changes to the Active Directory database and SYSVOL from domain members. Replicates those changes outward to other Domain Controllers. The standard DC type for production sites. |
| **Read-Only Domain Controller (RODC)** | Holds a read-only copy of the AD database. Changes cannot be written to an RODC directly — they are pushed to it by Read/Write DCs. Designed for branch offices and locations where physical security cannot be guaranteed: a compromised RODC cannot be used to poison the directory. |

---

## FSMO Roles

| Role | Scope |
|---|---|
| PDC Emulator | Domain-wide |
| RID Pool Master | Domain-wide |
| Infrastructure Master | Domain-wide |
| Schema Master | Forest-wide |
| Domain Naming Master | Forest-wide |

Query FSMO role holders with:

```cmd
C:\> netdom query fsmo

Schema master               <dc-hostname>.<domain>
Domain naming master        <dc-hostname>.<domain>
PDC                         <dc-hostname>.<domain>
RID pool manager            <dc-hostname>.<domain>
Infrastructure master       <dc-hostname>.<domain>

The command completed successfully.
```

---

## Replication

| Type | Description |
|---|---|
| **Intrasite Replication** | Within an Active Directory site, replication uses pull-based notification. When a DC receives a change, it notifies its replication partners. After a brief delay, the partners pull the changes. To minimise network chatter, intrasite replication uses a bidirectional ring topology by default — the KCC connects each DC to at most two replication partners per naming context, so changes propagate around the ring rather than flooding every DC directly. The Knowledge Consistency Checker (KCC) builds and maintains this topology automatically. |
| **Intersite Replication** | Between Active Directory sites, replication is schedule-based rather than notification-based. After the default schedule interval (180 minutes / 3 hours by default), a bridgehead DC at one site requests changes from the bridgehead DC at the other site. The receiving bridgehead then propagates changes to the other DCs in its site using intrasite replication. Intersite replication compresses data to reduce WAN bandwidth consumption. |

---

## DNS Record Types

Active Directory depends entirely on DNS for service location. Domain-joined clients locate Domain Controllers, Global Catalogue servers, Kerberos KDCs, and LDAP servers through SRV records registered automatically at DC promotion.

| Record | Description |
|---|---|
| A | Maps a hostname to an IPv4 address. The primary record type in Forward Lookup Zones. |
| AAAA | Maps a hostname to an IPv6 address. Used alongside A records in dual-stack environments. |
| SRV | Service location record. AD-integrated DNS zones use SRV records extensively so that domain-joined clients can locate Domain Controllers, Global Catalogue servers, Kerberos KDCs, and LDAP servers automatically. |
| PTR | Pointer record in a Reverse Lookup Zone. Maps an IP address back to a hostname. Used in logging, monitoring, and some authentication validation flows. |
| CNAME | Canonical name alias record. Points one hostname to another. Commonly used for load-balanced services and friendly names. |

---

## Deployment Best Practices

| Practice | Detail |
|---|---|
| Deploy at least two Domain Controllers per domain | A single DC is a single point of failure for authentication, Group Policy, and all directory-dependent services. Two DCs provide redundancy and eliminate a complete service outage if one DC is lost. |
| Enforce role separation | Domain Controllers should not run additional workload roles (Exchange, SQL Server, IIS) unless the environment is specifically Windows Small Business Server. Shared roles increase the attack surface, complicate patching, and introduce resource contention that affects authentication performance. |
| Use hardware covered by vendor support lifecycle | DC failures outside of vendor support leave you without firmware updates and without a support path. Always check that hardware will remain supported for the intended operational period before promotion. |
| Run sysprep on virtual machine templates | Virtual DCs built from templates that were not properly sysprepped will have duplicate machine SIDs. Run `sysprep.exe` before promoting any template-derived Windows Server. Run Memory Diagnostics from the installation media before first boot to detect hardware-level issues early. |
| Document DSRM passwords and store them securely | Directory Services Restore Mode (DSRM) is the only way to perform authoritative restores and certain AD database repairs. The DSRM password is set per-DC at promotion. If it is lost, that DC cannot be recovered without a rebuild. Store it in a secured password management system. |
| Use answer files for DC promotion | Scripted, reviewed, and signed-off promotions eliminate the risk of configuration drift between Domain Controllers. After use, include the answer file in documentation — the server strips passwords from it automatically after promotion. |
| Review dcpromo.log, dcpromoui.log, and Event Viewer post-promotion | Promotion can complete with warnings that indicate problems that will surface later. Review all three log sources immediately after promotion and resolve any issues before the DC enters production rotation. Note: `dcpromo.exe` is no longer an interactive command since Windows Server 2012 — promotion is performed via Server Manager or `Install-ADDSDomainController` in PowerShell, but the log files remain at the same paths. |
| Run Windows Update immediately after promotion | AD-specific updates are not offered to Windows Server until after the DC promotion role is applied. The post-promotion update run is therefore not optional — it may address critical AD service vulnerabilities. |
| Configure regular System State backups | System State backup is the correct mechanism for AD recovery. It captures the AD database, SYSVOL, registry, and boot files. Configure it to run periodically and test restores. Back up Group Policy Objects separately through the Group Policy Management Console — granular GPO recovery is not possible from a System State backup alone. |
| Run the AD Best Practices Analyser regularly | The built-in BPA identifies configuration issues against Microsoft's best practice baseline. Schedule regular BPA runs as part of the routine health-check process rather than waiting for a visible problem. |
