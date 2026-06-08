# ESXi CPU Temperature Monitor: A Shell Script for Real-Time IPMI Alerting

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** VMware / ESXi  
**Tags:** ESXi, VMware, IPMI, Shell Script, BMC, Thermal Monitoring, DevOps, Infrastructure

---

ESXi does not ship with a continuous, color-coded temperature dashboard. If you manage bare-metal hosts in a lab or production environment and want immediate visual feedback when a CPU runs hot — without standing up a full monitoring stack — this script covers the gap in under 30 lines of POSIX shell.

---

## Script Variables Reference

| Variable | Role | Notes |
|---|---|---|
| `THRESHOLD=70` | Alert boundary | Temperatures above this value are printed in red. Adjust based on your CPU's Tcase spec — Intel Xeon E5 Tcase (case temperature) is typically 67–82 °C depending on SKU; Tj-max (die junction) is higher at 91–105 °C. Set the threshold against Tcase, not Tj-max. |
| `INTERVAL=10` | Refresh cadence | Seconds between full sensor sweeps. Lower values increase resolution but add negligible load. 5–10 s is appropriate for active monitoring; 30 s for background logging. |
| `esxcli hardware ipmi sdr list` | IPMI sensor read | Queries the BMC (Baseboard Management Controller) via the in-band IPMI driver. Returns all SDR entries — temperatures, voltages, fan speeds — as plain text. |
| `grep "CPU"` | Sensor filter | Narrows the SDR output to CPU temperature sensors only. Modify to `"Temp"` or `"Fan"` to target different sensor classes. |
| `\033[31m … \033[0m` | ANSI red highlight | ANSI escape sequence for red foreground. ESXi's BusyBox shell supports `echo -e` for escape interpretation. `\033[0m` resets the terminal color after the value. |

---

## IPMI Facts

| Topic | Detail |
|---|---|
| In-band vs. Out-of-band | `esxcli` queries IPMI in-band — through the host OS kernel driver rather than a dedicated BMC network port. This means the script works without a separate iDRAC/iLO network connection configured. |
| SDR Cache | The first `esxcli sdr list` call may take 1–3 seconds to prime the SDR cache. Subsequent calls within the same session are faster. This is normal behavior and does not indicate a hardware issue. |
| Sensor Naming Varies | Dell servers label CPU temps as "CPU1 Temp" / "CPU2 Temp", HP iLO uses "CPU 01", Supermicro uses "CPU Temp". Verify the exact label on your platform with a one-off `esxcli hardware ipmi sdr list | grep -i cpu` before deploying. |
| Non-numeric Readings | The `[ -z "$temp" ]` guard catches sensors that report "N/A" or are not present (e.g., a single-CPU system querying CPU2). Without this guard the integer comparison crashes the loop. |

---

## Why IPMI?

VMware ESXi exposes hardware sensor data through the `esxcli hardware ipmi sdr list` command, which queries the host's BMC (Baseboard Management Controller) via the in-kernel IPMI driver. This is the same data source your server's iDRAC, iLO, or IPMI web interface reads — except here it's accessible directly from the ESXi shell without any separate network configuration or out-of-band management port.

The SDR (Sensor Data Repository) contains every hardware sensor the BMC knows about: CPU temperatures, inlet/exhaust temps, fan speeds, voltages, power draw. The script filters this down to CPU temperature rows only and presents them in a live-refreshing view.

---

## The Complete Script

```sh
#!/bin/sh
# CPU Temperature Monitor for ESXi
# Continuously monitors CPU temps, prints in red if >70C

THRESHOLD=70      # Temperature threshold for alert (C)
INTERVAL=10       # Seconds between checks

while true; do
    clear
    echo "CPU Temperature Monitor - $(date -u)"
    echo "---------------------------------------"

    # Get CPU temperatures from IPMI sensor list
    esxcli hardware ipmi sdr list | grep "CPU" | while read -r line; do
        # Extract sensor name and temperature value (numbers only)
        name=$(echo "$line" | awk '{print $4,$5}')
        temp=$(echo "$line" | awk '{print $7}' | grep -o '[0-9]\+')

        # Check if temperature is numeric
        if [ -z "$temp" ]; then
            temp=0
        fi

        # Display in red if above threshold
        if [ "$temp" -gt "$THRESHOLD" ]; then
            echo -e "$name: \033[31m${temp}C\033[0m"
        else
            echo "$name: ${temp}C"
        fi
    done

    echo "---------------------------------------"
    sleep $INTERVAL
done
```

---

## How the Color Logic Works

The highlight mechanism is a simple integer comparison feeding into ANSI escape codes — the same codes used by terminal emulators since the 1970s.

When a temperature exceeds the threshold, the script wraps the value in `\033[31m` (set foreground to red) and `\033[0m` (reset all attributes). The ESXi BusyBox shell's `echo -e` flag is required to interpret these escape sequences — plain `echo` without `-e` will print the literal escape string instead of rendering the color.

The session stays alive because the outer `while true` loop never exits. The `sleep $INTERVAL` call at the end of each cycle prevents the loop from consuming a full CPU core.

---

## Deploying the Script

Save the script to a persistent datastore path — the ESXi ramdisk (`/tmp`) is wiped on reboot. A recommended location is your local datastore:

```sh
# Save to a persistent datastore
vi /vmfs/volumes/datastore1/scripts/cpu_monitor.sh

# Make it executable
chmod +x /vmfs/volumes/datastore1/scripts/cpu_monitor.sh

# Run it
/vmfs/volumes/datastore1/scripts/cpu_monitor.sh
```

To start it automatically after a host reboot, add the call to `/etc/rc.local.d/local.sh` using `nohup` so it persists without an interactive session:

```sh
# In /etc/rc.local.d/local.sh — runs at boot
nohup /vmfs/volumes/datastore1/scripts/cpu_monitor.sh \
  > /vmfs/volumes/datastore1/scripts/cpu_monitor.log 2>&1 &
```

With this in place, the monitor starts silently at boot, logs to the datastore path (where you can `grep` for past thermal events), and you can attach to its output with `tail -f` at any time. The log is on the datastore so it survives host reboots — do not redirect it to `/tmp`, which is a ramdisk wiped on every reboot.

---

## Tuning for Your Environment

Two variables cover most customisation needs without touching the script logic.

**`THRESHOLD=70`**  
- Intel Xeon E5/E7 series: set to 75–80 (against Tcase; check your specific SKU datasheet).
- AMD EPYC: check AMD's published Tccd and Tctl limits — these CPUs report die temperature (Tdie) which runs higher than traditional Tcase.
- Consumer CPUs in a lab setting: 65 is a conservative starting point.

**`INTERVAL=10`**  
- 5 s — active troubleshooting during a workload spike.
- 10 s — normal monitoring, default.
- 30 s — background logging without `clear`, append output to a file instead.

If your server has more than two CPUs or uses a non-standard naming convention, change `grep "CPU"` to `grep -i "temp"` to catch all temperature sensors regardless of label.

---

## Limitations and What This Doesn't Replace

This script is a lightweight operational tool, not a replacement for a proper monitoring stack. It has no persistent alerting, no trend history, no multi-host view, and no integration with incident management. For production environments, pair it with Prometheus + node_exporter (IPMI exporter), vCenter alarms, or your organisation's SIEM.

Where this script excels is in situations where none of the above are available: a standalone lab ESXi host, a quick post-maintenance thermal check, or an environment where deploying Prometheus is more overhead than the task warrants.
