# VMware ESXi Hardening: A 45-Control Security Configuration Assessment

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / Security  
**Tags:** ESXi, VMware, Hardening, CIS Benchmark, PowerCLI, vSphere, Security Audit, Compliance, Infrastructure

---

ESXi hypervisors are the foundation of every virtual infrastructure — and because every workload runs on top of them, a misconfigured host is a single point of failure for your entire security posture. This assessment reviews 45 controls drawn from VMware's Security Configuration Guide, organized into actionable categories with PowerCLI audit and remediation commands for each.

---

## Scope and Assessment Approach

The controls in this guide span authentication, system services, management-plane security, network segmentation, logging, hardware trust mechanisms, and software integrity. Each control is graded by the action required:

- **Modify** — non-compliant by default; change the value.
- **Audit** — verify current state; correct if it deviates.
- **Add** — setting absent by default; add it explicitly.

PowerCLI commands are provided for both audit and remediation throughout. The latest VMware Security Configuration Guide introduces a priority-based model for remediation: **High-priority** controls should be remediated immediately via Host Profiles; **Medium** controls should be reviewed and planned; **Strategic** controls require hardware coordination or phased rollout.

---

## 1. Authentication Controls

| ID | Parameter | Desired | Default | Action | Risk |
|---|---|---|---|---|---|
| AccountLockFailures | Security.AccountLockFailures | 3 | 10 | Modify | Default of 10 gives attackers far too many brute-force attempts before lockout. |
| AccountUnlockTime | Security.AccountUnlockTime | 900 s | 120 s | Modify | 2-minute auto-unlock is too short; raises brute-force risk while still blocking legitimate users. |
| PasswordHistory | Security.PasswordHistory | 5 | 0 | Modify | No history tracking lets users immediately cycle back to compromised passwords. |
| PasswordQualityControl | Security.PasswordQualityControl | Site-specific | Varies | Modify | Weak complexity rules allow easily guessable credentials. |

---

## 2. Service Controls

| Service | Key | Default | Action | Risk |
|---|---|---|---|---|
| CIM (sfcbd-watchdog) | sfcbd-watchdog | Enabled | Disable | Exposes hardware/management info; enlarges attack surface. |
| SLP (slpd) | slpd | Disabled (patched ESXi 7.0 U3c+); Enabled (older) | Disable/Audit | Unused discovery service; actively exploited in ESXiArgs ransomware campaign (CVE-2021-21974). VMware changed the default to disabled in ESXi 7.0 U3c and later patches — verify your patch level. |
| SNMP (snmpd) | snmpd | Stopped (not running by default in ESXi 7.x unless configured) | Disable/Audit | Leaks host inventory data if community strings are weak or default. |
| SSH (TSM-SSH) | TSM-SSH | Stopped | Audit/Keep Off | Interactive shell; only enable for maintenance windows, then disable immediately. |
| ESXi Shell (TSM) | TSM | Stopped | Audit/Keep Off | Low-level shell access; keep disabled in production at all times. |
| MOB (enableMob) | Config.HostAgent.plugins.solo.enableMob | False | Audit | If enabled, attackers can manipulate internal ESXi APIs. |

---

## 3. Lockdown Mode & Access Control

Lockdown Mode forces all ESXi management traffic through vCenter, preventing direct host login and circumvention of RBAC. It should be enabled as **Normal** (not Strict, which disables DCUI entirely) on all production hosts.

Key lockdown mode considerations:
- Normal lockdown allows DCUI access for emergency recovery; Strict removes it entirely.
- Exception users list should contain only break-glass accounts.
- Audit regularly — lockdown mode can be quietly disabled by an administrator with direct host access.

---

## 4. Network Controls

| Control | Parameter | Desired | Default | Risk |
|---|---|---|---|---|
| Block Guest BPDU | Net.BlockGuestBPDU | 1 | 0 | VM-generated BPDUs can corrupt spanning-tree topology and drop uplinks. |
| Reject Forged Transmits | vSwitch SecurityPolicy | Reject | Accept | Allows VMs to spoof MAC addresses and impersonate other devices. |
| Reject MAC Changes | vSwitch SecurityPolicy | Reject | Accept | Enables VMs to impersonate other network nodes by changing their MAC. |
| Disable Promiscuous Mode | vSwitch SecurityPolicy | Reject | Reject | Promiscuous mode lets VMs capture all traffic on a port group. |
| dvFilter Bind IP | Net.DVFilterBindIpAddress | Null | Null/Site | If bound to an untrusted IP, dvfilter can leak or expose traffic. |
| Firewall — Mgmt Subnets Only | VMHostFirewallException | Restricted | All IPs | Open firewall allows any host to attempt management-plane access. |

---

## 5. Logging Controls

| Control | Parameter | Desired | Default | Action | Notes |
|---|---|---|---|---|---|
| Log Level | Config.HostAgent.log.level | info | info | Audit | Keep at Info. Verbose fills disks; too low misses critical events. |
| Persistent Log Dir | Syslog.global.logDir | /vmfs/volumes/… | [] /scratch (RAM) | Modify | In-memory logs are lost on reboot, destroying the forensic audit trail. |
| Remote Syslog | Syslog.global.logHost | tcp://syslog-host:514 | None | Add | Central log store prevents tampering and enables SIEM correlation. |

---

## 6. Shell Timeout Controls

| Control | Parameter | Desired | Default | Action |
|---|---|---|---|---|
| Interactive Timeout | UserVars.ESXiShellInteractiveTimeOut | 600 s | 0 (none) | Modify |
| Session Timeout | UserVars.ESXiShellTimeOut | 600 s | 0 (none) | Modify |
| DCUI Timeout | UserVars.DcuiTimeOut | 600 s | 600 s | Audit |
| Suppress Shell Warning | UserVars.SuppressShellWarning | 0 | 0 | Audit |
| Suppress HT Warning | UserVars.SuppressHyperthreadWarning | 0 | 0 | Audit |

---

## 7. Hardware Trust & Software Integrity

This category covers controls that require coordination with your hardware and infrastructure teams, or that carry risk during enablement and need phased rollout.

**High-priority items:**
- Enable hardware TPM 2.0 — prerequisite for vSphere Trust Authority and Secure Boot attestation.
- Enable Secure Boot on ESXi hosts — prevents unsigned VIBs from loading.
- Enable virtual TPM (vTPM) for VMs — required for Windows 11 and enhanced guest OS integrity.

**Medium-priority items:**
- Enable vSphere Trust Authority (vTA) — attested boot chain from hardware to hypervisor.
- Review VIB acceptance levels — set to `PartnerSupported` or `VMwareCertified` minimum.

**Strategic items:**
- Hardware lifecycle planning — ensure servers remain under vendor support for firmware updates.
- Side-channel mitigation review — validate CPU microcode patches for Spectre/Meltdown variants are applied.

---

## 8. Protocol Hygiene & Miscellaneous Controls

Additional controls covering protocol-level risk reduction:

- **Disable DCUI access for non-root users** — restrict DCUI to the root account only.
- **Set NTP servers** — time synchronisation is required for Kerberos authentication and log correlation.
- **Disable SFCBD** — the CIM broker service is rarely needed and exposes unnecessary API surface.
- **Configure host-based firewall rules** — restrict management plane access to defined management subnets.
- **Review VIB third-party acceptance** — third-party VIBs at `CommunitySupported` level bypass VMware signing validation.

---

## Recommended Next Steps

This assessment is most effective when operationalised, not just read. A practical five-step approach:

1. **Baseline audit** — run PowerCLI against all hosts to document current deviation from SCG defaults.
2. **Prioritise by risk** — address High-priority items first via Host Profiles for consistency.
3. **Test in nested environment** — validate changes in a nested ESXi lab before production rollout.
4. **Automate remediation** — encode compliant settings into Host Profiles; enforce via vCenter.
5. **Schedule re-assessment** — re-audit after every major vSphere upgrade and annually at minimum.
