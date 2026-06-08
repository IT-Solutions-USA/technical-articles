# VMware vSphere 7 Security Configuration Guide: A Practitioner's Overview

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / vSphere  
**Tags:** VMware, vSphere 7, ESXi, Security Hardening, SCG, PowerCLI, NIST, CMMC, PCI-DSS, Compliance

---

The VMware vSphere Security Configuration Guide (SCG) is the official baseline for hardening and auditing vSphere infrastructure. Version 7 — which introduced the most significant structural changes since vSphere 6.5 — reflects a fundamentally different threat landscape: ransomware targeting ESXi hosts, active exploitation of SLP and CIM, and CPU-level hardware vulnerabilities that simply did not exist when vSphere 6.7 shipped. This article walks through the structure of the guide, its methodology, and where to focus attention first.

---

## What Changed in vSphere 7 SCG

The v7 SCG introduced three structural changes that distinguish it from prior versions and reflect the changed threat environment of 2020–2021.

**1. Compliance alignment.**  
For the first time, the SCG is aligned to NIST 800-53 controls, with cross-mapping to NIST 800-171, CMMC, PCI-DSS, ISO 27001, and NERC CIP. This reduces the translation gap between what the SCG recommends and what a compliance auditor needs to verify — a gap that has historically produced significant friction during assessments.

**2. Commitment to the CIA triad.**  
Security in earlier SCG versions focused heavily on confidentiality. Version 7 explicitly extends the framework to cover integrity and availability, acknowledging that ransomware and CPU hardware vulnerabilities (Spectre, Meltdown, and successors) are existential threats to the availability of virtualised infrastructure.

**3. Secure by default, less to manage.**  
Where vSphere 7 changed its default configuration to match the SCG recommendation, the guidance was changed to "Audit or Remove" — meaning administrators can rely on the platform default rather than maintaining a custom parameter. This reduces the surface area of hardening configuration that needs to be tracked and validated over time.

---

## Notable New Controls in v7

| Control ID | Description |
|---|---|
| esxi-7.disable-slp | Introduced guidance to disable the Service Location Protocol (SLP) daemon. SLP was subsequently actively exploited in ransomware campaigns targeting VMware infrastructure (CVE-2021-21974 / ESXiArgs, 2021–2023), leading VMware to change the default to disabled in ESXi 7.0 U3c and later patches. |
| esxi-7.hardware-tpm | New guidance to acquire and enable hardware TPM 2.0 modules. TPM enables vSphere Trust Authority, secure boot attestation, and encrypted VM state — important and increasingly inexpensive on modern server hardware. |
| esxi-7.network-isolation-vmotion / vsan / hardware-management | Network isolation controls for vMotion, vSAN, and hardware management traffic — reinforcing defence-in-depth through dedicated, non-routable network segments for sensitive traffic flows. |
| esxi-7.supported / vcenter-7.supported | Addresses a common compliance loophole: an environment can be technically up to date on patches while running end-of-life software. This control requires that the vSphere version itself remain within the supported lifecycle. |
| vcenter-7.vami-access-dcli | Guidance on restricting access to the vCenter appliance management interface and the Direct Console UI (DCUI) — management plane controls that are frequently overlooked in vCenter hardening. |
| SSH removal from esxcli examples | All esxcli and VCLI examples from prior SCG versions were removed. The SCG now recommends PowerCLI and the vSphere Client exclusively — SSH is a management interface that should remain disabled except when diagnostically required. |

---

## Anatomy of the SCG: Five Control Categories

The SCG is distributed as a PDF and an accompanying Microsoft Excel spreadsheet. The spreadsheet is the working document — it contains five tabs, each representing a distinct control category.

| Tab | Description |
|---|---|
| VMware ESXi | Controls that apply directly at the hypervisor level — the vSphere Standard Switch, Host Client, lockdown mode, SSH state, and ESXi-specific advanced settings. This is the most operationally sensitive tab. |
| VMware vCenter Server | Controls specific to the vCenter Server appliance management interface (VAMI) and features enabled through vCenter, including the vSphere Distributed Switch, permissions model, and SSO configuration. |
| Virtual Machines | Controls that apply to the outside of a virtual machine — advanced VM settings, hardware version, vTPM configuration, and resource isolation. Does not require guest OS coordination. |
| In-Guest | Controls that must be coordinated with the guest operating system, such as VMware Tools settings, CPU vulnerability mitigations, and specific advanced parameters that affect guest behaviour. |
| Deprecated | Controls from prior SCG versions that are no longer applicable, have been superseded, or were removed because the guidance caused more harm than good. Review this tab when migrating from an older SCG version. |

---

## Action Types

| Action | Meaning |
|---|---|
| Add | The parameter or setting does not exist in the default configuration. You will need to add it explicitly to implement this control. |
| Audit | Check the current value and correct it if it deviates. This covers settings that exist but may have been changed by an administrator or a prior automation run. |
| Audit or Remove | vSphere 7 defaults already reflect this guidance. After auditing for deviations you can remove the parameter entirely and rely on the platform default — reducing the number of custom settings you have to manage. |
| Modify | The parameter exists and is present by default but must be changed from its current value. The SCG tells you what value it should be set to. |

---

## Where to Focus First

| Priority | Detail |
|---|---|
| Patch everything, at all levels | ESXi, vCenter, VM hardware versions, VMware Tools, and the underlying host firmware. Ransomware and crypto-mining campaigns have shown that unpatched ESXi is a high-value target. The SCG's guidance on esxi-7.supported and vcenter-7.supported formalises this as a compliance control, not just a best practice. |
| Disable SSH and leave it disabled | Enable SSH only when you actively need it for a specific diagnostic task, then disable it again immediately. The SCG made this a core tenet starting with v7 — removing all esxcli/VCLI examples to reinforce the message. Automated management via PowerCLI and the vSphere API is the right model. |
| Harden authentication | Review Active Directory integration for vCenter SSO, enforce strong password policies, restrict permissions to named accounts, and ensure lockdown mode is configured on ESXi hosts. These controls prevent lateral movement in the event any adjacent system is compromised. |
| Enforce network isolation | vMotion, vSAN, hardware management, and iSCSI traffic should each run on dedicated, non-routable network segments. The v7 SCG introduced explicit controls for this — esxi-7.network-isolation-* — because flat management networks remain one of the most common findings in vSphere environment assessments. |

---

## How to Use the SCG Without Turning It Into a Checklist

The most common misuse of the vSphere SCG — and of security guides generally — is treating it as a pass/fail checklist where every row must be implemented as written. The SCG explicitly rejects this model. Security is a process that involves tradeoffs between usability, staff time, budget, and risk appetite.

The right approach is to start with a discussion — between vSphere administrators, security leadership, and the teams responsible for workloads. The goal is to identify where your environment deviates from the SCG's recommendations, understand why, and make a deliberate, documented decision about each deviation. That documentation is the compliance artefact, not the spreadsheet.

Not every SCG control applies to every environment. A control around vSAN network isolation is irrelevant in an environment that does not run vSAN. Applying it anyway adds complexity without reducing risk.

---

## Testing Before Production: Use Nested Virtualisation

The SCG recommends an approach that VMware's own Hands-on Labs teams use extensively: nested virtualisation for testing. ESXi can be installed inside an ESXi host for non-production purposes. You can configure virtual TPMs, enable Secure Boot, set up vSAN, and exercise most SCG controls in a nested environment before touching production.

Critically, you can snapshot the entire nested environment — take all VMs offline first for cluster consistency — so that any destructive configuration change can be reverted without impacting production. This eliminates most of the risk associated with staged SCG implementation.

---

## Automation: PowerCLI Over SSH

All SCG controls are implemented through one of three paths: PowerCLI, the vSphere Client, or the Host Client. The SCG removed all esxcli and VCLI examples in v7 specifically to discourage SSH as a management interface. There is a single exception — verifying ESXi Secure Boot enablement may require SSH — but the guidance is to disable SSH again immediately after.

The PowerCLI examples in the SCG spreadsheet are prefixed with `#` to prevent accidental paste-and-execute. This is intentional. These scripts can make changes with significant operational consequences — test in a nested or non-production environment first, and own the impact of any change you run.

For teams building automation around SCG controls, VMware's Code Capture and API Explorer features inside the vSphere Client Developer Center provide a useful starting point — if you can perform an action in the UI, Code Capture will generate a PowerCLI or API equivalent automatically.
