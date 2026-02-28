# Storage Configuration and Backup

## Overview

This guide covers setting up the **MergerFS storage pool** from two 1 TB hard drives and configuring automated **weekly cold backups** to a third drive.

---

## Part 1: Mounting the Drives

### Identify Disks

List all connected disks and their current state:

```bash
# Show block devices
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Detailed fdisk output (first 200 lines)
sudo /sbin/fdisk -l | sed -n '1,200p'

# Show devices by ID (more reliable)
ls -l /dev/disk/by-id/

# Show current mounts
df -hT
```

### Format and Label Drives

Replace `sdX`, `sdY`, `sdZ` with your actual device names (from above):

```bash
# Format three drives (WARNING: destroys data)
sudo /sbin/mkfs.ext4 -F -L DATA1 /dev/sdX
sudo /sbin/mkfs.ext4 -F -L DATA2 /dev/sdY
sudo /sbin/mkfs.ext4 -F -L BACKUP1 /dev/sdZ

# Verify formatting completed
sudo /sbin/blkid | grep -E 'DATA1|DATA2|BACKUP1'
```

### Create Mount Points

```bash
# Create directories
sudo mkdir -p /srv/dataA /srv/dataB /srv/backup

# Set permissions
sudo chmod 755 /srv/dataA /srv/dataB /srv/backup
```

### Add to /etc/fstab

Get the exact UUIDs from blkid output:

```bash
# Show UUIDs
sudo /sbin/blkid | grep -E 'DATA1|DATA2|BACKUP1'

# Copy the UUID strings (no quotes, exact format):
# Example:
# UUID="11111111-2222-3333-4444-555555555555" for DATA1
# UUID="66666666-7777-8888-9999-AAAAAAAAAAAA" for DATA2
# UUID="BBBBBBBB-CCCC-DDDD-EEEE-FFFFFFFFFFFF" for BACKUP1
```

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add these lines (replace UUIDs with your actual ones, NO quotes):

```
UUID=11111111-2222-3333-4444-555555555555 /srv/dataA ext4 defaults,noatime 0 2
UUID=66666666-7777-8888-9999-AAAAAAAAAAAA /srv/dataB ext4 defaults,noatime 0 2
UUID=BBBBBBBB-CCCC-DDDD-EEEE-FFFFFFFFFFFF /srv/backup ext4 defaults,noatime 0 2
```

Save (Ctrl+O, Enter, Ctrl+X).

### Mount All Drives

```bash
# Mount everything in fstab
sudo mount -a

# Verify mounting
df -h | grep -E '/srv/data|backup'

# Check mount points are accessible
ls -la /srv/dataA /srv/dataB
```

### Keep Backup Cold (Unmounted by Default)

```bash
# Unmount backup drive (cold storage)
sudo umount /srv/backup

# Create backup pool directory
sudo mkdir -p /srv/backup/pool
```

---

## Part 2: Create MergerFS Pool

MergerFS combines multiple drives into a single mount point while keeping files on their original drives (no data loss if one fails).

### Install MergerFS

```bash
sudo apt update
sudo apt install -y mergerfs
```

### Add Pool to /etc/fstab

Edit `/etc/fstab` and add this single line:

```bash
sudo nano /etc/fstab
```

Append this line (copy exactly, no line breaks):

```
/srv/dataA:/srv/dataB /srv/data fuse.mergerfs defaults,allow_other,use_ino,cache.files=off,category.create=ff 0 0
```

**Parameters explained:**
- `allow_other`: Non-root users can access
- `use_ino`: Use inode numbers (helps with rsync)
- `cache.files=off`: No caching (ensures consistency)
- `category.create=ff`: First-found; write entire files to first drive with space

### Mount Pool

```bash
# Mount MergerFS pool
sudo mount -a

# Verify pool created
df -h /srv/data

# Expected: ~1.4–1.5 TB total (two 1 TB drives minus overhead)

# List contents
ls -la /srv/data
```

---

## Part 3: Create Nextcloud Directory

```bash
# Create Nextcloud directory (if not already present from installation)
sudo mkdir -p /srv/data/nextcloud

# Set initial permissions (will be refined during Nextcloud setup)
sudo chown -R homeserver:homeserver /srv/data/nextcloud
sudo chmod 755 /srv/data/nextcloud
```

---

## Part 4: Automated Weekly Backup

### Create Backup Script

Copy this entire block and run in terminal:

```bash
sudo tee /usr/local/bin/run-backup.sh >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_DEV="/dev/disk/by-label/BACKUP1"
BACKUP_MNT="/srv/backup"
SRC="/srv/data"
DEST_DIR="$BACKUP_MNT/pool"

echo "[backup] start"

mkdir -p "$BACKUP_MNT"

# If auto-mounted under /media, unmount it
if mount | grep -q "on /media/.*BACKUP1 "; then
    OTHER_MNT="$(mount | awk '/\/media\/.*BACKUP1/ {print $3; exit}')"
    echo "[backup] unmounting auto-mount at $OTHER_MNT"
    umount "$OTHER_MNT" || true
fi

# Mount if not already
if mountpoint -q "$BACKUP_MNT"; then
    echo "[backup] already mounted"
else
    echo "[backup] mounting $BACKUP_DEV at $BACKUP_MNT"
    mount "$BACKUP_DEV" "$BACKUP_MNT"
fi

mkdir -p "$DEST_DIR"

echo "[backup] rsync..."
rsync -aHAX --delete --info=progress2 --numeric-ids "$SRC"/ "$DEST_DIR"/

sync
echo "[backup] unmounting"
umount "$BACKUP_MNT" || umount -l "$BACKUP_MNT" || true

echo "[backup] done"
EOF

# Make executable
sudo chmod +x /usr/local/bin/run-backup.sh

# Remove Windows line endings if present
sudo sed -i 's/\r$//' /usr/local/bin/run-backup.sh
```

### Create Systemd Service

```bash
sudo tee /etc/systemd/system/run-backup.service >/dev/null <<'EOF'
[Unit]
Description=Cold backup of pool to BACKUP1

[Service]
Type=oneshot
ExecStart=/usr/local/bin/run-backup.sh
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7
EOF
```

### Create Systemd Timer (Weekly, Saturday 02:00)

```bash
sudo tee /etc/systemd/system/run-backup.timer >/dev/null <<'EOF'
[Unit]
Description=Weekly cold backup (mount -> rsync -> unmount)

[Timer]
OnCalendar=Sat 02:00
RandomizedDelaySec=15min
Persistent=true
Unit=run-backup.service

[Install]
WantedBy=timers.target
EOF
```

### Enable and Start Timer

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable timer (auto-start on boot)
sudo systemctl enable run-backup.timer

# Start immediately (backup will run at next Saturday 02:00)
sudo systemctl enable --now run-backup.timer

# Verify
sudo systemctl list-timers run-backup.timer
```

### Test Backup Manually

```bash
# Run backup immediately (for testing)
sudo systemctl start run-backup.service

# Check status
sudo systemctl status run-backup.service

# View output/logs
sudo journalctl -u run-backup.service -f

# Expected output shows rsync progress and completion
```

---

## Part 5: Monitor Backup Status

### View Backup Schedule

```bash
# Show next scheduled backup time
sudo systemctl list-timers run-backup.timer
```

### Check Last Backup

```bash
# View logs from last run
sudo journalctl -u run-backup.service -n 50

# Expected: "[backup] done" at end

# Verify backup disk has data
sudo mount /dev/disk/by-label/BACKUP1 /srv/backup
du -sh /srv/backup/pool
sudo umount /srv/backup
```

### Monitor During Backup

If backup is running:

```bash
# Real-time rsync progress
sudo journalctl -u run-backup.service -f

# In another terminal, check I/O
iostat -x 2

# Check disk free space
watch 'df -h'
```

---

## Troubleshooting

### Backup Drive Not Found
```bash
sudo lsblk -o NAME,SIZE,LABEL
# Verify device is present and labeled BACKUP1
```

### rsync Permission Error
```bash
# Ensure backup directory is owned by root
sudo chown -R root:root /srv/backup
sudo chmod 755 /srv/backup /srv/backup/pool
```

### Backup Takes Too Long
```bash
# First backup might take 1–6 hours (depends on data)
# Subsequent backups faster (only changed files)
# Consider scheduling at off-peak hours (currently Sat 02:00)
```

### "Device or resource busy" during unmount
```bash
# Script already handles this with forced unmount (-l)
# If manual umount needed:
sudo fuser -m /srv/backup  # Show processes using mount
sudo umount -l /srv/backup # Lazy unmount
```

---

## Disk Space Accounting

```bash
# Check current usage
df -h /srv/data
du -sh /srv/data/*

# Example output:
# /srv/data    1.8T  500G  1.3T  28%  /srv/data

# Check individual drives in pool
du -sh /srv/dataA /srv/dataB

# Backup disk used space
du -sh /srv/backup/pool
```

---

## Related Guides

- [Hardware Specs](../Hardware.md) – Drive details
- [Installing Nextcloud](./Nextcloud.md) – File permissions
- [Global Access](./Cloudflare-Tunnel.md) – Backup data off-site
