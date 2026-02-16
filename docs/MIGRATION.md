# Migration Guide: Debian Trixie → Fedora IoT (Blueberry)

## Overview

This guide documents the migration path from Raspberry Pi OS Lite (Debian Trixie) to Blueberry (custom Fedora IoT image). The Ansible project is designed to minimize migration friction—most changes are variable swaps, not playbook rewrites.

## Migration Readiness Assessment

### ✅ Zero-Change Components

These components transfer to Fedora IoT without modification:

- **Systemd units** - 100% portable
- **ujust justfiles** - Pure `just` syntax, OS-agnostic
- **Storage configuration** - mdadm, XFS, ext4 work identically
- **Podman setup** - Same container runtime
- **Tailscale client** - Package name differs, behavior identical
- **Mount units** - systemd standard, portable

### ⚠️ Variable-Swap Components

These require changing `vars/<OS>.yml` files only:

| Component         | Debian Package      | Fedora Package    | Status       |
|-------------------|---------------------|-------------------|--------------|
| Just binary       | Manual install      | `dnf install just`| Automated    |
| Podman            | `podman`            | `podman`          | Same         |
| mdadm             | `mdadm`             | `mdadm`           | Same         |
| Tailscale         | Debian repo         | Fedora repo       | Different URL|
| smartmontools     | `smartmontools`     | `smartmontools`   | Same         |

### ❌ Platform-Specific Exclusions

These will not be migrated:

- **APT-specific tasks** - Replaced with DNF equivalents
- **Debian networking tools** - Fedora IoT uses NetworkManager
- **Initramfs updates** - Fedora IoT uses dracut

## Pre-Migration Checklist

### 1. Validate Current Debian Deployment

```bash
# Ensure playbook is idempotent on Debian
ansible-playbook site.yml --check
ansible-playbook site.yml

# Run twice to verify idempotency
ansible-playbook site.yml

# All runs should show zero changes
```

### 2. Document Current State

```bash
# Capture system state
ujust version > pre-migration-state.txt
ujust status >> pre-migration-state.txt
df -h >> pre-migration-state.txt
sudo mdadm --detail /dev/md0 >> pre-migration-state.txt

# Backup configuration
sudo tar czf keystone-backup-$(date +%Y%m%d).tar.gz \
    /etc/keystone \
    /etc/systemd/system/cockpit.service \
    /etc/systemd/system/*.mount
```

### 3. Backup Critical Data

- **Git repository** - Push all commits to remote
- **Tailscale auth key** - Store securely (cannot recover from host)
- **Podman volumes** - Backup `/mnt/ssd/podman/volumes/`
- **Cockpit data** - Backup `/mnt/ssd/cockpit/`

## Migration Steps

### Phase 1: Prepare Fedora IoT Environment

#### 1.1 Create Blueberry Boot Media

```bash
# Download latest Blueberry image
curl -LO https://github.com/your-org/blueberry/releases/latest/blueberry-rpi5.img.xz

# Flash to microSD (temporary boot, OS goes to eMMC later)
xzcat blueberry-rpi5.img.xz | sudo dd of=/dev/sdX bs=4M status=progress
sync
```

#### 1.2 Boot Blueberry and Verify

```bash
# SSH to device
ssh fedora@blueberry.local

# Check OS
cat /etc/os-release
# Should show: ID=fedora, VARIANT_ID=iot

# Verify ostree
rpm-ostree status
```

#### 1.3 Install Ansible on Fedora IoT

```bash
# Layer Ansible on ostree
sudo rpm-ostree install ansible-core
sudo systemctl reboot

# After reboot, verify
ansible --version
```

### Phase 2: Update Inventory for Fedora

#### 2.1 Modify Inventory Variables

Edit `inventory/hosts.yml`:

```yaml
all:
  children:
    keystone_hosts:
      hosts:
        localhost:
          ansible_connection: local
          ansible_python_interpreter: /usr/bin/python3
      vars:
        # Set platform to fedora
        keystone_platform: fedora  # <-- CHANGED
        
        # Storage layout remains identical
        keystone_storage:
          os_device: /dev/mmcblk0
          ssd_device: /dev/nvme0n1
          backup_devices:
            - /dev/sda
            - /dev/sdb
          ssd_mount: /mnt/ssd
          backup_mount: /mnt/backup
```

#### 2.2 Verify Variable Files Exist

Ensure all roles have Fedora-specific vars:

```bash
find roles/ -name "Fedora.yml" -type f
```

Expected output:
```
roles/base-host/vars/Fedora.yml
roles/ujust/vars/Fedora.yml
roles/storage-primitives/vars/Fedora.yml
roles/container-runtime/vars/Fedora.yml
roles/tailscale/vars/Fedora.yml
```

### Phase 3: Deploy to Fedora IoT

#### 3.1 Test in Check Mode

```bash
# Clone repository to Fedora IoT host
git clone <your-repo> /opt/keystone
cd /opt/keystone

# Install Ansible collections
ansible-galaxy install -r requirements.yml

# Dry-run
ansible-playbook site.yml --check
```

Review output for errors. Expected differences:
- Package manager switches to `dnf`
- Repository URLs change
- Some paths may differ (`/etc/mdadm.conf` vs `/etc/mdadm/mdadm.conf`)

#### 3.2 Apply Configuration

```bash
# Run playbook
ansible-playbook site.yml

# Monitor for failures
# Storage tasks may take 10-15 minutes (RAID creation)
```

#### 3.3 Verify Deployment

```bash
# Check ujust installation
ujust --list

# Verify system status
ujust status

# Check storage
ujust storage-raid-status
ujust storage-usage

# Verify Cockpit
systemctl status cockpit

# Test Tailscale
tailscale status
```

### Phase 4: Data Migration

#### 4.1 Restore Podman Volumes

```bash
# If migrating from Debian to Fedora on same hardware:
# Volumes at /mnt/ssd/podman should persist

# If fresh install:
sudo rsync -avP /backup/podman/ /mnt/ssd/podman/
```

#### 4.2 Restore Cockpit Configuration

```bash
sudo rsync -avP /backup/cockpit/ /mnt/ssd/cockpit/
sudo systemctl restart cockpit
```

#### 4.3 Authenticate Tailscale

```bash
# If not using auth key in vault
sudo tailscale up --accept-routes
# Follow browser prompt to authenticate
```

### Phase 5: Validation

#### 5.1 Functional Testing

```bash
# Test all ujust commands
ujust system-info
ujust storage-health
ujust container-list
ujust tailscale-status

# Verify Cockpit access
curl -k https://$(tailscale ip -4):9090
# Should return Cockpit HTML
```

#### 5.2 Idempotency Testing

```bash
# Run playbook twice
ansible-playbook site.yml
ansible-playbook site.yml

# Second run should show:
# ok=X changed=0 unreachable=0 failed=0
```

#### 5.3 Reboot Testing

```bash
# Reboot and verify all services auto-start
sudo reboot

# After reboot
ujust status
# All services should show "active"
```

## Platform Differences Reference

### Package Manager

| Operation        | Debian                    | Fedora                  |
|------------------|---------------------------|-------------------------|
| Update cache     | `apt update`              | `dnf check-update`      |
| Install package  | `apt install <pkg>`       | `dnf install <pkg>`     |
| Upgrade system   | `apt upgrade`             | `dnf upgrade`           |
| Clean cache      | `apt clean`               | `dnf clean all`         |

Ansible handles this via `ansible.builtin.package` + OS-specific vars.

### File Paths

| Purpose           | Debian                   | Fedora                  |
|-------------------|--------------------------|-------------------------|
| mdadm config      | `/etc/mdadm/mdadm.conf`  | `/etc/mdadm.conf`       |
| Systemd units     | `/etc/systemd/system/`   | `/etc/systemd/system/`  |
| Binary installs   | `/usr/local/bin/`        | `/usr/local/bin/`       |
| Application data  | `/usr/share/`            | `/usr/share/`           |

Most paths are identical; mdadm config is handled via vars.

### Network Configuration

| Tool              | Debian                   | Fedora IoT              |
|-------------------|--------------------------|-------------------------|
| Primary tool      | `ifupdown` / netplan     | NetworkManager          |
| Tailscale setup   | Via `/etc/default/`      | Via systemd env files   |
| Firewall          | UFW (optional)           | firewalld (default)     |

Keystone uses Tailscale for networking, abstracting this difference.

### Immutable Root

Fedora IoT uses ostree for immutable root filesystem:

```bash
# Installing host packages requires layering
sudo rpm-ostree install <package>
sudo systemctl reboot

# Ansible playbook handles this automatically
```

**Impact on Keystone:**
- Minimal - most services are containerized
- Only affects `base-host` and `storage-primitives` roles
- Ansible detects ostree and uses `rpm-ostree` when needed

## Rollback Strategy

### If Migration Fails

#### Option 1: Revert to Debian Backup

```bash
# Boot from Debian eMMC
# Restore from backup:
sudo tar xzf keystone-backup-<date>.tar.gz -C /

# Re-run Ansible
ansible-playbook site.yml
```

#### Option 2: Re-deploy Fedora IoT

```bash
# Reflash Blueberry image
# Restore data from backup
# Re-run Ansible with --check first
ansible-playbook site.yml --check
ansible-playbook site.yml
```

### Data Safety

**Storage layout remains identical:**
- RAID array metadata persists (mdadm metadata v1.2)
- Filesystem labels remain (`LABEL=keystone-ssd`)
- Mount points unchanged (`/mnt/ssd`, `/mnt/backup`)

Only the OS root (eMMC) changes during migration.

## Post-Migration Optimizations

### Leverage Fedora IoT Features

#### 1. Automatic Updates

```bash
# Enable automatic ostree updates
sudo systemctl enable rpm-ostreed-automatic.timer
sudo systemctl start rpm-ostreed-automatic.timer
```

#### 2. Greenboot Health Checks

```bash
# Add custom health check
sudo tee /etc/greenboot/check/required.d/01-keystone-health.sh << 'EOF'
#!/bin/bash
# Verify critical services
systemctl is-active podman.socket tailscaled cockpit
EOF

sudo chmod +x /etc/greenboot/check/required.d/01-keystone-health.sh
```

#### 3. systemd-homed (Optional)

Consider using systemd-homed for user management if adding interactive users.

### Update Ansible Playbook

Add Fedora-specific enhancements:

```yaml
# roles/base-host/tasks/main.yml
- name: Configure rpm-ostree automatic updates
  ansible.builtin.systemd:
    name: rpm-ostreed-automatic.timer
    enabled: true
    state: started
  when: ansible_distribution == 'Fedora'
  tags: base
```

## Troubleshooting

### Issue: DNF Fails to Install Package

**Symptom:**
```
FAILED! => {"msg": "No package matching 'xyz' found available"}
```

**Solution:**
Check package name in Fedora:
```bash
dnf search <package>
```

Update `roles/*/vars/Fedora.yml` with correct name.

### Issue: Ostree Transaction Conflicts

**Symptom:**
```
error: transaction is already active
```

**Solution:**
```bash
sudo rpm-ostree cancel
ansible-playbook site.yml
```

### Issue: Systemd Units Not Starting

**Symptom:**
```
Job for cockpit.service failed
```

**Solution:**
```bash
# Check journal
journalctl -xeu cockpit.service

# Common issues:
# - Podman socket not running
# - Image pull failed
# - Volume mount permissions

# Verify dependencies
systemctl status podman.socket
podman info
```

### Issue: Storage Not Mounting

**Symptom:**
```
Failed to mount /mnt/ssd
```

**Solution:**
```bash
# Verify labels
sudo blkid | grep keystone

# Check mount unit
systemctl status mnt-ssd.mount

# Manual mount for debugging
sudo mount -L keystone-ssd /mnt/ssd
```

## Timeline & Effort Estimate

| Phase                  | Duration       | Effort              |
|------------------------|----------------|---------------------|
| Pre-migration backup   | 30 minutes     | Low                 |
| Blueberry install      | 15 minutes     | Low                 |
| Ansible deploy (Fedora)| 20 minutes     | Low                 |
| Data migration         | 1-4 hours      | Medium (disk speed) |
| Validation testing     | 30 minutes     | Low                 |
| **Total**              | **2.5-5 hours**| **Medium**          |

*Assumes no unexpected issues; RAID sync time variable.*

## Success Criteria

Migration is complete when:

✅ `ansible-playbook site.yml` runs idempotently on Fedora IoT  
✅ All ujust commands function identically to Debian  
✅ Storage layout matches AGENTS.md contract  
✅ Cockpit accessible via Tailscale  
✅ RAID health verified (`ujust storage-raid-status`)  
✅ All services survive reboot  
✅ No manual configuration required outside Git

## References

- [Fedora IoT Documentation](https://docs.fedoraproject.org/en-US/iot/)
- [rpm-ostree](https://coreos.github.io/rpm-ostree/)
- [Blueberry Project](../blueberry/)
- [AGENTS.md](../AGENTS.md) - Migration constraints (§9)
