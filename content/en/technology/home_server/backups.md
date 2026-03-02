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
This is one of those topics everyone postpones until the day everything explodes.

{{< callout context="note" >}}
Backups are not an optimization, not a “nice to have”, and not a "future-me" problem.  
They are the only thing standing between *a small mistake* and *permanent data loss*.
{{< /callout >}}

You will eventually:

* delete the wrong folder
* overwrite a config
* break a container
* corrupt a filesystem
* or a disk will just die randomly

It’s not pessimism, it’s statistics.

---

## So… what is a backup?

A backup is **a recoverable historical copy of your data stored on another physical device**.

Important distinction:

| Thing                                     | Is it a backup? |
| ----------------------------------------- | --------------- |
| Copy on same disk                         | ❌ no           |
| Sync (Nextcloud, Syncthing, rsync mirror) | ❌ no           |
| RAID                                      | ❌ no           |
| External disk with snapshots              | ✅ yes          |

RAID protects uptime.
Sync protects convenience.
Backups protect your life.

Also: a backup that only contains the *latest version* is dangerous.  
If you accidentally delete a folder and the backup updates → congratulations, you just backed up the mistake.

You want **history**.

---

### Why this matters especially on a homeserver

A **homeserver centralizes everything**:

* documents
* photos
* PKM vault
* docker volumes
* configs
* SSH keys
* databases

You basically created a **single point of failure** in your house.

And most real data loss is not hardware failure, it's you (or software acting on your behalf).

Typical real scenarios:

* `rm -rf` in the wrong directory
* bind mount pointing to the wrong path
* a broken docker container overwriting data
* a sync client propagating deletions
* filesystem corruption discovered weeks later

Notice the pattern: you often realize the problem **late**.  
That’s why versioned backups matter.

---

### The 3-2-1 Rule

Keep:

* 3 copies of data
* on 2 different media
* 1 offline/offsite

Example:

* main data → homeserver
* backup → external HDD
* safety → another drive / remote storage

Without this, you’re relying on luck.

---

## Why BorgBackup

[https://www.borgbackup.org/](https://www.borgbackup.org/)

Borg is basically *git for filesystems*.

Instead of copying files every time, it:

* splits them into chunks
* detects what changed
* only saves the difference
* compresses it
* encrypts it
* keeps snapshots

Result:
You can restore **a file from 4 months ago even if it no longer exists today**.

That single feature is the reason Borg is superior to rsync mirrors.

---

### Installation

```bash
sudo apt install -y borgbackup
borg --version
```

---

### Repository setup

##### 1) Create the repository folder (your backup HDD)

Assuming the external disk is `E:` in WSL:

```bash
mkdir -p /mnt/e/borg-repo
```

##### 2) Initialize the repository

```bash
borg init --encryption=repokey /mnt/e/borg-repo
```

You can also use `repokey-blake2` that is faster on modern CPUs.

It will ask for a passphrase.

{{< callout context="caution" >}}
**Do not invent one manually.**  
Generate it in your password manager and save it there.
{{< /callout >}}

{{< callout context="danger" >}}
If you lose this passphrase, the data is gone **forever**. No recovery.
{{< /callout >}}

##### 3) Export the key

```bash
borg key export /mnt/e/borg-repo /mnt/e/borg-repo.key
```

Save the key file inside your password manager / secure archive.

Then delete it from disk:

```bash
rm /mnt/e/borg-repo.key
```

This protects you if the repository metadata gets damaged.

---

## Create a backup

```bash
borg create --stats --progress /mnt/e/borg-repo::BACKUPNAME /mnt/c/PATH
```

You can run this many times.
Borg will not duplicate data — it will only store changes and create a snapshot.

#### See available snapshots

```bash
borg list /mnt/e/borg-repo
```

Think of these as restore points in time.

---

## Restore files

Create a restore folder:

```bash
mkdir -p /mnt/c/restore
cd /mnt/c/restore
```

Restore a snapshot:

```bash
borg extract /mnt/e/borg-repo::BACKUPNAME --strip-components 2
```

You now have a full copy independent from the original files.

---

### Automating the process

Manual backups always start well and then stop happening.

Create the script:

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

echo "== Borg versioned backup =="
echo "Repo:    $REPO"
echo "Source:  $SRC"
echo "Archive: $ARCHIVE"
echo

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

echo
echo "== Pruning old archives (retention policy) =="
borg prune -v --list "$REPO" \
  --keep-daily="$KEEP_DAILY" \
  --keep-weekly="$KEEP_WEEKLY" \
  --keep-monthly="$KEEP_MONTHLY" \
  --keep-yearly="$KEEP_YEARLY"

echo
echo "== Repository check (lightweight) =="
borg check "$REPO"

echo
echo "== Done =="
EOF
```

#### Run it

```bash
cd /mnt/c/PATH OF THE .sh FILE
chmod +x backup_versioned.sh
```

If Windows line ending problems:

```bash
dos2unix backup_versioned.sh
chmod +x backup_versioned.sh
```

thus, if necessary, install it: `sudo apt install dos2unix`.

Execute:

```bash
./backup_versioned.sh
```

---

## Delete the entire backup

If you really need it.

```bash
rm -rf /mnt/e/borg-repo
ls /mnt/e
```

---

## What you now have

You now have:

* version history
* encrypted backups
* space-efficient storage
* corruption detection
* the ability to go back in time

Retention policy:

* 7 daily
* 4 weekly
* 12 monthly
* 2 yearly

A backup only exists after you **restore something and open it successfully**.

{{< callout context="caution" title="Caution" icon="outline/alert-triangle" >}}
After finishing setup, restore a random folder and check the files.  
Until you do that, you are trusting, not backing up.
{{< /callout >}}