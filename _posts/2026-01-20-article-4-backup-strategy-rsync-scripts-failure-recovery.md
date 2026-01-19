---
layout: post
title: 'Article 4: Backup Strategy, Rsync Scripts, Failure Recovery'
date: 2026-01-20 00:49 +0530
categories: [Home Server]
tags: [Docker, NAS, Homelab, Linux]

description: Preparing for failures — power loss, disk death, and human error — so the system survives when things go wrong.

media_subpath: /assets/Home_Server/
---
**Safety, redundancy, and disaster readiness**
----------------------------------------------

This article covers the **most important part of a NAS : Backups and recovery.**

My goal is:

> If something breaks, I should know exactly what to restore and how.

**Data Classification (Critical Step)**
---------------------------------------

Before writing scripts, data must be classified : What you want to backup and what not? This is more important and require more thinking than writing the script which can be written using GPTs (no need to learn bash just for it), like what I did.

**Primary Data Disk — `/mnt/hdd4tb`**
-------------------------------------

Contains:

*   Media
*   Torrents
*   Games
*   Files
*   Photos and Videos

**This disk is NOT fully backed up.**

**Backup Disk — `/mnt/hdd1tb`**
-------------------------------

Contains **only irreplaceable data**:

*   Photos and videos
*   Important documents
*   Immich library (final processed output)
*   Docker compose files

This disk exists **only for backups**.

**Backup Model**
----------------

*   Tool: `rsync`
*   Frequency: **daily**
*   Retention: **last 180 days**
*   Direction:
    `/mnt/hdd4tb → /mnt/hdd1tb` and `/home/<user>/home_nas → /mnt/hdd1tb`

**Creating the Backup View**
----------------------------

The problem with backing up data from true paths:

```
/mnt/hdd4tb
/home/<user>/home_nas
```

*   Files may change while rsync is reading them which may lead to errors , corrupted backups and inconsistencies.

By creating a stable backup view that contains read only bind mounts of required data, we are free from such problems. So it creates a safety boundary.

**Directory Layout on Backup Disk**
-----------------------------------

```
/mnt/hdd1tb/backups/
├── snapshots/
│   ├── 2026-01-06/
│   │   ├── hdd4tb/
│   │   └── home_nas/
│   ├── 2026-01-07/
│   ├── ...
│   └── 2026-01-12/
│   │   ├── hdd4tb/
│   │   └── home_nas/
└── daily/
    └── latest -> /mnt/hdd1tb/backups/snapshots/2026-01-12
```

*   Each snapshot is like a full backup
*   Snapshots share unchanged files using hard links
*   `latest` always points to the most recent backup
*   Each snapshot take up space for the incremental changes only.

This looks like 180 full backups but **does not use 180× space**.

**The rsync Script**
--------------------

Create a script:

```
sudo vim /usr/local/bin/snapshot_backup.sh
```

**snapshot_backup.sh**
-----------------------

```
#!/bin/bash
set -euo pipefail
################ CONFIG ################
BACKUP_ROOT="/mnt/hdd1tb/backups"
SNAPSHOTS="$BACKUP_ROOT/snapshots"
VIEW="/home/<user>/backup_view"
HOME_NAS="/home/<user>/home_nas"
RETENTION_DAYS=180
MIN_FREE_GB=10
LOG="/var/log/snapshot_backup.log"
LOCK_FILE="/var/lock/snapshot_backup.lock"
LOCK_FD=9
TODAY="$(date +%F)"
#######################################
cleanup_mounts() {
  for p in \
    "$VIEW/hdd4tb/files" \
    "$VIEW/hdd4tb/immich" \
    "$VIEW/home_nas/immich_app"
  do
    mountpoint -q "$p" && umount "$p" || true
  done
}
finish() {
  status=$?
  cleanup_mounts
  if [ "$status" -eq 0 ]; then
    echo "=== Backup finished successfully at $(date) ===" >> "$LOG"
  else
    echo "=== Backup FAILED at $(date) (exit code $status) ===" >> "$LOG"
  fi
}
trap finish EXIT
#######################################
# Safety checks
#######################################
[ "$(id -u)" -eq 0 ] || { echo "Must run as root" >> "$LOG"; exit 1; }
for cmd in rsync mount umount df flock date mountpoint; do
  command -v "$cmd" >/dev/null || { echo "$cmd missing" >> "$LOG"; exit 1; }
done
exec {LOCK_FD}>"$LOCK_FILE"
flock -n "$LOCK_FD" || exit 0
#######################################
echo "=== Backup started at $(date) ===" >> "$LOG"
mkdir -p \
  "$SNAPSHOTS" \
  "$BACKUP_ROOT/daily" \
  "$VIEW/hdd4tb/files" \
  "$VIEW/hdd4tb/immich" \
  "$VIEW/home_nas/immich_app" \
  "$VIEW/home_nas/configs"
#######################################
# Bind mounts (read-only)
#######################################
cleanup_mounts
mount --bind /mnt/hdd4tb/files "$VIEW/hdd4tb/files"
mount --bind /mnt/hdd4tb/immich "$VIEW/hdd4tb/immich"
mount --bind "$HOME_NAS/immich_app" "$VIEW/home_nas/immich_app"
mount -o remount,bind,ro "$VIEW/hdd4tb/files"
mount -o remount,bind,ro "$VIEW/hdd4tb/immich"
mount -o remount,bind,ro "$VIEW/home_nas/immich_app"
#######################################
# Disk space check
#######################################
FREE_GB=$(df --output=avail -BG "$BACKUP_ROOT" | tail -1 | tr -dc '0-9')
if [ "$FREE_GB" -lt "$MIN_FREE_GB" ]; then
  echo "Low disk space: ${FREE_GB}GB free" >> "$LOG"
  exit 1
fi
#######################################
# Collect configs
#######################################
rsync -a --delete --prune-empty-dirs \
  --include='*/' \
  --include='*.yml' \
  --include='*.env' \
  --include='snapshot_backup.sh' \
  --exclude='*' \
  "$HOME_NAS/" \
  "$VIEW/home_nas/configs/"
#######################################
# Snapshot creation
#######################################
DEST="$SNAPSHOTS/$TODAY"
PREV="$(readlink -f "$BACKUP_ROOT/daily/latest" 2>/dev/null || true)"
mkdir -p "$DEST"
RSYNC_OPTS=(
  -aAXH
  --numeric-ids
  --delete-after
  --partial
  --inplace
  --ignore-errors
  --exclude='*.sock'
  --exclude='*.pid'
  --exclude='*.lock'
)
if [ -n "$PREV" ] && [ -d "$PREV" ] && [ -f "$PREV/.snapshot_complete" ] && [ "$PREV" != "$SNAPSHOTS/$TODAY" ]; then
  echo "Incremental snapshot using link-dest=$PREV" >> "$LOG"
  rsync "${RSYNC_OPTS[@]}" --link-dest="$PREV" "$VIEW/" "$DEST/"
else
  echo "First snapshot or fallback full sync" >> "$LOG"
  rsync "${RSYNC_OPTS[@]}" "$VIEW/" "$DEST/"
fi
touch "$DEST/.snapshot_complete"
ln -sfn "$DEST" "$BACKUP_ROOT/daily/latest"
#######################################
# Pruning (date-based, safe)
#######################################
ls -1d "$SNAPSHOTS/"[0-9]* 2>/dev/null \
  | sort \
  | head -n "-$RETENTION_DAYS" \
  | xargs -r rm -rf
FREE_GB_AFTER=$(df --output=avail -BG "$BACKUP_ROOT" | tail -1 | tr -dc '0-9')
echo "Backup completed. Free space remaining: ${FREE_GB_AFTER}GB" >> "$LOG"
```

Make executable:

```
chmod +x /usr/local/bin/snapshot_backup.sh
```

**What This Script Does**
-------------------------

*   Creates a dated snapshot directory
*   Uses `--link-dest` for hard-linked incremental backups
*   Copies only changed data
*   Deletes files removed from the source
*   Updates `daily/latest` **only on success**
*   Prunes snapshots **by date**, not modification time

**Scheduling with cron**
------------------------

Edit crontab:

```
sudo crontab -e
```

Add:

```
0 2 * * * /usr/local/bin/snapshot_backup.sh
```

Backups run daily at 2 AM and saves the logs to `/var/log/snapshot_backup.log`

**What Is Backed Up (Explicit List)**
-------------------------------------

I back up:

*   `/mnt/hdd4tb/immich`
*   `/mnt/hdd4tb/files`
*   `/home/<user>/home_nas` :only the yaml files and the Immich PostgreSQL data
*   `snapshot_backup.sh` (the script itself 😅)

**Testing Backups**
-------------------

Once a month:

```
ls /mnt/hdd1tb/backups/snapshots
cd /mnt/hdd1tb/backups/daily/latest
```

Verify:

*   Files exist
*   Old versions exist
*   Hard links are present (`ls -li`)

> A backup you’ve never restored from is not a backup.

**Failure Scenarios and Recovery**
----------------------------------

Case 1: File Deleted Accidentally
---------------------------------

```
cp /mnt/hdd1tb/backups/snapshots/<date>/path/to/file /restore/location
```

That’s it 🙌

Case 2: Primary Disk Failure (4 TB)
-----------------------------------

Steps:

1.  Power off
2.  Replace disk
3.  Recreate mount point
4.  Mount new disk
5.  Restore data from backup:

```
rsync -a /mnt/hdd1tb/daily/latest/ /mnt/hdd4tb/
```

System is back.

Case 3: Backup Disk Failure (1 TB)
----------------------------------

*   No immediate data loss
*   Replace disk
*   Reinitialise backup directory
*   Next backup recreates everything

This is why backups are **not** mirrored live.

**My Perspective**
------------------

This is not the best approach, I feel like I have to myself work more upon it but this article may give you a brief idea about how to approach this problem.

**Other Technologies**
----------------------

You will often see the following names when reading about other NAS setups. They solve **storage and redundancy problems**, not backups by default. They are mentioned here only for awareness and brief introduction.

### RAID

RAID combines multiple disks into a single logical unit to improve **fault tolerance**, **performance**, or both, depending on the RAID level.

RAID only protects against **disk failure**.
It does **not** protect against accidental deletion, corruption, or ransomware. RAID is **not a backup.**

### MergerFS

MergerFS is a simple filesystem layer that merges multiple disks into a single directory tree.

It provides **no redundancy** by itself. Each disk remains independent. If one disk fails, only the data on that disk is lost.

Its main advantage is **simplicity and flexibility**, and it pairs well with file-level backups like `rsync`.

> I plan to use MergerFS when I upgrade my NAS to more disks.

### ZFS

ZFS is both a filesystem and a volume manager built around **data integrity**. It uses checksums, copy-on-write, and snapshots to detect and prevent silent data corruption. It also includes built-in RAID-like redundancy.

ZFS is very reliable but also **strict**, **memory-hungry**, and less forgiving of mistakes. It works best when correctness and long-term integrity matter more than simplicity.

### SnapRAID

SnapRAID is a parity-based system designed for **mostly static data** such as media collections.

Unlike RAID, parity is updated manually or on a schedule, not in real time. This makes it safer against accidental deletion but unsuitable for frequently changing data.

SnapRAID is often used alongside MergerFS.

**Final Thought**
-----------------

People may obsess over:

*   RAID levels
*   Filesystems
*   Benchmarks

But **recovery** is what matters.

A simple backup that restores correctly beats a clever system you don’t understand.

Prev
----

*   [**Article 1:** A Minimal Home NAS setup, that just works](https://medium.com/@samarth_04/article-1-a-minimal-home-nas-setup-that-just-works-a9e87d5fb745)
*   [**Article 2:** OS installation, disk preparation, SMART checks](https://medium.com/@samarth_04/article-2-os-installation-disk-preparation-smart-checks-6ceda3a128a5)
*   [**Article 2.5:** Base System Hardening , Utilities & Folder Structure](https://medium.com/@samarth_04/article-2-5-base-system-hardening-utilities-folder-structure-d22fc190a668)
*   [**Article 3:** Docker setup, volume layout, service deployment](https://medium.com/@samarth_04/article-3-docker-setup-volume-layout-service-deployment-d3ca8da57542)
