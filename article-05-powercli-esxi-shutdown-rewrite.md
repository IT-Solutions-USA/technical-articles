# PowerCLI ESXi Shutdown: Code Review and Production Rewrite

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / PowerCLI  
**Tags:** PowerCLI, ESXi, VMware, PowerShell, Code Review, Automation, vSphere, Security

---

The original PowerCLI shutdown script circulating across VMware communities is widely copy-pasted into production runbooks. It works in a lab — but it has seven real problems that range from a plaintext password in the command line to a logic flaw that can trigger a host shutdown while VMs are still running. This article reviews every issue and presents a production-grade rewrite.

---

## Issues Found in the Original Script

| # | Severity | Issue | Problem | Fix |
|---|---|---|---|---|
| 1 | High | Plaintext password in the command line | The password is visible in plain text, will appear in shell history, process lists, and any logging. Anyone with access to the host running the script can read it. | Use `Get-Credential` to prompt for credentials at runtime. It returns a `PSCredential` object with the password stored as a `SecureString` — never written to disk or visible in logs. |
| 2 | High | No error handling — silent failures cascade | If `Connect-VIServer` fails, `Stop-VM` fails, or any other step throws, the script continues to the next line. The host shutdown command can be reached even when VMs are still running, risking data corruption. | Wrap everything in `try/catch/finally`. Set `$ErrorActionPreference = 'Stop'` so any terminating error is caught immediately instead of silently skipped. |
| 3 | High | No verification that VMs are actually off before host shutdown | The 10-second sleep is a blind wait — it doesn't check whether VMs actually powered off. If `Stop-VM` takes longer than 10 seconds (common), `Stop-VMHost` runs against a host that still has live VMs, which can fail or cause inconsistent state. | After the shutdown attempt, re-query `Get-VM` and check `PowerState`. Only proceed to `Stop-VMHost` when count of powered-on VMs is confirmed zero. If VMs remain, abort and throw. |
| 4 | Medium | `Disconnect-VIServer` without `-Server *` | Omitting `-Server` disconnects only the most recently connected server when multiple connections are open. In environments with multiple connections, other sessions are left dangling. | Use `Disconnect-VIServer -Server * -Confirm:$false` to close every open connection. Place this in the `finally` block so it always runs, even when an error occurs. |
| 5 | Low | No logging or progress output | When the script runs unattended (scheduled task, runbook), there is no record of what happened, when, or whether it succeeded. Failures leave no trail. | Add a `Write-Log` helper that prefixes every message with a timestamp and severity level. This makes the output useful both interactively and when redirected to a log file. |

---

## Improvements in the Rewrite

| Improvement | Description |
|---|---|
| `Get-Credential` instead of plaintext `-Password` | Prompts interactively and stores the password as a `SecureString`. For unattended use, retrieve credentials from PowerShell SecretManagement or a secrets vault — never embed them in the script. |
| `try / catch / finally` with `$ErrorActionPreference = 'Stop'` | Every error is caught immediately. The `finally` block guarantees `Disconnect-VIServer` runs even when the script fails mid-way. Without this, a failed `Stop-VM` could silently allow the host shutdown to proceed. |
| `Shutdown-VMGuest` → poll → `Stop-VM` fallback | Graceful guest OS shutdown first, with a configurable timeout and real polling loop. Only falls back to a hard power-off for VMs that do not respond within the grace period. |
| Hard abort before `Stop-VMHost` | If any VM is still powered on after all shutdown attempts, the script throws and exits. `Stop-VMHost` is never called with live VMs on the host. |
| VMware Tools status check per VM | Queries `ExtensionData.Guest.ToolsRunningStatus` before calling `Shutdown-VMGuest`. VMs without running Tools are flagged immediately as WARN and queued for force-off rather than silently failing. |
| `Write-Log` with timestamps and severity | Every action is logged with a UTC timestamp and color-coded severity (INFO=cyan, WARN=yellow, ERROR=red). Redirect stdout to a file for persistent audit records. |
| `Disconnect-VIServer -Server *` | Closes every open vSphere connection, not just the most recent. Placed in `finally` so it executes regardless of whether the script succeeded or failed. Always use `-Server *` when multiple connections may be open. |
| `param()` block with `-HardPowerOff` switch | All tunable values (timeout, poll interval, hard-off override) are script parameters, not magic numbers buried in the code. Supports `-HardPowerOff` for lab environments where guest shutdown is unnecessary. |

---

## The Original Script

Here is the script under review, exactly as typically found:

```powershell
# Connect to vCenter Server or ESXi host
Connect-VIServer -Server YourESXiHost -User YourUsername -Password YourPassword

# Get all powered-on virtual machines
$vms = Get-VM | Where-Object { $_.PowerState -eq 'PoweredOn' }

# Power off all virtual machines
foreach ($vm in $vms) {
    Stop-VM -VM $vm -Confirm:$false
}

# Disconnect all connected users
Get-VMHost | Foreach-Object { Get-VMHost $_.Name | Foreach-Object { $_ | Disconnect-VMHost } }

# Wait for a few seconds to ensure all users are disconnected
Start-Sleep -Seconds 10

# Shut down the ESXi host
Stop-VMHost -VMHost (Get-VMHost) -Confirm:$false

# Disconnect from the vCenter Server or ESXi host
Disconnect-VIServer -Confirm:$false
```

---

## How to Run the Rewritten Script

Save the script as `Invoke-ESXiShutdown.ps1` and call it with parameters:

```powershell
# Standard run — prompts for credentials, waits 120s for graceful shutdown
.\Invoke-ESXiShutdown.ps1 -Server <esxi-host>

# Shorter timeout
.\Invoke-ESXiShutdown.ps1 -Server <esxi-host> -GuestShutdownTimeout 30

# Skip graceful shutdown entirely — hard power-off all VMs immediately
.\Invoke-ESXiShutdown.ps1 -Server <esxi-host> -HardPowerOff

# Log output to file for audit trail
.\Invoke-ESXiShutdown.ps1 -Server <esxi-host> *>&1 | Tee-Object -FilePath shutdown.log
```

For unattended execution (scheduled task, runbook), replace `Get-Credential` with a credential retrieved from PowerShell SecretManagement:

```powershell
# Requires Microsoft.PowerShell.SecretManagement + a registered vault
$cred = Get-Secret -Name "vsphere-svc-account"
# Get-Secret returns a SecureString by default; -AsPlainText is not needed here
```
