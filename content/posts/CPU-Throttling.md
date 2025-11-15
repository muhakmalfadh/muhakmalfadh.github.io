+++
title = 'CPU Throttling'
date = '2025-11-15T10:17:15+07:00'
draft = false
tags = [ "linux", "throttling", "bd_prochot", "msr", "systemd", "power management"]
categories = [ "sysadmin" ]
+++

**Summary:** *If your modern Linux laptop locks its CPU speed to the absolute minimum (often 200MHz or 400MHz) when on battery, despite power profiles being set to 'Balanced,' the problem is likely **firmware**—not software. This guide provides the definitive hardware workaround using Model Specific Registers (MSRs) to disable the rogue BD\_PROCHOT signal.*

## 1\. The Frustrating Symptom

You've got a powerful laptop, running a stable distribution like Omarchy OS (or Arch, Fedora, etc.).

  * **Plugged In:** The CPU runs perfectly, hitting its advertised boost speeds ($3\text{ GHz}$ or more).
  * **On Battery:** The CPU instantly locks to its minimum frequency, often **200MHz** or **400MHz**. The system becomes sluggish, even under zero load.
  * **The Check:** You check your power settings:
      * `powerprofilesctl get` shows **`balanced`**.
      * The CPU governor is correctly set to the dynamic **`powersave`** (when using Intel P-State).
        * ```bash
          $ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
          powersave
          ```
      * No obvious culprits like TLP or `auto-cpufreq` are installed.

This behavior is a strong indicator that the operating system has been **overridden by the hardware's firmware**.

-----

## 2\. The Root Cause: BD\_PROCHOT and the New Battery

The likely cause, especially if you recently replaced the laptop battery, is the **BD\_PROCHOT** (Bi-Directional Processor Hot) signal.

  * **What it is:** BD\_PROCHOT is a safety signal asserted by the laptop's **Embedded Controller (EC)**. It forces the CPU to hard-throttle to its minimum frequency regardless of the OS settings.
  * **Why it Triggers:** Laptops are often programmed to trigger this safety signal if the EC detects a potential power fault. A common scenario is when a **replacement or non-OEM battery** is installed, and the EC cannot verify its power signature or accurately read its status. The EC defaults to maximum safety, resulting in the aggressive $200\text{ MHz}$ lock.

Since this signal operates below the kernel level, a software configuration change won't fix it—we need a surgical software workaround.

-----

## 3\. The Definitive Fix: Disabling BD\_PROCHOT via MSR

The solution is to use the **`wrmsr`** (Write Model Specific Register) utility to tell the CPU to **ignore** the BD\_PROCHOT signal.

### Step 3.1: Create the MSR-Write Script

The safest way to disable the signal is to read the register's current value, clear only the single bit responsible for the throttle (Bit 0), and write the modified value back.

First, ensure you have the `msr-tools` package installed (e.g., `sudo pacman -S msr-tools`).

Create the script at `/usr/local/bin/disable-bd-prochot.sh`:

```bash
#!/bin/bash
set -euo pipefail

echo "[BD_PROCHOT] Starting disable script..."

# Load msr module if not already loaded
if ! lsmod | grep -q '^msr'; then
    echo "[BD_PROCHOT] Loading msr module..."
    modprobe msr || { echo "[BD_PROCHOT] Failed to load msr module!"; exit 1; }
fi

# Wait until /dev/cpu/0/msr exists (some systems take a moment)
for i in {1..5}; do
    if [ -e /dev/cpu/0/msr ]; then
        break
    fi
    echo "[BD_PROCHOT] Waiting for /dev/cpu/0/msr..."
    sleep 1
done

if [ ! -e /dev/cpu/0/msr ]; then
    echo "[BD_PROCHOT] Error: /dev/cpu/0/msr not found!"
    exit 1
fi

# Read current MSR (0x1FC) value from core 0
CURRENT_VAL=$(rdmsr -d 0x1FC) || {
    echo "[BD_PROCHOT] Failed to read MSR!"
    exit 1
}

echo "[BD_PROCHOT] Current MSR value: $CURRENT_VAL"

# Check if BD_PROCHOT bit is set (bit 0)
if (( CURRENT_VAL % 2 == 1 )); then
    NEW_VAL=$((CURRENT_VAL - 1))
    echo "[BD_PROCHOT] Disabling BD_PROCHOT (new value: $NEW_VAL)"
    wrmsr -a 0x1FC "$NEW_VAL"
    echo "[BD_PROCHOT] BD_PROCHOT cleared on all cores."
else
    echo "[BD_PROCHOT] BD_PROCHOT already disabled."
fi

echo "[BD_PROCHOT] Done."
```

**Important:** Make the script executable: `sudo chmod +x /usr/local/bin/disable-bd-prochot.sh`

### Step 3.2: Create the Systemd Service

Since MSR settings are volatile (they reset on every reboot), we must create a `systemd` service to run this script automatically at startup.

Create the service file at `/etc/systemd/system/bd-prochot-fix.service`:

```ini
[Unit]
Description=Disable BD_PROCHOT to fix 200MHz throttling
# Ensure the service runs AFTER the required kernel modules and filesystems are ready
After=sysinit.target local-fs.target
Requires=sysinit.target

[Service]
Type=oneshot
# The script will load the msr module and run wrmsr
ExecStart=/usr/local/bin/disable-bd-prochot.sh
StandardOutput=journal
StandardError=journal

[Install]
# Attach the service to the standard environment that desktop/multi-user systems use.
WantedBy=multi-user.target
```

### Step 3.3: Enable and Start the Service

Finally, apply the service changes and enable it permanently:

```bash
sudo systemctl daemon-reload           # Reload systemd manager
sudo systemctl enable --now bd-prochot-fix.service  # Enable and run immediately
```

## 4\. Final Thoughts on Safety

Disabling BD\_PROCHOT is a powerful **workaround** that removes a hardware-level safety switch. While modern CPUs have internal temperature protection, it is always recommended to **monitor your CPU temperatures** when running intensive tasks after applying this fix, especially on battery.

To check your logs and ensure the script ran correctly after a reboot:

```bash
journalctl -u bd-prochot-fix.service -b
```