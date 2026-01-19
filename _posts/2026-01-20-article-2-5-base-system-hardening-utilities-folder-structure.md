---
layout: post
title: 'Article 2.5: Base System Hardening, Utilities & Folder Structure'
date: 2026-01-20 00:49 +0530
categories: [Home Server]
tags: [Docker, NAS, Homelab, Linux]

description: Hardening the base system, setting utilities, and defining structure so everything above it behaves predictably.

media_subpath: /assets/Home_Server/
---

**Laying the foundation**
-------------------------

### Purpose:

*   Make the system usable, debuggable, and predictable
*   Avoid “why is this broken?” moments later
*   Still no Docker yet

**1. DNS Configuration**
-------------------------

ISP-provided DNS is often unreliable. On Jio, it also blocks or interferes with certain domains and services.
The fix is simple: use a clean, external resolver and encrypt DNS queries.

Configure DNS and DNS-over-TLS
------------------------------

Edit the resolved configuration:

```
sudo vim /etc/systemd/resolved.conf
```

Set:

```
[Resolve]
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
DNSOverTLS=yes
```

Restart:

```
sudo systemctl restart systemd-resolved
```

This uses **Quad9** with DNS-over-TLS enabled.

What this gives you:

*   Reliable resolution
*   No ISP DNS interference
*   Encrypted DNS queries (prevents tampering and inspection)

You can use Google DNS instead, but Quad9 provides malware blocking by default.

**2. Essential Tools**
-------------------------

These tools are required for **monitoring, debugging, networking, and file movement**.

Monitoring
----------

*   `btop` – real-time CPU, disk, memory, and network usage

![](3.png)

Build / system
--------------

*   `build-essential` – required for compiling tools later (not mandatory, but useful)
*   `curl`, `wget` – API calls, downloads, testing endpoints

File transfer
-------------

*   `scp` – simple file copy over SSH
*   `rsync` – efficient, resumable transfers (used later for backups)

Media
-----

*   `yt-dlp` – youtube media downloader tool

Networking / diagnostics
------------------------

*   `ip a` – IP address details
*   `ip link` – interface status
*   `ping` – reachability
*   `nslookup`, `dig` – DNS debugging
*   `iperf3` – network throughput testing
*   `fast-cli` – quick internet speed check
*   `tailscale status` – Tailscale connectivity

Install all
-----------

```
sudo apt install \
  btop build-essential \
  curl wget rsync \
  iperf3 dnsutils iproute2
```

Install yt-dlp
--------------

Install uv first (the rust based client for python)

```
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install yt-dlp
```

```
#usage
yt-dlp '<playlist/video url>'
#if limit reaches add cookies
#get the cookies from browser using 'Get cookies.txt LOCALLY' extension
yt-dlp '<playlist/video url>' --cookies <file_path>
#send the cookies file from working device to server using scp on working device
scp <path_to_cookies> <user>@<server_ip>:<path_to_save_cookies>
```

Install fast-cli
----------------

Download the latest release from the releases section and unzip it to `/usr/local/bin/`

> [https://github.com/sindresorhus/fast-cli](https://github.com/sindresorhus/fast-cli)

Run using `fast-cli`

**3. Laptop-Specific Behaviour**
--------------------------------

Now, you might be using a laptop as the server. And a server looks cool when it works even if the lid is closed , which by default makes it sleep. So :

```
sudo vim /etc/systemd/logind.conf
```

Set:

```
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

Then:

```
sudo systemctl restart systemd-logind
```

**4. Wi-Fi Configuration**
--------------------------

> _Ethernet is always preferred_

So Wi-Fi is a:

*   Backup
*   Temporary
*   Laptop-only

**Setup**
---------

1.  Check interface starting with

```
ip link
```

2. TUI method (recommended)

```
sudo nmtui
#steps
#select 'Activate a connection'
#go to 'Wi-Fi' section and select the connection
#enter the password and you are good to go
```

3. CLI method (reference only)

```
sudo nmcli dev wifi rescan
sudo nmcli dev wifi list
sudo nmcli dev wifi --ask connect "SSID"
#enter the passoword
```

*   Server-grade reliability → Ethernet
*   Wi-Fi is acceptable for edge cases
*   By default, all the traffic goes through the ethernet

**5. Folder Structure on `/mnt/hdd4tb`**
----------------------------------------

**Data disk (`/mnt/hdd4tb`)**
-----------------------------

```
/mnt/hdd4tb/
├── media/
│   ├── movies/
│   ├── shows/
│   ├── anime/
│   ├── cartoon/
│   └── music/
├── torrents/
├── games/
├── immich/
└── files/
```

**Application layout (`/home/<user>/home_nas`)**
------------------------------------------------

```
/home/<user>/home_nas/
├── misc/
├── monitoring/
├── media_server/
├── immich_app/
├── file_server/
├── docker-compose.yml
└── .env
```

Principle:

*   **Actual data** lives on the HDD
*   **Container configs and compose files** live in the home directory
*   `/mnt/hdd1tb` is reserved for backups (covered later)

This separation:

*   Simplifies backups
*   Makes migrations trivial
*   Avoids Docker owning your data

> Reference guide (aligned with this approach):
> [https://trash-guides.info/File-and-Folder-Structure/](https://trash-guides.info/File-and-Folder-Structure/)

**What’s Next**
---------------

*   [**Article 3:** Docker setup, volume layout, service deployment](https://medium.com/@samarth_04/article-3-docker-setup-volume-layout-service-deployment-d3ca8da57542)
*   [**Article 4:** Backup strategy, rsync scripts, failure recovery](https://medium.com/@samarth_04/article-4-backup-strategy-rsync-scripts-failure-recovery-ffb22a6755ba)

**Prev**
--------

*   [**Article 1:** A Minimal Home NAS setup, that just works](https://medium.com/@samarth.jindal2004/article-1-a-minimal-home-nas-setup-that-just-works-a9e87d5fb745)
*   [**Article 2:** OS installation, disk preparation, SMART checks](https://medium.com/@samarth.jindal2004/article-2-os-installation-disk-preparation-smart-checks-6ceda3a128a5)
