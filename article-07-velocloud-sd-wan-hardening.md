# VMware SD-WAN VeloCloud: Security Hardening Guidelines

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Network / SD-WAN  
**Tags:** SD-WAN, VeloCloud, VMware, Network Security, Hardening, BGP, OSPF, Zero Trust

---

Compromised network devices can be used to redirect traffic, exfiltrate data, and pivot into internal systems — making SD-WAN Edge hardening a critical part of any enterprise security posture. This guide covers every control plane across the VMware SD-WAN VeloCloud architecture: Orchestrator access management, Edge lockdown, BGP and OSPF route filtering, data plane segmentation, and physical security.

---

## The Three Planes — What Gets Hardened and Why

Network device hardening is structured around three functional planes. In the VMware SD-WAN architecture, each plane is consumed by a different component, and each requires a different set of controls.

| Plane | SD-WAN Component | Function |
|---|---|---|
| Management Plane | VMware SD-WAN Orchestrator | Access, configure, manage and monitor network devices. The Orchestrator is the primary portal for all administrative operations. |
| Control Plane | Orchestrator + Gateways (proxy) | Protocols and processes that communicate routing information between devices — BGP, OSPF, and SD-WAN path selection. |
| Data Plane | VMware SD-WAN Gateways + Edges | The actual transport of packets from source to destination. Gateways are the primary distributed data plane; Edges consume all three planes. |

The VMware SD-WAN Edges consume all three planes from the Orchestrator and Gateways. This makes the Orchestrator the single highest-value target — a compromised Orchestrator account grants management-plane access to every Edge in the fabric.

---

## Orchestrator Access Management

| Priority | Control | Detail |
|---|---|---|
| Critical | Limit SuperUser accounts to 3 | SuperUsers can create Standard Admin accounts and have unrestricted access. Restricting the count reduces the blast radius of a compromised credential. SuperUsers should create role-limited accounts for day-to-day operations. |
| Critical | Apply the principle of least privilege | Assign users only the privilege level required for their specific tasks. When the correct level is unclear, default to 'Enterprise Read Only'. Escalate privileges temporarily rather than permanently. |
| Critical | Enable two-factor authentication (2FA) | Enable 2FA in System Settings. Supported methods include SMS to a registered mobile phone number and TOTP authenticator apps (e.g. Google Authenticator, Microsoft Authenticator). TOTP is preferred over SMS as it is not vulnerable to SIM-swapping attacks. This prevents credential-only attacks from gaining portal access. |
| High | Enforce a strong password policy | Minimum 12 characters. Mandatory mix of lowercase, uppercase, numbers, and special characters. Longer passwords are significantly more secure — consider a 20-character minimum for privileged accounts. |
| High | Disable accounts that are no longer in use | Establish a regular cadence to review all active accounts and their privilege levels. Disable accounts immediately when a user changes roles or leaves the organisation. Do not delete — disable, to preserve audit history. |
| High | Enforce individual accounts — no shared credentials | Shared accounts eliminate traceability. Every action in the Orchestrator portal must be attributable to a specific individual for audit and forensic purposes. One account per person, no exceptions. |
| Medium | Force password reset on suspected compromise | If an account shows unusual activity or credentials may have been exposed, initiate an immediate forced password reset from the individual account settings. Do not wait for the scheduled rotation cycle. |
| Medium | Disable support access and user management delegations to the operator | Support access delegation can expose user-identifiable information (source IP, hostname, MAC address) to the operator during diagnostic sessions. User management delegation allows operators to create users and reset passwords in the enterprise account. Both should be disabled in System Settings unless actively needed. |

---

## Edge Access Controls

| Control | Risk | Detail |
|---|---|---|
| Disable SSH (Support Access) — set to Deny All | High | SSH access to the Edge's diagnostic CLI should be off by default. If temporary access is needed for a specific troubleshooting task, restrict it to a single source host and revert to Deny All immediately after. |
| Disable Local Web UI — set to Deny All | High | The local web interface exposes port configurations, system status, and diagnostics. It is always available pre-activation and must be locked down post-activation. If access is required, restrict to one host and set a non-standard port (not 80, 443, or 8080). |
| Restrict or disable SNMP | High | If SNMP is not in use, set to Deny All. If it is used: restrict access to collector IP addresses only, enable SNMPv3 exclusively (v1/v2c are unencrypted), set long non-predictable community strings and v3 usernames/passwords, and enable Privacy with AES encryption for all data exchange. |
| Review and purge NAT/port forwarding rules | Medium | Establish a scheduled review of all port forwarding and NAT rules. Remove any entries that are no longer actively used — each open port is an additional attack surface exposed to public vectors. |

---

## Control Plane Security

| Control | Detail |
|---|---|
| Remove all inactive routing configurations | Stale routing entries can be exploited to form rogue protocol adjacencies. Any BGP peer, OSPF neighbour, or static route that is no longer in active use should be removed immediately. |
| Tag all BGP received routes with community attributes | Community tagging of received routes promotes transparency of route origination, making it significantly easier to identify anomalous or unexpected routing entries in the RIB. |
| Filter unknown routes for both BGP and OSPF | Configure explicit prefix filters to drop routes that cannot be attributed to known networks. A default deny on inbound routing updates prevents route injection attacks. |
| Filter received default routes (BGP and OSPF) | Do not accept a default route from a BGP peer or OSPF neighbour unless it is an intentional, documented design decision. An injected default route can redirect all traffic through an attacker-controlled path. |
| Require certificate authentication for Edge provisioning | Set Authentication Mode to 'Certificate Required' in the Provision New Edge dialog. This allows the Orchestrator to cryptographically verify the identity of each Edge device and enables revocation of a compromised Edge from the SD-WAN fabric. |

---

## Data Plane Controls

| Control | Detail |
|---|---|
| Enable Segmentation in every profile | Enable segmentation even if a single segment is currently in use. Converting a flat profile to a segmented one later requires significant re-planning. With segmentation already enabled, isolating a suspected compromised segment during an incident is a single configuration change. |
| Isolate guest traffic on a dedicated segment | Associate all guest or untrusted wireless traffic to a separate segment. Disable Cloud VPN for the guest segment. Optionally configure overlapping address space in the guest segment to further enforce isolation and conserve address space. |
| Disable all unused physical ports in the profile | Default to closed on all ports at the profile level. Re-enable specific ports in the Edge configuration only as needed. This prevents rogue devices from connecting to an open port and gaining access to network resources. |
| Disable wireless radio if not required | If the Edge's wireless capability is not in use, disable the radio in the profile device settings. An active but unconfigured wireless radio is an open access point waiting to be exploited. |
| Enforce WPA-Enterprise + RADIUS for wireless | If wireless is offered from the Edge, configure WPA-ENT authentication with RADIUS for AAA services. The RADIUS server should be reachable only over the corporate VPN — never over open Internet links, as RADIUS is not encrypted in transit without additional protection (e.g. RADSEC). |
| Disable all inactive non-VMware SD-WAN site tunnels | Remove or disable any IPSec tunnels to non-SD-WAN sites that are no longer in active use. At minimum, set the tunnel to disabled status to prevent it from attempting renegotiation. |
| Use a strong PSK for non-SD-WAN site tunnels | When configuring non-SD-WAN site tunnels, use the Orchestrator-generated PSK or replace it with a manually set key of at least 32 characters, mixing lowercase letters, uppercase letters, numbers, and special characters. |
| Allow UDP/2426 inbound at hub-side firewalls | If an external firewall sits at the WAN side of a hub Edge, ensure it allows inbound UDP/2426. This is the primary data plane port used by VCMP (VeloCloud Management Protocol) for all branch-to-hub overlay tunnel traffic — not just initial establishment. Allow all outbound flows without restriction. |
