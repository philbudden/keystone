# Storage Setup Guide

This guide covers **manual preparation** of storage devices before running the Keystone Ansible playbook.

> ⚠️ **WARNING**: These operations are destructive and will erase all data on the target devices. Verify device paths carefully before proceeding.

## Overview

Keystone uses a three-tier storage architecture:

| Device Type | Purpose | Managed By | Requires Pre-setup |
|-------------|---------|------------|-------------------|
| eMMC/SD Card | OS only | Manual OS installation | ✅ OS installed |
| M.2 NVMe SSD | Containers + state | Ansible (filesystem/mount) | ✅ Partitioned |
| 2× SATA HDD | Backup storage (RAID1) | Ansible (RAID/filesystem) | ❌ Raw devices |

## Prerequisites

Before starting, you need:
- Root/sudo access to the target system
- Raspberry Pi OS Lite (or compatible) already installed on eMMC/SD
- Physical hardware installed (SSD, HDDs)

## Step 1: Identify Your Devices

List all block devices:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Example output on Raspberry Pi CM5:
```
NAME         SIZE TYPE MOUNTPOINT MODEL
mmcblk0     29.1G disk            
├─mmcblk0p1  512M part /boot/firmware
└─mmcblk0p2 28.6G part /
nvme0n1      1.8T disk            Samsung SSD 990 PRO 2TB
sda          3.6T disk            WDC WD40EZRZ-00G
sdb          3.6T disk            WDC WD40EZRZ-00G
```

Identify your devices:
- **OS device**: `mmcblk0` (already partitioned during OS install)
- **SSD device**: `nvme0n1` (needs partitioning)
- **HDD devices**: `sda`, `sdb` (leave as raw devices for RAID)

## Step 2: Partition the SSD

The SSD must be partitioned **before** running the Ansible playbook.

### Option A: Single Partition (Recommended)

Create one large partition for all container storage:

```bash
# WARNING: This will erase all data on /dev/nvme0n1
sudo parted /dev/nvme0n1 --script -- \
  mklabel gpt \
  mkpart primary 0% 100%

# Verify the partition was created
lsblk /dev/nvme0n1
```

Expected result:
```
NAME        SIZE TYPE MOUNTPOINT
nvme0n1     1.8T disk
└─nvme0n1p1 1.8T part
```

### Option B: Multiple Partitions (Advanced)

If you want to separate container storage from other data:

```bash
# Example: 500GB for containers, rest for other use
sudo parted /dev/nvme0n1 --script -- \
  mklabel gpt \
  mkpart containers 0% 500GB \
  mkpart data 500GB 100%
```

For Keystone, use the first partition: `ssd_device: /dev/nvme0n1p1`

### Verify Partition Table

```bash
sudo parted /dev/nvme0n1 print
```

## Step 3: Prepare HDD Devices (RAID)

**Do NOT partition the HDD devices.** Ansible will use them as raw devices for mdadm RAID.

Verify they are clean (no existing filesystems):

```bash
sudo blkid /dev/sda /dev/sdb
```

If output shows existing filesystems, you can wipe them:

```bash
# WARNING: This will erase all data
sudo wipefs -a /dev/sda
sudo wipefs -a /dev/sdb
```

## Step 4: Configure Inventory

Edit `inventory/hosts.yml` and uncomment the `keystone_storage` block:

```yaml
keystone_storage:
  # Informational only - not managed by playbook
  os_device: /dev/mmcblk0
  
  # SSD storage - use the partition, not the raw device
  ssd_device: /dev/nvme0n1p1      # ← Note: p1 partition
  ssd_mount: /mnt/ssd
  
  # RAID storage - use raw devices, not partitions
  backup_devices:
    - /dev/sda                      # ← Note: raw device
    - /dev/sdb                      # ← Note: raw device
  backup_mount: /mnt/backup
```

### Important Notes

- **SSD device**: Use the **partition** (e.g., `/dev/nvme0n1p1`), not the raw device
- **HDD devices**: Use **raw devices** (e.g., `/dev/sda`), not partitions
- **Device paths**: May differ on your system - always verify with `lsblk`

## Step 5: Verify Configuration

Before running the playbook, verify:

### Check SSD partition exists
```bash
lsblk /dev/nvme0n1p1
# Should show the partition
```

### Check HDDs are raw (no partitions)
```bash
lsblk /dev/sda /dev/sdb
# Should show only the disk, no partitions
```

### Check no filesystems on HDDs
```bash
sudo blkid /dev/sda /dev/sdb
# Should return no output (or show old metadata to be wiped)
```

## Step 6: Run the Playbook

Once devices are prepared, run the Ansible playbook:

```bash
cd ~/keystone
source .venv/bin/activate
ansible-playbook site.yml --tags storage --check --diff  # Dry-run first
ansible-playbook site.yml                                # Full deployment
```

The playbook will:
1. ✅ Create XFS filesystem on SSD partition (`/dev/nvme0n1p1`)
2. ✅ Create systemd mount unit for `/mnt/ssd`
3. ✅ Create RAID1 array from raw HDDs (`/dev/md0`)
4. ✅ Create ext4 filesystem on RAID array
5. ✅ Create systemd mount unit for `/mnt/backup`
6. ✅ Configure SMART monitoring for all devices
7. ✅ Configure mdadm monitoring for RAID

## Troubleshooting

### "Device is already in use"

If you get errors about devices being in use:

```bash
# Check what's using the device
sudo lsof /dev/nvme0n1
sudo fuser -m /dev/nvme0n1

# If mounted, unmount it
sudo umount /dev/nvme0n1p1
```

### "Partition table exists"

If `parted` complains about existing data:

```bash
# Wipe all filesystem signatures
sudo wipefs -a /dev/nvme0n1

# Then run the partition command again
```

### "RAID device already exists"

If mdadm finds existing RAID metadata:

```bash
# Stop any existing RAID arrays
sudo mdadm --stop /dev/md0

# Zero out superblocks on devices
sudo mdadm --zero-superblock /dev/sda /dev/sdb

# Then re-run the playbook
```

### Verify Device Paths

Device paths can change between boots or systems. Always verify:

```bash
# List devices by ID (more stable)
ls -l /dev/disk/by-id/

# Use by-id paths in inventory if needed
ssd_device: /dev/disk/by-id/nvme-Samsung_SSD_990_PRO_2TB_S7XXXXXXXXXX-part1
```

## Alternative: Skip Custom Storage

If you don't want to configure custom storage, simply **leave the `keystone_storage` section commented out** in `inventory/hosts.yml`.

The system will use default paths:
- Containers: `/var/lib/containers` (on OS device)
- Cockpit: `/var/lib/cockpit` (on OS device)
- No RAID backup storage

This is perfectly valid for testing or single-disk setups.

## Post-Deployment Verification

After the playbook completes:

```bash
# Check mounts
findmnt /mnt/ssd /mnt/backup

# Check RAID status
sudo mdadm --detail /dev/md0

# Check filesystems
df -h /mnt/ssd /mnt/backup

# Use ujust commands
ujust storage-health
ujust storage-usage
ujust storage-raid-status
```

## Migration from Existing Data

If you have existing data to preserve:

### Before partitioning:
1. **Backup your data** to another location
2. Partition and configure storage as above
3. Run the Ansible playbook to set up mounts
4. Restore your data to the new mount points

### For containers:
```bash
# After playbook completes and /mnt/ssd is mounted
sudo rsync -av /var/lib/containers/ /mnt/ssd/containers/
sudo systemctl restart podman.socket
```

## Architecture Reference

See [AGENTS.md](../AGENTS.md) §4 for the full storage architecture contract and design rationale.

See [README.md](../README.md) for the storage layout table and feature overview.
