Reproducible CM5 NAS Host Platform (GitOps + Container-Native)

---

## 1. Architectural Intent

This repository defines the **host-layer** of a reproducible, GitOps-driven NAS platform targeting a **Raspberry Pi CM5**.

It is responsible for:

- Provisioning and governing the **host OS**
- Managing **storage primitives**
- Installing and configuring the **container runtime**
- Running **host-bound infrastructure containers only** (Cockpit)

It is **not** responsible for application or user-facing services.  
Those are managed in a **separate repository** dedicated to containerized services.

### Supported OS Strategy

- **Current OS:** Raspberry Pi OS Lite (Debian Trixie)
- **Future Target:** Fedora IoT (custom image builds anticipated)

All design decisions must **minimize migration friction** to Fedora IoT.

This architectural direction is fixed.

This repository is a **long-lived infrastructure system**, not a script collection.

---

## 2. Scope Definition (Critical)

### 2.1 In Scope (This Repository)

This repository owns:

- Host OS configuration (minimal, governed)
- Disk discovery and provisioning
- RAID configuration for backup disks
- Filesystem creation and mount units
- Podman installation and policy
- Systemd units for container orchestration
- Tailscale client installation and policy
- Cockpit (containerized, host-managed)

### 2.2 Explicitly Out of Scope

This repository **must not** manage:

- Application services (e.g. media servers, databases, sync tools)
- User workloads
- Backup jobs or retention logic
- Monitoring stacks
- Any container other than Cockpit

Those belong in the **services repository**, which consumes this host as a substrate.

The LLM agent must refuse changes that violate this boundary.

---

## 3. System Design Principles

### 3.1 Layering Model

#### Layer 0 — Firmware / Boot (Out of Scope)
- Hardware, EEPROM, bootloader configuration

#### Layer 1 — Host OS (This Repository)

- Minimal OS configuration
- Storage primitives only
- Podman runtime
- Tailscale client
- Systemd orchestration
- Cockpit container

No application logic.

#### Layer 2 — Container Services (Separate Repository)

- All non-host infrastructure
- Application containers
- Data services
- User-facing workloads

Host and services must be **independently reproducible**.

---

## 4. Storage Layout Contract (Non-Negotiable)

| Component                          | Device          | Purpose               |
|-----------------------------------|-----------------|-----------------------|
| OS                                | eMMC            | Immutable system base |
| Containers + writable host state  | M.2 SSD         | Runtime + metadata    |
| Backup storage                    | 2 × HDD (RAID1) | Durable backups       |

Rules:

- eMMC is **OS only**
- M.2 SSD is the **only writable system disk**
- HDDs are **backup-only**
- RAID metadata format must be pinned
- Mount points must be stable and explicit

The LLM agent must never redesign this layout unless explicitly instructed.

---

## 5. Governance Rules

### 5.1 Idempotency

All automation must:

- Be safely re-runnable
- Be declarative
- Prefer Ansible modules over shell/command
- Use guards (`creates`, `unless`, state checks) when unavoidable

Manual host changes are **configuration drift**.

---

### 5.2 Determinism & Version Pinning

The system must be reproducible.

Pin:

- Podman versions (where possible)
- Container image tags or digests
- Ansible role versions
- systemd unit templates
- RAID and filesystem parameters

Upgrades must be explicit, reviewed, and documented in Git.

---

### 5.3 Image Provenance

Container images must:

- Come from trusted upstream registries
- Prefer official or well-maintained images
- Use explicit tags or digests
- Never use `latest`
- Never embed secrets
- Remain portable to Fedora IoT

Local image builds are forbidden unless fully defined in-repo.

---

### 5.4 Secret Handling

Secrets must:

- Never be committed to Git
- Never appear in plaintext systemd units
- Never be baked into images

Allowed mechanisms:

- Environment files stored outside the repo
- Ansible Vault
- Runtime secret mounts
- Podman secrets (future Fedora IoT alignment)

The LLM agent must refuse to generate code that embeds credentials.

---

### 5.5 Host Mutation Boundaries

The host OS may only be modified via:

- Ansible roles in this repository
- Version-controlled systemd unit files
- Declarative storage configuration

Forbidden:

- Manual edits
- One-off commands
- Interactive configuration
- Snowflake fixes

Drift must be reconciled through code.

---

### 5.6 GitOps Discipline

This repository is the **single source of truth** for the host.

Rules:

- All changes via pull request
- Every change must be reviewable
- Commits must explain *architectural intent*
- Prefer incremental refactors
- No host-only configuration outside Git

---

## 6. Cockpit Design Contract

Cockpit is the **only container managed here**.

Requirements:

- Runs as a Podman container
- Managed via systemd
- No cockpit packages installed on host
- Minimal, explicit volume bindings
- Network exposure aligned with tailnet-only access
- Compatible with Fedora IoT container patterns

The systemd unit is part of the host layer and must be version controlled.

---

## 7. Networking & Access Model

Trust boundary:

- Only devices on the Tailscale tailnet are trusted

Rules:

- Services bind only to:
  - Tailscale interface
  - Localhost where appropriate
- No service may listen on `0.0.0.0` without explicit justification
- Firewall rules must enforce this model

The LLM agent must not weaken these constraints.

---

## 8. Repository Structure Expectations

Expected high-level structure:
```code
inventory/
roles/
base-host/
storage-primitives/
container-runtime/
tailscale/
cockpit-container/
group_vars/
host_vars/
systemd/
containers/
docs/
AGENTS.md
README.md
```

Separation of concerns is mandatory:

- `base-host` → minimal OS config
- `storage-primitives` → disks, RAID, filesystems, mounts
- `container-runtime` → Podman + policies
- `cockpit-container` → Cockpit unit and configuration

No service logic in storage roles.  
No storage logic in container roles.

---

## 9. Migration Strategy (Debian → Fedora IoT)

Design must anticipate:

- Immutable / ostree-style systems
- Reduced host package surface
- Container-native service delivery
- systemd-first orchestration

Avoid:

- Debian-only hacks
- `apt`-specific assumptions
- Hard-coded filesystem paths that diverge on Fedora IoT

Every design choice should be evaluated for migration impact.

---

## 10. Responsibilities of the LLM Agent

The LLM agent must:

- Respect all architectural boundaries
- Avoid redesigning constraints unless explicitly asked
- Propose minimal, reviewable changes
- Explain trade-offs clearly
- Refactor safely without changing behavior unless instructed
- Avoid introducing drift
- Never invent infrastructure components
- Never suggest manual steps outside IaC
- Challenge unsafe or non-deterministic patterns
- Prefer declarative solutions

If uncertain, ask for clarification rather than assume.

---

## 11. Change Management Expectations

When proposing changes, the LLM must:

1. State architectural impact
2. Confirm alignment with:
   - Idempotency
   - Determinism
   - Host mutation boundaries
3. Describe migration implications
4. Suggest the smallest viable change
5. Avoid overengineering

---

## 12. What This Repository Is Not

- Not an application platform
- Not a services repository
- Not a backup orchestration system
- Not manually administered infrastructure
- Not mutable state outside Git

---

## 13. Long-Term Evolution

This project is expected to evolve toward:

- Custom Fedora IoT images
- Fully container-native host services
- Minimal host footprint
- Strong contract with a separate services repository
- Strict GitOps discipline

Architectural stability outweighs feature velocity.

---

## 14. Final Governance Rule

If a change:

- Breaks reproducibility
- Introduces drift
- Blurs host vs service boundaries
- Embeds secrets
- Weakens the tailnet-only trust model

**It must not be merged.**

This document defines governance for both humans and AI agents.  
It is a long-term contract, not a task checklist.
