+++
title = 'Docker Kernel Compilation Guide for Xiaomi Mi 10T (Apollon)'
date = '2025-11-23T07:31:49+07:00'
draft = false
tags = ["docker", "android", "kernel", "xiaomi"]
categories = ["devops", "mobile"]
+++

**Target Device:** Xiaomi Mi 10T / 10T Pro (apollon)  
**Chipset:** Snapdragon 865 (SM8250)  
**OS:** LineageOS 21/22 (Android 14/15)  
**Goal:** Enable native Docker support (Namespaces, Cgroups, OverlayFS) by patching and recompiling the kernel.

---

## 1. Prerequisites (PC Side)

Install the necessary build tools on Arch Linux:

```bash
sudo pacman -S base-devel git bc cpio zip unzip ncurses xmlto
sudo pacman -S aarch64-linux-gnu-gcc arm-none-eabi-gcc
````

-----

## 2\. Get Source Code & Tools

Create a working directory and clone the necessary repositories.

```bash
mkdir kernel_build && cd kernel_build

# 1. Clone the Kernel Source (SM8250 Common)
git clone https://github.com/LineageOS/android_kernel_xiaomi_sm8250 -b lineage-22.2 kernel
```

-----

## 3\. Prepare the Environment

Run these commands in your terminal **every time** before running `make`.

```bash
cd kernel
export ARCH=arm64
export SUBARCH=arm64
export PATH=$(pwd)/../clang/bin:$PATH
export CROSS_COMPILE=aarch64-linux-gnu-
export CROSS_COMPILE_ARM32=arm-none-eabi-
export CC=clang
```

-----

## 4\. Configuration (Defconfig)

The Mi 10T uses a fragmented configuration. You must merge the base, common, and device configs manually.

### A. Merge the Configs

```bash
# Combine Kona (Base) + SM8250 (Common) + Apollo (Device)
cat arch/arm64/configs/vendor/kona-perf_defconfig \
    arch/arm64/configs/vendor/xiaomi/sm8250-common.config \
    arch/arm64/configs/vendor/xiaomi/apollo.config \
    > arch/arm64/configs/vendor/docker_apollon_defconfig
```

### B. Load the Config

```bash
make O=out vendor/docker_apollon_defconfig
```

### C. Enable Docker Flags

Run `make O=out menuconfig` and enable the following:

  * **General setup**
      * `[*] POSIX Message Queues`
      * **Control Group support**
          * `[*] Memory controller`
          * `[*] Device controller`
      * **Namespaces support**
          * `[*] User namespace`
          * `[*] PID Namespaces`
          * `[*] IPC Namespaces`
  * **Networking support** -\> **Networking options** -\> **Network packet filtering framework (Netfilter)**
      * `[*] Bridged IP/ARP packets filtering`
      * **Core Netfilter Configuration** -\> **Netfilter Xtables support** -\> `[*] addrtype address type match support`
  * **File systems**
      * `<*> Overlay filesystem support`
  * **Disable WError** (Crucial for modern compilers)
      * Search for `CC_WERROR` (Press `/`) -\> Uncheck `[ ] Compile the kernel with warnings as errors`

-----

## 5\. Applying Fixes (Modern Compiler Patching)

The kernel code is from \~2020. Modern GCC/Clang is too strict. We must patch the `Makefile` and source code to ignore specific errors.

### A. Patch the Makefile (Ignore Flags)

Add the following compiler flags to the `Makefile` to suppress specific errors:

  * Ignore formatting errors: `-Wno-error=format`
  * Ignore enum/int mismatch (Wi-Fi driver): `-Wno-error=enum-int-mismatch`
  * Ignore indentation issues: `-Wno-error=misleading-indentation`
  * Ignore array parameter mismatches: `-Wno-error=array-parameter`
  * Ignore address checks: `-Wno-error=address`
  * Ignore uninitialized variables: `-Wno-error=maybe-uninitialized -Wno-error=uninitialized`

Here are the differences compared to the original Makefile:

```diff
diff --git a/Makefile b/Makefile
index 40d2f3742..9fb6e87f9 100644
--- a/Makefile
+++ b/Makefile
@@ -444,7 +444,7 @@ LINUXINCLUDE    := \
                 $(USERINCLUDE)
 
 KBUILD_AFLAGS   := -D__ASSEMBLY__
-KBUILD_CFLAGS   := -Wall -Wundef -Wstrict-prototypes -Wno-trigraphs \
+KBUILD_CFLAGS   := -Wall -Wno-error=format -Wno-error=enum-int-mismatch -Wno-error=misleading-indentation -Wno-error=array-parameter -Wno-error=address -Wno-error=maybe-uninitialized -Wno-error=uninitialized -Wundef -Wstrict-prototypes -Wno-trigraphs \
                    -fno-strict-aliasing -fno-common -fshort-wchar \
                    -Werror-implicit-function-declaration \
                    -Werror=return-type -Wno-format-security \
```

### B. Patch Source Code (`bpf_trace.c`)

The `inline` keyword causes an error in this specific file.

```diff
diff --git a/kernel/trace/bpf_trace.c b/kernel/trace/bpf_trace.c
index 82fe096cb..d6a9d290e 100644
--- a/kernel/trace/bpf_trace.c
+++ b/kernel/trace/bpf_trace.c
@@ -360,7 +360,7 @@ static DEFINE_RAW_SPINLOCK(trace_printk_lock);
 
 #define BPF_TRACE_PRINTK_SIZE   1024
 
-static inline __printf(1, 0) int bpf_do_trace_printk(const char *fmt, ...)
+static __printf(1, 0) int bpf_do_trace_printk(const char *fmt, ...)
 {
         static char buf[BPF_TRACE_PRINTK_SIZE];
         unsigned long flags;
```

-----

## 6\. Compilation

Build the kernel. This takes 10–20 minutes.

```bash
make O=out -j$(nproc) Image.gz-dtb
```

  * **Result:** File located at `out/arch/arm64/boot/Image.gz-dtb` (\~22MB).

-----

## 7\. Packaging (AnyKernel3)

We use AnyKernel3 to create a flashable zip.

```bash
# 1. Get AnyKernel3
cd ..
git clone https://github.com/osm0sis/AnyKernel3
cd AnyKernel3

# 2. Clean dummy files
rm Image zImage dtb 2>/dev/null

# 3. Copy your new kernel
cp ../kernel/out/arch/arm64/boot/Image.gz-dtb .
```

### **CRITICAL: Fix `anykernel.sh` for Mi 10T**

Edit `anykernel.sh` and make these specific changes:

1.  **Disable Device Check:**
    `do.devicecheck=0`
2.  **Hardcode Partition Path:**
    `block=/dev/block/bootdevice/by-name/boot`

### **Zip the Installer**

```bash
zip -r9 ../Docker-Kernel-Apollon.zip * -x .git README.md *placeholder
```

-----

## 8\. Installation

1.  **Backup:** Boot into Recovery (Lineage/TWRP) and backup the `Boot` partition (if possible).
2.  **Flash:**
      * On Phone: Apply Update -\> Apply from ADB.
      * On PC: `adb sideload Docker-Kernel-Apollon.zip`
3.  **Reboot:** Restart System.

-----

## 9\. Post-Install Setup (Termux)

Once booted, open Termux (Root) and verify the kernel:

```bash
su
zcat /proc/config.gz | grep "CONFIG_CGROUPS" # Should be =y
```

### Install [Termux:Boot](https://wiki.termux.com/wiki/Termux:Boot)

Create the following scripts inside the `~/.termux/boot` directory.

### sshd Auto Start (`00-start-sshd.sh`)

```bash
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock
sshd
```

### Auto-Start Script (`01-start-docker.sh`)

**Note:** Replace `<IP_GATEWAY>` below with your actual router IP (e.g., `192.168.2.254`).

```bash
#!/data/data/com.termux/files/usr/bin/bash
# 1. Switch to Root to set up Kernel & Docker
# We use 'su -c' to run these commands as root silently
su -c '
# Mount the "wires" (Cgroups)
mount -t tmpfs -o mode=755 tmpfs /sys/fs/cgroup
mkdir -p /sys/fs/cgroup/devices
mount -t cgroup -o devices cgroup /sys/fs/cgroup/devices
mkdir -p /sys/fs/cgroup/cpu
mount -t cgroup -o cpu cgroup /sys/fs/cgroup/cpu
mkdir -p /sys/fs/cgroup/memory
mount -t cgroup -o memory cgroup /sys/fs/cgroup/memory

# Fix routing to use bridge network
iptables -P FORWARD ACCEPT
iptables -I FORWARD -j ACCEPT

iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

# Fix Android Policy Routing (Allow Docker traffic to use Main table)
ip rule add pref 1 from all lookup main
# Restore Gateway (Android wipes this from Main table)
ip route add default via <IP_GATEWAY> dev wlan0 2>/dev/null

# Fix Path for Root
export PATH=/data/data/com.termux/files/usr/bin:$PATH

# Start Docker Daemon silently
dockerd > /dev/null 2>&1 &

# Allow User to Run Docker (Non-Root)
sleep 5  # Wait for socket to be created
chmod 666 /data/data/com.termux/files/usr/var/run/docker.sock
'
```

### Deploy Nginx Container

Now you can run containers as a normal user:

```bash
docker run -d --rm -p 8080:80 nginx:latest
```

Try accessing it at `<Phone_IP>:8080`.
