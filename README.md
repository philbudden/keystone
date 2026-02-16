# Keystone NAS Platform

**Reproducible, GitOps-driven host layer for Raspberry Pi CM5 NAS**

> :curling_stone: Provides the storage and host substrate for all managed services

Keystone is an Ansible-based infrastructure system that provisions and governs the **host layer only** of a container-native NAS platform. It manages OS configuration, storage primitives, container runtime, and essential infrastructure—but **not** application services.

## Quick Start

```bash
# 1. Clone repository
git clone <your-repo-url> /opt/keystone
cd /opt/keystone

# 2. Install Ansible dependencies
ansible-galaxy install -r requirements.yml

# 3. Review and customize inventory
vim inventory/hosts.yml

# 4. Deploy (dry-run first)
ansible-playbook site.yml --check
ansible-playbook site.yml

# 5. Verify installation
ujust --list
ujust status
```

## What is Keystone?

Keystone implements **Layer 1** (Host OS) in a two-layer architecture:

- **Layer 1 (This Repository)**: Host platform, storage, Podman, Tailscale, Cockpit
- **Layer 2 (Separate Repository)**: Application containers, user services, data workloads

This separation ensures:
- Host and services are **independently reproducible**
- Clear architectural boundaries (no scope creep)
- GitOps discipline (Git is single source of truth)
- Migration-ready (Debian → Fedora IoT with minimal friction)

## Platform Support

| Platform                     | Status      | Notes                          |
|------------------------------|-------------|--------------------------------|
| Raspberry Pi OS Lite (Trixie)| ✅ Primary  | Current target                 |
| Fedora IoT (Blueberry)       | ✅ Planned  | Future target, minimal changes |
| Other Debian/Fedora          | ⚠️ Untested | May work with inventory changes|

All design decisions minimize Fedora IoT migration friction (see [Migration Guide](docs/MIGRATION.md)).

## Features

### ✅ Host Management
- Minimal OS configuration (packages, sysctl, timezone)
- OS-abstracted package management (Debian ↔ Fedora)
- Systemd-first orchestration

### ✅ Storage Primitives
- Software RAID (mdadm) for backup disks
- Filesystem management (XFS for SSD, ext4 for RAID)
- Systemd mount units (GitOps-compatible)
- SMART monitoring and health checks

### ✅ Container Runtime
- Podman installation and configuration
- Quadlet support for systemd integration
- Container storage on dedicated SSD

### ✅ Network Security
- Tailscale VPN integration
- Tailnet-only firewall rules (no public exposure)
- SSH restricted to Tailscale network

### ✅ Infrastructure UI
- Cockpit web interface (containerized)
- Accessible via Tailscale only
- Podman + systemd management

### ✅ Operator UX
- `ujust` task runner (blueberry-compatible)
- Intuitive command interface (`ujust storage-health`)
- Self-documenting (`ujust --list`)

## Architecture

### Storage Layout

Per [AGENTS.md](AGENTS.md), storage is strictly partitioned:

| Device          | Purpose                     | Filesystem | Mount Point |
|-----------------|-----------------------------|------------|-------------|
| eMMC            | OS only (immutable)         | ext4       | /           |
| M.2 SSD         | Containers + writable state | XFS        | /mnt/ssd    |
| 2× HDD (RAID1)  | Backup storage              | ext4       | /mnt/backup |

**Rules:**
- eMMC is OS-only (no user data)
- SSD is the only writable system disk
- HDDs are backup-only

### Role Structure

```
roles/
├── base-host/           # Minimal OS configuration
├── ujust/               # Task runner (blueberry-style)
├── storage-primitives/  # RAID, filesystems, mounts
├── container-runtime/   # Podman, Quadlet
├── tailscale/           # VPN client, firewall
└── cockpit-container/   # Web UI (containerized)
```

Each role has clear boundaries and responsibilities. See [Architecture Docs](docs/README.md).

## Usage

### Common Operations

```bash
# System maintenance
ujust system-update        # Update packages
ujust system-info          # Show system info
ujust system-clean         # Clean package cache

# Storage management
ujust storage-raid-status  # Check RAID health
ujust storage-health       # Check disk SMART status
ujust storage-usage        # Show disk usage

# Container management
ujust container-list       # List running containers
ujust container-stats      # Show resource usage
ujust container-cockpit-restart  # Restart Cockpit

# Network operations
ujust tailscale-status     # Show tailnet status
ujust tailscale-ip         # Get Tailscale IP
```

See [Operations Guide](docs/OPERATIONS.md) for complete command reference.

### Selective Deployment

```bash
# Deploy only storage configuration
ansible-playbook site.yml --tags storage

# Deploy only Cockpit
ansible-playbook site.yml --tags cockpit

# Skip network configuration
ansible-playbook site.yml --skip-tags network
```

### Idempotency Testing

```bash
# Run twice; second run should show zero changes
ansible-playbook site.yml
ansible-playbook site.yml
```

## Documentation

- **[Architecture Overview](docs/README.md)** - Design principles, components, deployment workflow
- **[Migration Guide](docs/MIGRATION.md)** - Debian → Fedora IoT migration path
- **[Operations Guide](docs/OPERATIONS.md)** - Complete ujust command reference
- **[AGENTS.md](AGENTS.md)** - Architectural governance and constraints

## Design Principles

### 1. GitOps-First
- Git is the **single source of truth**
- All changes are reviewable and auditable
- No manual configuration outside version control

### 2. Idempotent & Declarative
- Ansible playbooks are safely re-runnable
- State is declared, not scripted
- No side effects from repeated execution

### 3. OS-Portable
- Abstracts Debian vs Fedora differences
- OS-specific logic isolated in `vars/` files
- Playbooks remain platform-agnostic

### 4. Container-Native
- Services run as containers wherever possible
- Minimal host package footprint
- Aligns with immutable infrastructure principles

### 5. Tailnet-Only Security
- All services bind to Tailscale network only
- No public internet exposure
- Firewall enforces tailnet-only access

## Scope Boundaries

### ✅ In Scope (This Repository)
- Host OS configuration
- Disk discovery and provisioning
- RAID configuration
- Filesystem creation and mount units
- Podman installation and policy
- Tailscale client installation
- Cockpit (containerized, host-managed)

### ❌ Out of Scope
- Application services (media servers, databases, etc.)
- User workloads
- Backup jobs or retention logic
- Monitoring stacks
- Any container other than Cockpit

Application services belong in a **separate repository** that consumes this host as a substrate.

## Requirements

### Hardware
- Raspberry Pi CM5 (or compatible RPi5-series)
- eMMC module (OS)
- M.2 NVMe SSD (container storage)
- 2× SATA HDDs (backup storage)

### Software
- Raspberry Pi OS Lite (Debian Trixie) **or** Fedora IoT
- Ansible 2.15+
- Python 3.9+

### Network
- Tailscale account (for VPN access)

## Installation

### Prerequisites

```bash
# On control machine (laptop/workstation)
pip install ansible

# On target host (Raspberry Pi)
# Ensure SSH access and sudo privileges
```

### ⚠️ Important Safety Warnings

**CRITICAL: Storage operations are destructive!**

Before running the playbook:

1. **Backup all data** on devices specified in inventory
2. **Verify device paths** - incorrect paths will destroy wrong disks
3. **RAID creation will wipe backup devices** completely
4. **Filesystem creation only runs on empty devices** - existing filesystems are preserved

The playbook includes safety checks:
- ✅ Filesystem creation skipped if device already has a filesystem
- ✅ RAID creation verifies no existing mdadm metadata
- ✅ Mount dependencies enforced (Podman/Cockpit require SSD)
- ✅ Idempotent operations - safe to re-run

**First-time deployment:**
```bash
# 1. BACKUP YOUR DATA
# 2. Verify inventory device paths match your hardware
vim inventory/hosts.yml

# 3. Dry-run to see what will change
ansible-playbook site.yml --check --diff

# 4. Review all storage-related tasks carefully
ansible-playbook site.yml --tags storage --check --diff
```

### Deployment Steps

1. **Clone repository**
   ```bash
   git clone <your-repo-url> /opt/keystone
   cd /opt/keystone
   ```

2. **Install Ansible collections**
   ```bash
   ansible-galaxy install -r requirements.yml
   ```

3. **Customize inventory**
   ```bash
   vim inventory/hosts.yml
   # Update storage device paths if needed
   ```

4. **Deploy infrastructure**
   ```bash
   # Dry-run first
   ansible-playbook site.yml --check
   
   # Apply configuration
   ansible-playbook site.yml
   ```

5. **Verify installation**
   ```bash
   ujust --list
   ujust status
   ```

6. **Access Cockpit**
   ```bash
   # Get Tailscale IP
   ujust tailscale-ip
   
   # Access via browser
   # https://<tailscale-ip>:9090
   ```

## Customization

### Changing Storage Layout

Edit `inventory/hosts.yml`:

```yaml
keystone_storage:
  os_device: /dev/mmcblk0      # Change if using SD card
  ssd_device: /dev/nvme0n1     # Change to /dev/sda if using SATA
  backup_devices:
    - /dev/sda                 # Update based on actual devices
    - /dev/sdb
  ssd_mount: /mnt/ssd
  backup_mount: /mnt/backup
```

### Adding Custom Tasks

Edit `/usr/share/keystone/just/60-custom.just`:

```just
# Custom backup task
custom-backup:
    @echo "Starting backup..."
    @rsync -avz /mnt/ssd/ /mnt/backup/ssd-backup/
    @echo "Backup complete!"
```

### Secrets Management

Use Ansible Vault for sensitive data:

```bash
# Create encrypted vars file
ansible-vault create inventory/group_vars/keystone_hosts/vault.yml

# Add Tailscale auth key
tailscale_auth_key: "tskey-auth-..."

# Deploy with vault password
ansible-playbook site.yml --ask-vault-pass
```

## Troubleshooting

### Ansible Fails on First Run

**Symptom:** Package installation errors

**Solution:** Verify OS detection
```bash
ansible -m setup localhost | grep ansible_distribution
```

### Storage Mounts Fail

**Symptom:** Mount units don't start

**Solution:** Check device labels
```bash
sudo blkid | grep keystone
```

### Cockpit Not Accessible

**Symptom:** Cannot reach web UI

**Solution:** Verify firewall and Tailscale
```bash
sudo systemctl status cockpit
tailscale status
```

See [Architecture Docs](docs/README.md#troubleshooting) for detailed troubleshooting.

## Contributing

1. **Understand scope boundaries** - Read [AGENTS.md](AGENTS.md) first
2. **Make minimal changes** - Surgical edits only
3. **Test on both platforms** - Debian and Fedora IoT
4. **Document decisions** - Explain *why*, not just *what*
5. **Commit semantically** - Use conventional commits

## License

See [LICENSE](LICENSE)

## Acknowledgments

- **Blueberry** - Reference implementation for ujust system
- **Fedora IoT** - Target platform architecture
- **Ansible** - Infrastructure as Code tooling
