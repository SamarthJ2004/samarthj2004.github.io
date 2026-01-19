---
layout: post
title: 'Article 2: OS installation, disk preparation, SMART checks'
date: 2026-01-20 00:48 +0530
categories: [Home Server]
tags: [Docker, NAS, Homelab, Linux]

description: Selecting the base system and making sure the ground is stable before anything is built on it.

media_subpath: /assets/Home_Server/
---

**Choosing and preparing the land**
-----------------------------------

### This article covers:

*   Installing the operating system
*   Preparing and mounting disks
*   Verifying disk health before trusting them with data

OS Installation
---------------

The OS used is **Debian**.
There are many good Debian installation tutorials available online, so this section focuses on **practical decisions**, not screenshots.

Create a Bootable Installer
---------------------------

1.  Take a USB pendrive
2.  Install **Ventoy** on it
3.  Copy the Debian ISO onto the drive

> **Downloads**
> 
> Ventoy: [https://github.com/ventoy/Ventoy/releases/download/v1.1.10/ventoy-1.1.10-linux.tar.gz](https://github.com/ventoy/Ventoy/releases/download/v1.1.10/ventoy-1.1.10-linux.tar.gz)
> 
> Debian ISO: [https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.2.0-amd64-netinst.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.2.0-amd64-netinst.iso)

Ventoy lets you keep multiple ISOs on one USB and avoids re-flashing every time.

*   Connect: External display (TV or monitor), Ethernet cable, USB drive
*   Power on the system
*   Repeatedly press the **boot menu key**
*   Select the USB drive
*   Ventoy menu appears → select the Debian ISO
*   Continue with the installation

Installer Choices (Important)
-----------------------------

During installation:

*   **I did not set a root password**
    This enables sudo access for your user. This is just a workaround and not the ideal practise.
*   I installed a lightweight GUI (**XFCE**) for emergency or future use.
    This is optional. You can run headless-only if you prefer.

Select these components:

*   XFCE
*   SSH server
*   System utilities
*   Debian desktop environment

Once installation completes, reboot.

Network Setup and SSH Access
----------------------------

DHCP Reservation (Do This Early)
--------------------------------

Before disconnecting the screen:

1.  Run: `ip a`
2.  Identify the Ethernet interface (`enp…`)
3.  Note its MAC address
4.  Create a DHCP reservation in your router
    Example: `MAC → 192.168.x.x`

This prevents IP changes after reboots.

Install Tailscale
-----------------

Install dependencies:

```
sudo apt install curl
```

Install **Tailscale**:

```
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring the node online:

```
# For browser
sudo tailscale up
# For using the auth key
sudo tailscale up --authkey tskey-auth-<authkey>
```

At this point:

*   The NAS is reachable over Tailscale
*   You no longer need a screen or keyboard

SSH Into the Server
-------------------

From your main laptop:

```
ssh user@<tailscale-ip>
```

or

```
ssh user@192.168.x.x
```

Accept the fingerprint and enter the password.
From now on, everything is remote.

Disk Preparation and Mounting
-----------------------------

Identify Disks
--------------

List block devices: `lsblk`

Get UUIDs: `sudo blkid`

Example devices:

*   `/dev/sda` → 4 TB HDD
*   `/dev/sdb` → 1 TB HDD

Create Mount Points
-------------------

for me (you can name as per your convenience):

```
sudo mkdir -p /mnt/hdd4tb /mnt/hdd1tb
```

Configure Automatic Mounting
----------------------------

Install an editor (use what you prefer): I prefer vim, but for convenience I will once tell both and then continue without caring about the specifics.

`sudo vim /etc/fstab` or `sudo nano /etc/fstab`: for loading automatic mounting of the HDDs on startup.

For nano do `crtl + o` to save and `crtl + x` to exit

For vim do `i` to enter insert mode , make the changes and then press `esc` then `:wq` to save and exit

> Want to learn vim? just complete the vim tutorial using vimtutor

```
sudo apt install vim
```

Add entries:

```
UUID=<uuid-of-4tb> /mnt/hdd4tb ext4 defaults 0 2
UUID=<uuid-of-1tb> /mnt/hdd1tb ext4 defaults 0 2
```

Reload and mount:

```
sudo systemctl daemon-reload
sudo mount -a
```

Verify:

```
lsblk
df -h
```

You should now see both disks mounted correctly.

Disk Health & SMART Checks (Critical)
-------------------------------------

Before trusting _any_ disk with data, check its health.

Install **smartmontools**:

```
sudo apt install smartmontools
```

Detect disks:

```
sudo smartctl --scan
```

Check a disk:

```
sudo smartctl -a /dev/sdX
```

(replace `sdX` with the actual device)

> You can use the CrystalDiskInfo app for Windows

What to Check (New Drives)
--------------------------

Now, the most important , lets check if the drive are good or not (new and working?)

Key attributes to look at:

*   **Power_On_Hours :** Should be very low for a new disk, ideally 0/1 but less than 50 works
*   **Reallocated_Sector_Ct :** Must be **0**
*   **Current_Pending_Sector :** Must be **0**
*   **Offline_Uncorrectable :** Must be **0**
*   **Read / Write Error Rates :** Should not show abnormal growth

If any of the above are non-zero on a _new_ drive, return it immediately.

SMART does **not** prevent failures.
It only gives early warning. Backups still matter more.

What’s Next
-----------

*   [**Article 2.5:** Base System Hardening , Utilities & Folder Structure](https://medium.com/@samarth_04/article-2-5-base-system-hardening-utilities-folder-structure-d22fc190a668)
*   [**Article 3:** Docker setup, volume layout, service deployment](https://medium.com/@samarth_04/article-3-docker-setup-volume-layout-service-deployment-d3ca8da57542)
*   [**Article 4:** Backup strategy, rsync scripts, failure recovery](https://medium.com/@samarth_04/article-4-backup-strategy-rsync-scripts-failure-recovery-ffb22a6755ba)

Prev
----

*   [**Article 1:** A Minimal Home NAS setup, that just works](https://medium.com/@samarth.jindal2004/article-1-a-minimal-home-nas-setup-that-just-works-a9e87d5fb745)
