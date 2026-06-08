# Cisco Device Hardening: Disabling Call Home and Controlling Outbound Communications

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Defense & Intelligence Systems  
**Tags:** Cisco, Network Security, Device Hardening, Call Home, Outbound Traffic, IOS, Air-Gapped, DISA STIG, CMMC, Zero Trust

---

Cisco networking equipment ships pre-configured with telemetry and support features — including Call Home, Smart Licensing transport, and HTTP management interfaces — that routinely initiate outbound connections to Cisco's cloud infrastructure. In defence, intelligence, and regulated environments, these unsolicited outbound flows can violate network boundary policy, create audit findings, or expose sensitive operational data. This guide walks through a complete, structured approach to disabling and locking down all of them.

---

## Why Call Home Is Enabled by Default

Cisco introduced Call Home to simplify TAC support workflows. When enabled, the feature automatically opens a support case, attaches relevant device diagnostics, and notifies Cisco of hardware or software events — without operator intervention. For commercial enterprise environments, this accelerates resolution times considerably.

However, the default configuration creates several problems for high-security deployments:

- **Unsolicited outbound traffic** — Call Home initiates connections to `tools.cisco.com` and `api.cisco.com` without operator action, which violates deny-by-default egress policies.
- **Data exfiltration risk** — diagnostic bundles sent to Cisco may contain sensitive configuration data, interface names, routing table excerpts, or hostname/domain information that reveals internal network topology.
- **Audit findings** — DISA STIG, CMMC, and equivalent frameworks require explicit control over all outbound communications from network infrastructure devices.
- **Air-gap violations** — in classified or physically isolated environments, any outbound internet connectivity is prohibited. Call Home will continue attempting to reach Cisco's cloud, generating connection failures in logs and potentially triggering security alerts.

---

## Step-by-Step Configuration

### Step 01: Disable Call Home Globally

Start by turning off the Call Home feature at the global configuration level. This single command prevents the device from initiating any unsolicited outbound communication to Cisco's telemetry and support infrastructure.

```ios
no service call-home
```

### Step 02: Disable Smart Licensing Transport

Smart Licensing transport sends periodic licensing telemetry to Cisco's cloud. In air-gapped or restricted environments this communication must be stopped.

> **Note:** Not supported on all platforms. If the command returns an error, skip this step safely.

```ios
license smart transport off
```

### Step 03: Deactivate the Default CiscoTAC-1 Call-Home Profile

Even after disabling Call Home globally, the default CiscoTAC-1 profile remains in the running configuration. To eliminate any possibility of fallback behaviour — for example during a software reload or partial configuration restore — explicitly deactivate the profile and strip its HTTP destination.

```ios
call-home
 profile CiscoTAC-1
  no active
  no destination address http https://tools.cisco.com/its/service/oddce/services/DDCEService
```

> **IOS-XE note:** On IOS-XE (including CSR1000v), the `http` keyword is used for both HTTP and HTTPS destination addresses in call-home profiles. On some IOS-XE versions you may also need `no destination address https <url>` — verify with `show call-home profile all` after applying.

### Step 04: Disable HTTP Client and Secure Server

The IOS HTTP client allows the device to initiate outbound web requests; the secure server exposes a management web interface. Disabling both reduces the device's attack surface and ensures it cannot initiate or host web-based communications.

```ios
no ip http client
no ip http secure-server
```

> **IOS-XE (CSR1000v) note:** On IOS-XE platforms, also disable the HTTPS client explicitly:
> ```ios
> no ip http secure-client
> ```

### Step 05: Disable DNS Lookups

When DNS lookup is enabled and an operator miskeys a command, IOS attempts to resolve the mistyped string as a hostname — causing a 30+ second CLI hang while it waits for a DNS response. In air-gapped environments, that response never arrives. Disabling DNS lookups eliminates the delay and prevents any unintended DNS queries from leaving the device.

```ios
no ip domain-lookup
```

### Step 06: Remove NTP Servers and Peers

If NTP is not used in your time-synchronisation architecture, remove any configured time servers and peers. Leaving these configured directs periodic NTP traffic to external hosts — potentially outside the network boundary.

```ios
no ntp server <address>
no ntp peer <address>
```

### Step 07: Disable SNMP

SNMP opens a management channel that can be queried or exploited if not properly secured. If SNMP monitoring is not in use, disable it entirely to close the vector. If you do need SNMP, lock it down to SNMPv3 with authentication and privacy settings before production deployment.

```ios
no snmp-server
```

### Step 08: Block Outbound HTTPS with an ACL

For the most stringent outbound control posture — particularly in defence or intelligence system deployments — implement an ACL that explicitly denies HTTPS traffic exiting the WAN interface. This acts as a hard enforcement layer independent of the software-level disables above.

> **Note:** Replace `GigabitEthernet1.45` with your actual external-facing interface. Verify the ACL does not conflict with existing outbound policies before applying.

```ios
ip access-list extended BLOCK-OUTBOUND-HTTPS
 deny tcp any any eq 443
 permit ip any any
!
interface GigabitEthernet1.45
 ip access-group BLOCK-OUTBOUND-HTTPS out
```

---

## Verifying the Configuration

After applying the steps above, verify the resulting configuration with the following show commands before saving to NVRAM:

```ios
show call-home
show call-home profile all
show running-config | include http
show running-config | include snmp
show running-config | include domain-lookup
show ip access-lists BLOCK-OUTBOUND-HTTPS
```

Confirm that `call-home` shows as inactive, the CiscoTAC-1 profile shows no active flag, and no HTTP client or SNMP server lines appear in the running config. Once verified, save with `write memory`.

---

## Compliance and Framework Context

| Framework | Relevance |
|---|---|
| DISA STIG (Network Devices) | Requires disabling all unnecessary management protocols and outbound telemetry. Call Home and HTTP management interfaces are explicit findings in Cisco IOS STIG benchmarks. |
| CMMC Level 2/3 | Access control and configuration management domains require that all outbound communications from infrastructure are authorised and logged. Unsolicited Call Home traffic violates this requirement. |
| NIST 800-53 CM-7 | Least Functionality — prohibits ports, protocols, and services that are not required for the device's operational role. |
| Zero Trust Architecture | All flows must be explicitly authorised. An appliance initiating unsolicited outbound HTTPS connections to a vendor cloud is incompatible with a deny-by-default egress posture. |
