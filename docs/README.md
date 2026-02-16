# Keystone NAS Platform - Architecture Documentation

## Overview

Keystone is a **host-layer infrastructure system** for managing a Raspberry Pi CM5-based NAS platform using GitOps principles. It provisions and governs the host OS, storage primitives, container runtime, and essential infrastructure services—but **not** application workloads.

This repository represents **Layer 1** (Host OS) in a two-layer architecture:
- **Layer 1 (This Repository)**: Host platform, storage, Podman, Tailscale, Cockpit
- **Layer 2 (Separate Repository)**: Application containers, user services, data workloads

## Design Principles

### 1. GitOps-First
- Git is the **single source of truth** for host configuration
- All changes are reviewable, reversible, and auditable
- No manual configuration outside version control
- Drift is considered a failure condition

### 2. Idempotent & Declarative
- Ansible playbooks are safely re-runnable
- State is declared, not scripted
- No side effects from repeated execution
- Prefer Ansible modules over shell commands

### 3. OS-Portable
All design decisions minimize migration friction between:
- **Current Platform**: Raspberry Pi OS Lite (Debian Trixie)
- **Target Platform**: Blueberry (Fedora IoT for RPi5)

OS-specific logic is isolated in `vars/` files; playbooks remain platform-agnostic.

### 4. Container-Native
- Services run as containers wherever possible
- Cockpit uses Podman + systemd Quadlet pattern
- Minimal host package footprint
- Aligns with immutable infrastructure principles

### 5. Tailnet-Only Security Model
- All services bind to Tailscale network only
- No public internet exposure by default
- Firewall enforces tailnet-only access
- SSH restricted to Tailscale IPs

## Architecture Components

### Role Structure

```
roles/
├── base-host/           # Minimal OS configuration, packages, sysctl
├── ujust/               # Task runner (blueberry-compatible)
├── storage-primitives/  # RAID, filesystems, mount units
├── container-runtime/   # Podman, Quadlet, storage config
├── tailscale/           # VPN client, firewall rules
└── cockpit-container/   # Web UI (containerized)
```

Each role has clear boundaries and responsibilities (see AGENTS.md §8).

### Storage Layout

Per AGENTS.md §4, storage is strictly partitioned:

| Device          | Purpose                     | Filesystem | Mount Point     |
|-----------------|-----------------------------|------------|-----------------|
| eMMC            | OS only (immutable)         | ext4       | /               |
| M.2 SSD         | Containers + writable state | XFS        | /mnt/ssd        |
| 2× HDD (RAID1)  | Backup storage              | ext4       | /mnt/backup     |

**Non-negotiable rules:**
- eMMC is OS-only; no user data
- SSD is the only writable system disk
- HDDs are backup-only
- Mount points are managed via systemd units

### ujust Task Runner

Keystone includes a `ujust` system mirroring blueberry's approach:

```
/usr/share/keystone/
├── justfile                 # Main entry point
└── just/
    ├── 00-system.just       # System maintenance
    ├── 10-storage.just      # Storage operations
    ├── 20-containers.just   # Container management
    ├── 30-tailscale.just    # Network tasks
    └── 60-custom.just       # User extensions
```

Operators use `ujust <task>` for common operations instead of remembering Ansible commands.

## Key Files

### Inventory
- `inventory/hosts.yml`: Host definitions and global variables
- `inventory/group_vars/`: Group-level configuration
- `inventory/host_vars/`: Host-specific overrides

### Playbooks
- `site.yml`: Main playbook; orchestrates all roles
- Tags support selective execution: `--tags storage`, `--tags cockpit`

### Configuration
- `ansible.cfg`: Ansible behavior, privilege escalation, output format
- `requirements.yml`: External Ansible collections (Galaxy)

## Deployment Workflow

### Initial Provisioning

```bash
# 1. Install Ansible dependencies
ansible-galaxy install -r requirements.yml

# 2. Verify inventory
ansible-inventory --list

# 3. Run playbook (dry-run first)
ansible-playbook site.yml --check

# 4. Apply configuration
ansible-playbook site.yml

# 5. Verify ujust installation
ujust --list
```

### Incremental Changes

```bash
# Run specific roles only
ansible-playbook site.yml --tags storage
ansible-playbook site.yml --tags cockpit

# Reconfigure from ujust
ujust reconfigure
```

### Idempotency Testing

```bash
# Run twice; second run should show zero changes
ansible-playbook site.yml
ansible-playbook site.yml
```

## OS Abstraction Strategy

### Package Manager

Roles use `ansible.builtin.package` with OS-specific variables:

```yaml
# roles/base-host/vars/Debian.yml
base_host_packages:
  - apt-transport-https
  - python3-pip

# roles/base-host/vars/Fedora.yml
base_host_packages:
  - python3-pip
```

Playbooks load the correct vars file based on `ansible_distribution`.

### Systemd Units

Systemd units are portable across Debian and Fedora:
- Placed in `/etc/systemd/system/` (standard override location)
- Use conservative settings (Restart=always, resource limits)
- Network dependencies: `After=network-online.target`

### Filesystem Paths

Avoid OS-specific paths:
- `/usr/local/bin/` - portable binary location
- `/etc/systemd/system/` - custom units
- `/usr/share/keystone/` - application data
- `/var/lib/keystone/` - stateful data

## Security Model

### Secret Management

**Prohibited:**
- Secrets in Git (plaintext or encrypted)
- Hardcoded credentials in playbooks
- Secrets baked into container images

**Allowed:**
- Ansible Vault for sensitive vars
- Environment files outside repo (`/etc/keystone/secrets.env`)
- Runtime secret injection (Podman secrets)

Example:
```yaml
# group_vars/keystone_hosts/vault.yml (encrypted)
tailscale_auth_key: "tskey-auth-..."
```

### Firewall Rules

- Default policy: **DENY**
- Allow SSH from Tailscale network only (100.64.0.0/10)
- Allow Cockpit from Tailscale network only
- No services on `0.0.0.0` without explicit justification

## Migration Path: Debian → Fedora IoT

### What Transfers Directly

✅ **Systemd units** - Portable as-is  
✅ **ujust justfiles** - No changes needed  
✅ **Storage management** - mdadm, filesystems identical  
✅ **Podman configuration** - Same tools, same behavior  
✅ **Tailscale setup** - Package name changes, logic identical

### What Requires Changes

⚠️ **Package installation** - Swap `vars/Debian.yml` → `vars/Fedora.yml`  
⚠️ **Repository setup** - Different GPG keys, repo URLs  
⚠️ **Boot configuration** - Fedora IoT uses ostree; playbook skips this

### Migration Checklist

1. Update inventory: Set `keystone_platform: fedora`
2. Test playbook in Fedora IoT VM
3. Verify all roles execute without errors
4. Confirm storage layout matches
5. Validate ujust commands function identically
6. Test idempotency on Fedora IoT

The playbook is designed to require **zero role-level changes** for migration—only variable swaps.

## Troubleshooting

### Ansible Fails on First Run

**Symptom:** Tasks fail with package errors  
**Solution:** Ensure correct OS detected:
```bash
ansible -m setup localhost | grep ansible_distribution
```

### Storage Mount Fails

**Symptom:** Mount units fail to start  
**Solution:** Check device labels:
```bash
sudo blkid | grep keystone
```

Recreate labels if missing:
```bash
sudo xfs_admin -L keystone-ssd /dev/nvme0n1
```

### Cockpit Not Accessible

**Symptom:** Cannot reach https://tailscale-ip:9090  
**Solution:** Verify service and firewall:
```bash
sudo systemctl status cockpit
sudo ufw status | grep 9090
tailscale status
```

### ujust Command Not Found

**Symptom:** `ujust: command not found`  
**Solution:** Ensure role executed:
```bash
ansible-playbook site.yml --tags ujust
which ujust
```

## Maintenance

### Regular Tasks

```bash
# Update system packages
ujust system-update

# Check RAID health
ujust storage-raid-status

# Check disk health
ujust storage-health

# View container status
ujust container-list
```

### Monthly Tasks

```bash
# RAID scrub (verify data integrity)
ujust storage-raid-scrub

# Container cleanup
ujust container-prune
```

### Disaster Recovery

1. **Backup Git repository** - Clone to external location
2. **Backup `/etc/keystone/`** - Contains version tracking
3. **Backup Tailscale auth** - Store auth key securely
4. **Backup Podman volumes** - Located in `/mnt/ssd/podman/`

Restore:
1. Install fresh OS
2. Clone this repository
3. Run `ansible-playbook site.yml`
4. Restore Podman volumes
5. Run `tailscale up` with auth key

## Design Decisions Log

### Why ujust Instead of Direct Ansible?

**Rationale:**
- Operator UX: `ujust storage-health` > `ansible-playbook site.yml --tags storage-health`
- Blueberry compatibility: Maintains muscle memory for Fedora IoT users
- Flexibility: ujust can run non-Ansible tasks (scripts, ad-hoc commands)

### Why Cockpit as Container?

**Rationale:**
- Aligns with container-first principle
- Avoids host package pollution
- Easier to update (pull new image vs. package upgrade)
- Portable to Fedora IoT without changes

### Why RAID1 Not ZFS?

**Rationale:**
- ZFS overhead inappropriate for RPi5 + HDDs
- mdadm is simpler, more portable
- ZFS requires kernel modules (complicates OS migration)
- RAID1 sufficient for backup use case

### Why systemd Mounts Not /etc/fstab?

**Rationale:**
- systemd units are declarative (fits GitOps model)
- Better dependency management (network, device readiness)
- Easier to manage via Ansible (templates, handlers)
- Fedora IoT encourages systemd-first approach

## References

- [AGENTS.md](../AGENTS.md) - Architectural governance
- [Blueberry](../blueberry/) - Reference implementation
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
