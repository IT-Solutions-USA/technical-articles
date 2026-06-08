# ESXi Recovery Playbook: hostd, localcli, vim-cmd, and Log Triage

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / ESXi Shell  
**Tags:** ESXi, hostd, localcli, vim-cmd, VMware, Troubleshooting, Recovery, PowerCLI

---

When the vSphere client cannot reach an ESXi host and PowerCLI connections time out, the management agent (hostd) has likely crashed or failed to start. This playbook covers the full recovery sequence: check and restart hostd, read the right logs to find the root cause, fall back to `localcli` when `esxcli` is unavailable, manage VMs directly from the ESXi Shell with `vim-cmd`, and audit snapshots with PowerCLI once connectivity is restored.

---

## Step 1 — Check and Restart the hostd Service

The hostd management agent handles all vSphere API calls — the vSphere client, PowerCLI, and the REST API all talk to hostd. If it is down, no remote management works. Connect via SSH or the DCUI console and run these commands in order:

| Command | Description |
|---|---|
| `/etc/init.d/hostd status` | Check whether the hostd management agent is currently running. Returns a PID if running, or 'hostd is not running' if crashed or never started. |
| `/etc/init.d/hostd start` | Start the hostd service. If this command completes without error, wait 30–60 seconds then verify the vSphere client can reconnect before proceeding. |
| `/etc/init.d/hostd restart` | Restart a hostd service that is running but unresponsive. Terminates the existing process and starts a fresh one — less disruptive than a full host reboot. |

**Decision tree:**
1. `/etc/init.d/hostd status` — if running but unresponsive, jump to restart.
2. If not running: attempt start and wait 60 seconds for the vSphere client to reconnect.
3. If start fails or hostd crashes again immediately: check the logs below before attempting another restart.

---

## Step 2 — Read the Right Logs

When hostd will not stay up, the logs tell you why. Three log files cover the full picture:

| Log File | Purpose | What to Look For |
|---|---|---|
| `/var/log/vmkernel.log` | VMkernel kernel messages | SCSI/storage errors, NFS timeouts, memory pressure, CPU scheduling failures. Kernel panics will appear here first. |
| `/var/log/hostd.log` | Management agent (hostd) log | hostd startup failures, API errors, certificate issues, out-of-memory conditions preventing hostd from initialising. This is the primary log for diagnosing hostd crashes. |
| `/var/log/vpxa.log` | vCenter agent (vpxa) log | vpxa-to-vCenter communication failures, certificate mismatches, VIB installation errors. If the host is disconnected from vCenter but hostd is running, check here. |

Tail the logs in real time to watch hostd start up and catch the exact line where it fails:

```sh
# Watch hostd log live
tail -f /var/log/hostd.log

# Search for errors in the last 200 lines
tail -200 /var/log/hostd.log | grep -i error

# Check vmkernel for storage or memory pressure
tail -100 /var/log/vmkernel.log | grep -iE "error|warn|scsi|nfs|oom"

# Check vpxa for vCenter connectivity issues
tail -100 /var/log/vpxa.log | grep -i error
```

---

## Step 3 — Use localcli When esxcli Is Unavailable

`esxcli` routes through the hostd API — so when hostd is down, `esxcli` commands will fail or hang. `localcli` is a direct CLI that bypasses hostd entirely and accesses the underlying ESXi subsystems directly. Every `esxcli` command has a `localcli` equivalent with the same syntax.

| Command | Description |
|---|---|
| `localcli network firewall get` | Show the current firewall state (enabled/disabled) and the default action for unmatched traffic. Use this to confirm whether the firewall is blocking management traffic. |
| `localcli network firewall set --enable false` | Disable the ESXi host firewall entirely. Use only for emergency troubleshooting to rule out firewall blocking the management agent — re-enable immediately once diagnosed. |
| `localcli system maintenanceMode set --enable true` | Place the host in maintenance mode via localcli. vMotion is not available when hostd is down, so VMs must be shut down manually before this can complete cleanly. |
| `localcli vm process list` | List all running VM processes and their World IDs. This is how you identify which VMs are still running on the host when the vSphere client cannot connect. |
| `localcli vm process kill -w <World ID> -t soft` | Send a graceful shutdown signal to a specific VM process identified by its World ID from the list above. Use `-t soft` first; escalate to `-t hard` or `-t force` if the VM does not respond. |

**Kill type reference:**

| Flag (-t) | Action | When to Use |
|---|---|---|
| `soft` | Sends a soft power-off signal to the VMX process. If VMware Tools is running and responsive, the guest OS may perform a graceful shutdown — but this is not guaranteed. Functionally similar to pressing the soft-power button. | Try first |
| `hard` | Hard power-off equivalent — like Stop-VM in PowerCLI | If soft fails |
| `force` | Forcibly kills the VMX process — use only as last resort | Last resort |

```sh
# Example: list VMs, find World ID, send soft kill
localcli vm process list
# Output includes: DisplayName, World ID, VMX Cartel ID, UUID

localcli vm process kill -w 12345 -t soft
# If no response after 60s, escalate:
localcli vm process kill -w 12345 -t hard
```

---

## Step 4 — Manage VMs from the ESXi Shell with vim-cmd

Once the ESXi Shell is accessible (via SSH or DCUI), `vim-cmd` provides VM lifecycle management that works for most operations independently of hostd. Power management operations (power.on, power.off, power.shutdown) do not require hostd to be running. These commands are available even when the vSphere client cannot connect.

| Command | Description |
|---|---|
| `vim-cmd vmsvc/getallvms` | List all registered VMs with their VMID, display name, datastore path, guest OS, and version. VMID is what every other vim-cmd subcommand needs. |
| `vim-cmd vmsvc/get.snapshotinfo <vmid>` | Display snapshot tree details for a specific VM — snapshot name, creation time, description, and whether memory was included. Useful before shutdown to check for orphaned snapshot chains. |
| `vim-cmd vmsvc/snapshot.get <vmid>` | Alternate snapshot query that returns a flat list of snapshot IDs for a VM. Use this when get.snapshotinfo output is unclear and you need bare snapshot identifiers. |
| `vim-cmd vmsvc/power.getstate <vmid>` | Check the current power state of a specific VM (Powered on, Powered off, Suspended). Useful to confirm a VM actually shut down after issuing a power-off. |
| `vim-cmd vmsvc/power.shutdown <vmid>` | Graceful guest OS shutdown for a specific VM via VMware Tools — equivalent to `Shutdown-VMGuest` in PowerCLI. Requires Tools to be running. |
| `vim-cmd vmsvc/power.off <vmid>` | Hard power-off a specific VM. No guest notification. Use only when power.shutdown does not work or VMware Tools is not running. |

**Full recovery sequence using vim-cmd:**

```sh
# 1. List all registered VMs and their IDs
vim-cmd vmsvc/getallvms

# 2. Check snapshot state before powering off
vim-cmd vmsvc/get.snapshotinfo <vmid>

# 3. Check current power state
vim-cmd vmsvc/power.getstate <vmid>

# 4. Graceful shutdown (requires VMware Tools)
vim-cmd vmsvc/power.shutdown <vmid>

# 5. If no response, hard power-off
vim-cmd vmsvc/power.off <vmid>

# 6. Repeat for each VM, then place host in maintenance mode
localcli system maintenanceMode set --enable true
```

---

## Step 5 — Audit Snapshots with PowerCLI

Once hostd is back up and the vSphere client can reconnect, check the snapshot state across all VMs before performing any shutdown or maintenance. Stale snapshots grow during host issues and can cause disk space exhaustion.

```powershell
# Connect to host or vCenter
$cred = Get-Credential
Connect-VIServer -Server YourHost -Credential $cred

# List all snapshots across all VMs
Get-VM | Get-Snapshot | Select-Object VM, Name, Created, SizeMB, Description

# Find snapshots older than 7 days
Get-VM | Get-Snapshot | Where-Object { $_.Created -lt (Get-Date).AddDays(-7) } |
    Select-Object VM, Name, Created, SizeMB |
    Sort-Object Created

# Find VMs with more than one snapshot (chains)
Get-VM | Where-Object { (Get-Snapshot -VM $_).Count -gt 1 } |
    Select-Object Name, @{N="SnapshotCount"; E={(Get-Snapshot -VM $_).Count}}
```

VMware recommends deleting snapshots within 24–72 hours for production VMs; chains that have grown large or aged significantly increase the risk of disk space exhaustion during consolidation. Orphaned snapshot chains are one of the most common causes of vSphere disk space emergencies.
