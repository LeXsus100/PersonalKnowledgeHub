---
title: "Backups"
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 330
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
Backups are a fundamental reliability measure. They are not an optimization and not optional.  
Without backups, a single error or hardware failure can result in irreversible data loss.

At some point, one of the following will occur:

* Accidental deletion of files or directories
* Overwriting a configuration file
* Misconfiguration of a container or service
* Filesystem corruption
* Storage device failure

These events are normal operational risks, not exceptional situations.

---

## What Is a Backup?

A backup is a **recoverable historical copy of data stored on a separate physical device**.

Two properties are essential:

1. The copy must be stored on different physical media.
2. Historical versions must be preserved.

### What Is *Not* a Backup

| Mechanism                                                 | Backup?    |
| --------------------------------------------------------- | ---------- |
| Copy on the same disk                                     | ❌ No      |
| File synchronization (Nextcloud, Syncthing, rsync mirror) | ❌ No      |
| RAID                                                      | ❌ No      |
| External disk with versioned snapshots                    | ✅ Yes     |

* RAID improves availability but does not protect against logical errors.
* Synchronization mirrors the current state, including deletions.
* A backup must allow recovery of previous states.

A system that stores only the latest version is insufficient.  
If a file is deleted or corrupted and the backup updates immediately, the damaged state replaces the valid one.  
Version history is therefore mandatory.

---

### Why This Is Critical on a Homeserver

A homeserver often centralizes:

* Personal documents
* Photos
* PKM vaults
* Docker volumes
* Configuration files
* SSH keys
* Databases

This architecture introduces a **single point of failure**.

Empirical data shows that most data loss incidents are caused by user error or software misbehaviour rather than hardware failure.

Typical scenarios:

* Executing `rm -rf` in the wrong directory
* Incorrect bind mounts overwriting host data
* A malfunctioning container modifying persistent volumes
* A sync client propagating deletions
* Filesystem corruption discovered weeks later

In many cases, the issue is detected after some delay.  
For this reason, versioned backups are required.

---

### The 3-2-1 Rule

Maintain:

{{< callout context="tip" >}}
* 3 total copies of the data
* Stored on 2 different types of media
* With at least 1 copy offline or offsite
{{< /callout >}}

Example:

* Primary data on the homeserver
* Backup on an external hard drive
* Additional copy on a second external drive or remote storage

This reduces dependency on a single device or location.

---

## BorgBackup

BorgBackup is a deduplicating, encrypted, versioned backup tool.

🌐 Website: [https://www.borgbackup.org/](https://www.borgbackup.org/)

Borg operates by:

* Splitting files into content-defined chunks
* Deduplicating unchanged chunks
* Compressing data
* Encrypting repository contents
* Creating snapshot-based archives

Each backup execution creates a **snapshot**.  
Files can be restored exactly as they existed at a specific time, even if they no longer exist in the current filesystem.

This snapshot model distinguishes Borg from simple rsync mirrors, which only reflect the most recent state.

---

### Installation

```bash
sudo apt install -y borgbackup
borg --version
```

---

### Repository Setup

#### 1. Create Repository Directory

Example (external disk mounted as `E:` in WSL):

```bash
mkdir -p /mnt/e/borg-repo
```

#### 2. Initialize Repository

```bash
borg init --encryption=repokey-blake2 /mnt/e/borg-repo
```

`repokey-blake2` is recommended on modern CPUs for improved performance.

You will be prompted for a passphrase.

* Generate it using a password manager.
* Store it securely and do not rely on memory.

If the passphrase is lost, the repository is unrecoverable.

#### 3. Export Repository Key

```bash
borg key export /mnt/e/borg-repo /mnt/e/borg-repo.key
```

Store this file securely (for example, inside an encrypted archive or password manager attachment), then remove it from the backup disk:

```bash
rm /mnt/e/borg-repo.key
```

This protects against repository metadata corruption.

#### 4. Retention

Based on your policy, you can decide whether to keep a fixed number of backups or a based on data. The most common example is:

```bash
borg prune -v --list /mnt/e/borg-repo --keep-daily=7 --keep-weekly=4 --keep-monthly=6
```

which means Borg keep the last 7 days, 4 weekly backups and 6 monthly ones.

Because Borg use a log-structured repository, the previous command does not immediately free up space. Thus you have to write the following command as well:

```bash
borg compact /mnt/e/borg-repo
```

It's important to run these two retention commands in order to check whether there are outdated backups and free up space for the incoming backup.

---

## Creating a Backup & Check

```bash
borg create --stats --progress /mnt/e/borg-repo::'{hostname}_{now:%Y-%m-%d}' /mnt/c/PATH
```

Each execution creates a new snapshot. Unchanged data is not duplicated.

#### List Snapshots

```bash
borg list /mnt/e/borg-repo
```

Each entry corresponds to a restorable point in time.  
If you want a more detailed info:

```bash
borg info /mnt/e/borg-repo::BACKUPNAME
```

Every now and then (because it could be slow) you should also check for integrity:

```bash
borg check /mnt/e/borg-repo
```

and its in-depth variant:

```bash
borg check --verify-data /mnt/e/borg-repo
```

To check the size of the backups data, the most practical command is the following:

```bash
borg list --format '{archive:<36} {time} {size} {nfiles} files' /mnt/e/borg-repo
```

---

## Restoring Data

Create a restore directory:

```bash
mkdir -p /mnt/c/restore
cd /mnt/c/restore
```

Extract a full snapshot:

```bash
borg extract /mnt/e/borg-repo::BACKUPNAME --strip-components 2
```

or a specific folder adding the path after the backup's name (in this case `name/university`):

```bash
borg list /mnt/e/borg-repo::BACKUPNAME name/university
borg extract /mnt/e/borg-repo::BACKUPNAME name/university --strip-components 2
```

The restored files are independent from the original source.

---

## Automation

Manual backups are unreliable over time. Automating execution ensures consistency.

Create a script:

```bash
cat > ~/backup_versioned.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

REPO="/mnt/e/borg-repo"
SRC="/mnt/c/Luigi"

ARCHIVE="BACKUPNAME-$(date +%Y-%m-%d_%H%M)"

KEEP_DAILY=7
KEEP_WEEKLY=4
KEEP_MONTHLY=12
KEEP_YEARLY=2

if [ ! -d "$REPO" ]; then
  echo "ERROR: Borg repository not found at: $REPO"
  exit 1
fi

if [ ! -d "$SRC" ]; then
  echo "ERROR: Source folder not found at: $SRC"
  exit 1
fi

borg create --stats --progress \
  --exclude '**/$RECYCLE.BIN/**' \
  --exclude '**/System Volume Information/**' \
  --exclude '**/Thumbs.db' \
  --exclude '**/Desktop.ini' \
  "$REPO::$ARCHIVE" \
  "$SRC"

borg prune -v --list "$REPO" \
  --keep-daily="$KEEP_DAILY" \
  --keep-weekly="$KEEP_WEEKLY" \
  --keep-monthly="$KEEP_MONTHLY" \
  --keep-yearly="$KEEP_YEARLY"

borg check "$REPO"
EOF
```

Make executable:

```bash
chmod +x backup_versioned.sh
```

If Windows line endings cause issues:

```bash
dos2unix backup_versioned.sh
chmod +x backup_versioned.sh
```

Run:

```bash
./backup_versioned.sh
```

---

## Deleting the Repository

```bash
rm -rf /mnt/e/borg-repo
```

This permanently removes all backups.

---

## Verification

A backup strategy is not complete until restoration has been tested.

After setup:

1. Restore a random directory
2. Open several files
3. Confirm integrity

A backup should be considered valid only after a successful restoration test.