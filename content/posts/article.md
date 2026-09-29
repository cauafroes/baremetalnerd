---
title: "Turn an old android into a Home Server with chroot"
date: 2026-09-27T10:00:00Z
draft: false
tags: ["linux", "android", "chroot", "hardware", "embedded", "raspberry-pi"]
categories: ["Bare Metal"]
summary: "Why buy a Raspberry Pi when your old smartphone has faster UFS storage, an integrated UPS battery, and an octa-core ARM processor? Here is how and why chrooting Ubuntu on a Galaxy S8 creates the ultimate budget home server."
---

If you have an old flagship smartphone like a Samsung Galaxy S8 sitting in a drawer gathering dust, you are holding a high-performance, power-efficient micro-server. While single-board computers (SBCs) like the Raspberry Pi remain the default choice for self-hosters and home lab enthusiasts, old smartphones often outperform them in raw specs, built-in redundancy, and overall value.

By leveraging **`chroot`** (change root) on a rooted Android device, you can bypass the performance limitations of user-space terminal emulators like Termux PRoot and run a native ARM64 Ubuntu environment alongside the Linux kernel.

---

## Hardware Breakdown: Galaxy S8 vs. Raspberry Pi 4

When comparing single-board computers to decommissioned flagship hardware, the specs of the smartphone speak for themselves:

| Hardware Component | Samsung Galaxy S8 (Exynos 8895 / SD 835) | Raspberry Pi 4 (4GB Model) |
| :--- | :--- | :--- |
| **CPU Architecture** | Octa-Core (4x2.3 GHz + 4x1.7 GHz) | Quad-Core Cortex-A72 @ 1.5 GHz |
| **Storage Speed** | 64GB **UFS 2.1** (~500 MB/s read) | MicroSD Card (~30–90 MB/s read) |
| **Built-in Power Backup** | 3,000 mAh Li-Ion (Integrated UPS) | None (Requires external UPS) |
| **Network & Wireless** | Wi-Fi 5 (802.11ac), Bluetooth 5.0, LTE | Wi-Fi 5 (802.11ac), Bluetooth 5.0, Gigabit Ethernet |
| **Display / Touch** | Integrated 1440p Super AMOLED Display | Requires external HDMI monitor |
| **Cost** | **$0** (Reused hardware) | ~$55–$75+ (Board only, no power/storage) |

---

## Why a Chrooted Smartphone Wins

### 1. Superior Storage Performance (UFS 2.1 vs. MicroSD)
The primary bottleneck on most Raspberry Pi setups is the MicroSD card. MicroSD cards offer poor random write performance and frequently fail under database-heavy workloads (e.g., SQLite, Docker containers, logging daemons). 

The Galaxy S8 utilizes onboard **UFS 2.1 flash memory**, delivering read/write performance comparable to an entry-level SATA SSD:

```text
# MicroSD Sequential Write Speed (Raspberry Pi)
~25 MB/s to 45 MB/s

# Galaxy S8 UFS 2.1 Sequential Write Speed
~180 MB/s to 220 MB/s
```

### 2. Built-In Battery Backup (Uninterruptible Power Supply)
A momentary power flicker can corrupt a Raspberry Pi’s filesystem if it is writing to disk. A smartphone comes with a built-in battery that acts as an integrated **UPS**. If your mains power drops, your home server continues running, keeping network sockets, background daemons, and cron jobs active without interruption.

### 3. Integrated Diagnostics & Status Display
When a headless server drops off the local network, debugging a Raspberry Pi requires an HDMI adapter and a external monitor. An old phone provides an integrated, high-density touch display right out of the box—ideal for rendering system statistics via `htop` or showing custom server dashboards.

---

## What Is `chroot` and Why Not Use PRoot?

Most Android Linux tutorials rely on **PRoot** (via Termux), which uses `ptrace` in user space to simulate root access and intercept system calls without unlocking the bootloader. 

While PRoot is easy to set up, system call interception incurs significant CPU overhead:

```text
User Application ---> [ ptrace overhead ] ---> PRoot Layer ---> Android Kernel
```

With **`chroot`**, you execute commands directly against the underlying Android Linux kernel with full root privileges:

```text
User Application ---> Native System Calls ---> Android Linux Kernel
```

Because `chroot` uses native system calls without virtualization or call-trapping layers, your Ubuntu environment runs at **100% native hardware speeds**.

---

## Setting Up Ubuntu via `chroot` on Android

### Prerequisites
* A rooted Samsung Galaxy S8 (or any ARM64 Android device with root access).
* `adb` installed on your workstation or a terminal app on the device.
* BusyBox installed on the phone.
* An Ubuntu ARM64 rootfs image (e.g., `ubuntu-base-22.04-base-arm64.tar.gz`).

### Step 1: Create the Chroot Directory Structure
Connect to the phone via ADB shell as root:

```bash
adb shell
su
mkdir -p /data/local/ubuntu
```

### Step 2: Extract the Root Filesystem
Transfer and extract the official Ubuntu ARM64 base image into your chroot target:

```bash
cd /data/local/ubuntu
tar -xzvf /sdcard/Download/ubuntu-base-22.04-base-arm64.tar.gz
```

### Step 3: Mount Virtual Filesystems
To give the chroot environment access to system hardware, process tables, and network interfaces, mount `dev`, `proc`, and `sys`:

```bash
mount -o bind /dev /data/local/ubuntu/dev
mount -t proc proc /data/local/ubuntu/proc
mount -t sysfs sys /data/local/ubuntu/sys
mount -t devpts devpts /data/local/ubuntu/dev/pts
```

### Step 4: Enter the Chroot Environment
Set up DNS resolution and drop into the chroot shell:

```bash
cp /etc/resolv.conf /data/local/ubuntu/etc/
chroot /data/local/ubuntu /bin/bash
```

Once inside, configure basic environment variables:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export HOME=/root
apt update && apt install -y htop open-ssh-server neofetch curl git
```

---

## Best Use Cases for a Chrooted Smartphone Server

* **Self-Hosted Services:** Run lightweight servers like Pi-hole, AdGuard Home, or Syncthing.
* **Low-Power Web Server:** Host static sites (including Hugo!) using Nginx or Caddy.
* **MQTT & Home Automation:** Serve as an MQTT broker (`mosquitto`) or Node-RED automation node.
* **Network Storage / Backup:** Attach USB storage OTG docks to turn the phone into an ultra-portable NAS or rsync target.

---

## Conclusion

Before buying another single-board computer for your home lab, look into your drawer of old gadgets. With native `chroot` access, a recycled smartphone like the Galaxy S8 provides a faster, safer, and self-powered Linux node completely free of charge.