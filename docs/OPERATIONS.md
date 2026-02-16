# Operations Guide - ujust Command Reference

## Introduction

The `ujust` command provides a user-friendly interface to common Keystone operations. It wraps the `just` task runner with Keystone-specific recipes organized by function.

**Philosophy:** Operators should use `ujust` for day-to-day tasks, not raw Ansible playbooks.

## Quick Start

```bash
# List all available commands
ujust

# Show Keystone version
ujust version

# Check system status
ujust status
```

## Command Reference

### System Management (`system-*`)

#### `ujust system-update`
Update all system packages.

```bash
ujust system-update
```

**What it does:**
- Debian: `apt update && apt upgrade -y`
- Fedora: `dnf upgrade -y`

**When to use:** Weekly or monthly maintenance

**Idempotent:** Yes

---

#### `ujust system-clean`
Clean package manager cache and remove unused packages.

```bash
ujust system-clean
```

**What it does:**
- Debian: `apt autoremove -y && apt clean`
- Fedora: `dnf autoremove -y && dnf clean all`

**When to use:** After system-update, or when disk space is low

**Idempotent:** Yes

---

#### `ujust system-info`
Display system information (hostname, kernel, uptime, memory, disk usage).

```bash
ujust system-info
```

**Example output:**
```
=== System Information ===
Hostname: keystone
Kernel: 6.6.31+rpt-rpi-v8
Uptime: up 5 days, 3 hours
Load: 0.15, 0.10, 0.08

=== Memory Usage ===
              total        used        free      shared  buff/cache   available
Mem:           7.6Gi       1.2Gi       5.8Gi        12Mi       612Mi       6.2Gi
Swap:          2.0Gi          0B       2.0Gi

=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   29G  4.2G   24G  15% /
/dev/nvme0n1    932G   45G  840G   6% /mnt/ssd
```

**When to use:** Troubleshooting, capacity planning

**Idempotent:** Yes

---

#### `ujust system-reboot`
Reboot the system (with confirmation prompt).

```bash
ujust system-reboot
```

**What it does:**
1. Prompts for confirmation
2. Runs `sudo reboot` if confirmed

**When to use:** After kernel updates, major configuration changes

**Safety:** Requires typing "yes" to proceed

---

### Storage Management (`storage-*`)

#### `ujust storage-raid-status`
Display RAID array status and health.

```bash
ujust storage-raid-status
```

**Example output:**
```
=== RAID Status ===
/dev/md0:
        Version : 1.2
  Creation Time : Sun Feb 16 10:30:00 2026
     Raid Level : raid1
     Array Size : 1953383488 (1862.89 GiB 2000.26 GB)
  Used Dev Size : 1953383488 (1862.89 GiB 2000.26 GB)
   Raid Devices : 2
  Total Devices : 2
    Persistence : Superblock is persistent

    Update Time : Sun Feb 16 14:00:00 2026
          State : clean
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
  Spare Devices : 0

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda
       1       8       16        1      active sync   /dev/sdb
```

**When to use:**
- Daily health checks
- After disk replacement
- Investigating storage issues

**Idempotent:** Yes

---

#### `ujust storage-health`
Check disk health using SMART data.

```bash
ujust storage-health
```

**Example output:**
```
=== Disk Health (SMART) ===
--- /dev/nvme0n1 ---
SMART overall-health self-assessment test result: PASSED

--- /dev/sda ---
SMART overall-health self-assessment test result: PASSED

--- /dev/sdb ---
SMART overall-health self-assessment test result: PASSED
```

**When to use:**
- Weekly health checks
- Before/after disk operations
- Investigating performance issues

**Idempotent:** Yes

**Note:** Requires smartmontools installed (handled by storage-primitives role)

---

#### `ujust storage-usage`
Show filesystem usage for all Keystone mounts.

```bash
ujust storage-usage
```

**Example output:**
```
=== Filesystem Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1    932G   45G  840G   6% /mnt/ssd
/dev/md0        1.9T  120G  1.7T   7% /mnt/backup
```

**When to use:**
- Capacity planning
- Investigating "disk full" errors
- Routine monitoring

**Idempotent:** Yes

---

#### `ujust storage-raid-scrub`
Start a RAID array scrub (data integrity check).

```bash
ujust storage-raid-scrub
```

**What it does:**
```bash
echo "check" > /sys/block/md0/md/sync_action
```

**When to use:**
- Monthly maintenance (recommended)
- After unclean shutdown
- Before critical backups

**Duration:** Hours to days (depends on array size)

**Monitor progress:**
```bash
watch cat /proc/mdstat
```

**Safety:** Read-only operation; array remains online

---

### Container Management (`container-*`)

#### `ujust container-list`
List all running Podman containers.

```bash
ujust container-list
```

**Example output:**
```
=== Running Containers ===
NAMES     STATUS                  IMAGE
cockpit   Up 3 days (healthy)     quay.io/cockpit/ws:324
```

**When to use:**
- Verifying service status
- Troubleshooting container issues
- Daily health checks

**Idempotent:** Yes

---

#### `ujust container-stats`
Show real-time resource usage for containers.

```bash
ujust container-stats
```

**Example output:**
```
ID            NAME      CPU %     MEM USAGE / LIMIT    MEM %     NET I/O        BLOCK I/O    PIDS
a1b2c3d4e5f6  cockpit   0.12%     45MiB / 7.6GiB       0.58%     12kB / 8kB     0B / 0B      3
```

**When to use:**
- Investigating performance issues
- Capacity planning
- Identifying resource-heavy containers

**Idempotent:** Yes

---

#### `ujust container-cockpit-restart`
Restart the Cockpit container.

```bash
ujust container-cockpit-restart
```

**What it does:**
```bash
sudo systemctl restart cockpit
```

**When to use:**
- After Cockpit configuration changes
- Troubleshooting Cockpit access issues
- Applying container image updates

**Downtime:** ~5-10 seconds

**Idempotent:** Yes

---

#### `ujust container-cockpit-logs`
Stream Cockpit container logs (follows in real-time).

```bash
ujust container-cockpit-logs
```

**What it does:**
```bash
journalctl -u cockpit -n 50 -f
```

**When to use:**
- Debugging Cockpit connection issues
- Investigating authentication failures
- Monitoring access patterns

**Exit:** Ctrl+C

---

#### `ujust container-prune`
Remove unused containers, images, and volumes.

```bash
ujust container-prune
```

**What it does:**
```bash
podman system prune -af --volumes
```

**When to use:**
- Reclaiming disk space
- Cleaning up after image updates
- Monthly maintenance

**Warning:** Removes **all** unused resources

**Safety:** Active containers are not affected

---

### Tailscale Network (`tailscale-*`)

#### `ujust tailscale-status`
Show Tailscale connection status and peer list.

```bash
ujust tailscale-status
```

**Example output:**
```
100.64.0.5   keystone             user@example.com  linux   active; relay "nyc", tx 1234567 rx 9876543
100.64.0.10  laptop               user@example.com  macOS   active; direct, tx 456789 rx 123456
```

**When to use:**
- Verifying tailnet connectivity
- Checking peer connections
- Troubleshooting network issues

**Idempotent:** Yes

---

#### `ujust tailscale-ip`
Show Keystone's Tailscale IPv4 address.

```bash
ujust tailscale-ip
```

**Example output:**
```
100.64.0.5
```

**When to use:**
- Getting IP for SSH/Cockpit access
- Scripting (e.g., `cockpit_ip=$(ujust tailscale-ip)`)
- Network configuration

**Idempotent:** Yes

---

#### `ujust tailscale-reconnect`
Restart Tailscale connection.

```bash
ujust tailscale-reconnect
```

**What it does:**
1. `tailscale down`
2. Wait 2 seconds
3. `tailscale up`

**When to use:**
- After network configuration changes
- Troubleshooting connection issues
- Forcing relay/direct connection switch

**Downtime:** ~2-5 seconds

**Idempotent:** Yes

---

#### `ujust tailscale-logs`
Stream Tailscale daemon logs.

```bash
ujust tailscale-logs
```

**What it does:**
```bash
journalctl -u tailscaled -n 50 -f
```

**When to use:**
- Debugging connection failures
- Investigating authentication issues
- Monitoring network events

**Exit:** Ctrl+C

---

### Custom Tasks (`custom-*`)

#### `ujust custom-example`
Example custom task (placeholder).

```bash
ujust custom-example
```

**Purpose:** Template for user-defined tasks

**Customization:**
Edit `/usr/share/keystone/just/60-custom.just`:

```just
# Custom backup task
custom-backup:
    @echo "Starting backup..."
    @rsync -avz /mnt/ssd/ /mnt/backup/ssd-backup/
    @echo "Backup complete!"
```

Then run:
```bash
ujust custom-backup
```

**Best practices:**
- Prefix with `custom-`
- Document behavior in comments
- Use `@echo` for user feedback
- Avoid destructive operations without confirmation

---

## Advanced Usage

### Chaining Commands

```bash
# Update system, then clean cache
ujust system-update && ujust system-clean

# Check all storage health metrics
ujust storage-raid-status && ujust storage-health && ujust storage-usage
```

### Integration with Scripts

```bash
#!/bin/bash
# Daily health check script

echo "=== Daily Keystone Health Check ==="
ujust storage-health
ujust storage-usage
ujust container-list
ujust tailscale-status
```

### Automation via Cron

```bash
# Edit root crontab
sudo crontab -e

# Add weekly RAID scrub (Sundays at 2 AM)
0 2 * * 0 /usr/local/bin/ujust storage-raid-scrub

# Add daily health check (6 AM)
0 6 * * * /usr/local/bin/ujust storage-health | mail -s "Keystone Health" admin@example.com
```

## Extending ujust

### Adding New Tasks

1. **Choose appropriate module:**
   - System tasks → `00-system.just`
   - Storage tasks → `10-storage.just`
   - Container tasks → `20-containers.just`
   - Network tasks → `30-tailscale.just`
   - Custom tasks → `60-custom.just`

2. **Edit module file:**
   ```bash
   sudo vim /usr/share/keystone/just/10-storage.just
   ```

3. **Add recipe:**
   ```just
   # Verify filesystem integrity
   storage-fsck:
       @echo "Checking SSD filesystem..."
       @sudo xfs_repair -n /dev/nvme0n1
   ```

4. **Test:**
   ```bash
   ujust storage-fsck
   ```

5. **Commit to Git:**
   ```bash
   # Copy to repo templates
   cp /usr/share/keystone/just/10-storage.just \
      roles/ujust/templates/10-storage.just.j2
   
   git add roles/ujust/templates/10-storage.just.j2
   git commit -m "feat: Add storage-fsck command"
   ```

### Parameterized Tasks

```just
# Backup specific directory
custom-backup path:
    @echo "Backing up {{ path }}..."
    @rsync -avz {{ path }} /mnt/backup/
    @echo "Backup complete!"
```

Usage:
```bash
ujust custom-backup /mnt/ssd/important-data
```

## Troubleshooting

### `ujust: command not found`

**Cause:** ujust role not applied

**Solution:**
```bash
ansible-playbook site.yml --tags ujust
```

### `just: command not found`

**Cause:** `just` binary not installed

**Solution (Debian):**
```bash
ansible-playbook site.yml --tags ujust --tags packages
```

**Solution (Fedora):**
```bash
sudo dnf install just
```

### `Error: Justfile not found`

**Cause:** Justfile not deployed

**Solution:**
```bash
ansible-playbook site.yml --tags ujust
```

### Recipe Fails with Permission Denied

**Cause:** Task requires root but not using sudo

**Solution:** Add `sudo` to task:
```just
storage-raid-status:
    @sudo mdadm --detail /dev/md0
```

## Best Practices

1. **Use ujust for operator tasks** - Don't expose raw Ansible to operators
2. **Prefix by category** - `system-`, `storage-`, etc.
3. **Always include help text** - Use `@echo` to explain what's happening
4. **Confirm destructive actions** - Use `read -p "Continue? (yes/no)"`
5. **Keep tasks idempotent** - Safe to run multiple times
6. **Log important operations** - Output to `/var/log/keystone/`
7. **Test before committing** - Verify on both Debian and Fedora

## Comparison: ujust vs Ansible

| Task                     | ujust                     | Ansible                                    |
|--------------------------|---------------------------|--------------------------------------------|
| Update system            | `ujust system-update`     | `ansible-playbook site.yml --tags base`    |
| Check RAID               | `ujust storage-raid-status`| Manual `mdadm` command                    |
| Restart Cockpit          | `ujust container-cockpit-restart` | `ansible-playbook site.yml --tags cockpit` |
| **Operator preference**  | ✅ Fast, memorable        | ❌ Verbose, requires Ansible knowledge     |
| **GitOps compliance**    | ⚠️ Stateless (no tracking)| ✅ Declarative, tracked                    |

**When to use which:**
- **ujust:** Day-to-day operations, troubleshooting, maintenance
- **Ansible:** Infrastructure changes, deployments, drift correction

## References

- [just Manual](https://github.com/casey/just#readme)
- [Blueberry ujust Implementation](../blueberry/system_files/usr/share/ublue-os/just/)
- [Keystone Architecture](./README.md)
