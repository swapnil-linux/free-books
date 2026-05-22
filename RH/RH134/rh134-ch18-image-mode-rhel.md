# RH134 Chapter 18 - Working with Image-based Red Hat Enterprise Linux

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Describe, create, install, and manage Red Hat Enterprise Linux in image mode, which uses container images to define and deliver the operating system.

---

## Windows vs Linux: Image Mode Equivalents

| Windows Concept | Linux Image Mode Equivalent |
|---|---|
| Windows Update (individual patches) | `bootc upgrade` (atomic image replacement) |
| Golden image / WIM deployment | bootc image pushed to registry |
| WSUS / SCCM patch management | Container registry + `bootc upgrade` |
| Group Policy for config consistency | Containerfile defines all config - structural consistency |
| System Restore / Restore Point | `bootc rollback` (boot previous OSTree deployment) |
| "Gold standard" server clone | All servers run the same image tag - identical by design |
| `sfc /scannow` (integrity check) | Immutable composefs root - integrity is the default |
| Sysprep image capture | `podman build` + `podman push` to registry |

---

## 1. Package Mode vs Image Mode

| | Package Mode (Traditional) | Image Mode (New in RHEL 10) |
|---|---|---|
| OS delivery | RPM packages via `dnf` | Container image from registry |
| Updates | `dnf update` on running system | Build new image, push, `bootc upgrade`, reboot |
| Root filesystem | Mutable (writable) | **Immutable** (read-only via composefs) |
| Configuration | Files edited on each server | Defined in Containerfile, baked into image |
| Drift prevention | Manual auditing, Ansible | Structural - every server running same tag is identical |
| Rollback | `dnf downgrade` (complex) | `bootc rollback` (boot previous deployment) |
| System state verification | `rpm -Va`, manual checks | Compare to known image digest |
| Managed with | `dnf`, `rpm`, config files | `bootc`, Containerfile, container registry |
| Reboot needed for OS update | No | **Always** |

> **The key insight:** In image mode, the OS is treated exactly like an application container. The same tools (Podman, container registries) manage both. The difference is that the container image boots as a full operating system.

### Why Image Mode?

| Problem (Package Mode) | Solution (Image Mode) |
|---|---|
| Infrastructure drift (servers diverge over time) | All servers boot the same immutable image |
| "Works on my server, not on yours" | Every server running the same tag is bit-for-bit identical |
| Rollback is complex and error-prone | `bootc rollback` + reboot - atomic and reliable |
| Security: attacker can modify OS binaries | Immutable root - modifications are lost on reboot |
| Compliance: prove systems match intended state | Image digest = cryptographic proof of state |

---

## 2. The Image Mode Workflow

```
┌─────────────────────────────────────────────────────────┐
│  BUILD                                                  │
│  Write Containerfile with FROM rhel-bootc               │
│  podman build -t registry/user/myos:latest .            │
└─────────────────────────┬───────────────────────────────┘
                          |
                          v
┌─────────────────────────────────────────────────────────┐
│  PUSH                                                   │
│  podman push registry/user/myos:latest                  │
│  (Registry becomes source of truth)                     │
└─────────────────────────┬───────────────────────────────┘
                          |
                          v
┌─────────────────────────────────────────────────────────┐
│  DEPLOY (first time)                                    │
│  Kickstart with ostreecontainer --url=REGISTRY/IMAGE    │
│  (replaces %packages section)                           │
└─────────────────────────┬───────────────────────────────┘
                          |
                          v
┌─────────────────────────────────────────────────────────┐
│  MANAGE (ongoing)                                       │
│  Edit Containerfile -> rebuild -> push (same tag)       │
│  bootc upgrade (on each system, stages new image)       │
│  reboot (applies staged image atomically)               │
│  bootc rollback (if needed - boots previous deployment) │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Creating a Bootable Container Image

### The `rhel-bootc` Base Image

Unlike application containers (which use `ubi10/ubi`), bootable container images must use `rhel-bootc` as the base. This image includes the kernel, boot loader, systemd, and all other components needed to boot a real server.

```bash
# Pull the base image
podman pull registry.redhat.io/rhel10/rhel-bootc:latest
```

> The `rhel-bootc` image is subject to the Red Hat EULA. You cannot publicly redistribute it or images derived from it.

### Containerfile for Image Mode vs Application Mode

| Difference | Application Container | Image Mode (bootc) |
|---|---|---|
| Base image | `FROM ubi10/ubi` | `FROM rhel-bootc:latest` |
| Starting services | `ENTRYPOINT ["/usr/sbin/httpd"]` | `RUN systemctl enable httpd` |
| Port exposure | `EXPOSE 80` (documented only) | `RUN firewall-cmd --add-service=https` |
| `CMD` / `ENTRYPOINT` | Used to define what runs | Replaced by systemd |
| `ENV` at runtime | Works as expected | Replaced by systemd service config |

### Example Bootable Containerfile

```dockerfile
# Use the rhel-bootc base image (not ubi10/ubi!)
FROM registry.redhat.io/rhel10/rhel-bootc:latest

# Install packages and clean up (creates a new image layer)
RUN dnf -y install httpd mod_ssl && dnf clean all

# Enable services via systemd (not ENTRYPOINT)
RUN systemctl enable httpd && firewall-cmd --add-service=https

# Copy configuration files
ADD ./etc/ /etc
COPY ./index.html /var/www/html/index.html
```

> **Key difference from application containers:**
> - Use `systemctl enable` instead of `ENTRYPOINT` to start services
> - Use `firewall-cmd` instead of `EXPOSE` to open ports
> - `ENTRYPOINT` and `CMD` are ignored in image mode
> - Systemd runs as PID 1 in a deployed image-mode system

### Building and Pushing the Image

```bash
# Build the bootable container image
# --squash merges all new layers into one (reduces complexity)
podman build --squash -t registry.example.com/user/myos:latest .

# Test locally BEFORE deploying to hardware (runs as an application container)
podman run -d -p 8080:80 registry.example.com/user/myos

# Push to registry (makes it available for deployment)
podman push registry.example.com/user/myos:latest

# Tag a specific version alongside the :latest tag
podman tag registry.example.com/user/myos:latest \
           registry.example.com/user/myos:v1.0
podman push registry.example.com/user/myos:v1.0
```

---

## 4. Installing RHEL in Image Mode with Kickstart

### Key Difference from Package Mode Kickstart

In package mode, you use `%packages` to specify what to install. In image mode, you replace `%packages` with the `ostreecontainer` command pointing to your bootable container image.

```bash
# Package mode Kickstart (remove this section entirely for image mode):
%packages
@^minimal-environment
vim-enhanced
%end

# Image mode replacement (add this single line):
ostreecontainer --url=registry.example.com:5000/user/myos:latest
```

### Minimal Image Mode Kickstart File

```bash
# Use text display mode
text

# Storage configuration
clearpart --all --initlabel --disklabel=gpt
reqpart --add-boot
part / --grow --fstype xfs

# Network
network --bootproto=dhcp --device=link --activate
firewall --disabled

# User management (bootc image has no default user - must create one)
user --name="student" --groups="wheel" --plaintext --password="student"
rootpw --plaintext redhat

# THIS replaces the %packages section:
ostreecontainer --url=registry.lab.example.com:5000/user/webserver-bootc

# Pre-install script (authenticate to registry before installation)
%pre
skopeo login -u student -p redhat registry.lab.example.com:5000 \
    --authfile=/run/ostree/auth.json
%end

# Disable kdump
%addon com_redhat_kdump --disable
%end
```

> **No `%packages` section in image mode Kickstart.** The `ostreecontainer` command tells Anaconda to pull and deploy the bootable container instead of installing RPM packages.

### Key Points for Image Mode Installation

- The `rhel-bootc` image contains no default user accounts - always create users in Kickstart
- Authentication credentials for private registries must be provided in `%pre` before the pull
- Storage layout differs: `reqpart --add-boot` + `part / --grow` instead of `autopart`
- All software must be in the bootable container image - there is no `%packages` to add extras

---

## 5. Managing Image-mode Systems with `bootc`

### Checking System Status

```bash
# Show staged, booted, and rollback image information
bootc status
```

```
Staged image: registry.example.com:5000/user/myos    <- new image ready to boot
  Digest: sha256:52e3...051a
  Version: 10.0 (2025-06-23 19:14:17 UTC)
● Booted image: registry.example.com:5000/user/myos   <- currently running
  Digest: sha256:1270...15ed
  Version: 10.0 (2025-06-23 18:49:52 UTC)
Rollback image: registry.example.com:5000/user/myos  <- can roll back to this
  Digest: sha256:87b0...27bc
```

| State | Meaning |
|---|---|
| **Staged** | New image downloaded and ready - boots on next reboot |
| **Booted** | Currently running image (marked with `●`) |
| **Rollback** | Previous image - can boot into it with `bootc rollback` |

### Updating a System

The standard update workflow after pushing a new image to the registry:

```bash
# Step 1: Stage the update (downloads new layers, does NOT change running system)
bootc upgrade

# Check it was staged (no service impact yet)
bootc status

# Step 2: Reboot to apply the staged update
reboot

# Step 3: Verify the new image is now booted
bootc status

# Option: stage AND reboot automatically in one command
bootc upgrade --apply
```

```bash
# Check if an update is available without applying it
bootc upgrade --check

# bootc upgrade and bootc update are aliases (same command)
bootc update
```

### Automatic Updates

By default, image-mode systems automatically check for and apply updates:

```bash
# Check the automatic update timer status
systemctl status bootc-fetch-apply-updates.timer

# Disable automatic updates (for controlled deployments)
systemctl mask bootc-fetch-apply-updates.timer

# The automatic update process:
# bootc-fetch-apply-updates.timer triggers regularly
# -> checks registry for a new image digest
# -> if found, downloads layers and stages
# -> system reboots at next maintenance window (or manually)
```

### Rolling Back a System

```bash
# Roll back to the previous deployment (queues for next boot)
bootc rollback

# Then reboot to apply the rollback
reboot

# After reboot:
# - Previous image is now booted
# - Rolled-back image becomes the new rollback entry
# - Any staged update is discarded
```

### Saving Registry Credentials for `bootc`

```bash
# Save credentials to the system auth file so bootc can pull without prompting
podman login REGISTRY --authfile=/etc/ostree/auth.json
```

---

## 6. Image Mode Filesystem Layout

The filesystem is similar to a traditional RHEL installation, but directories behave differently.

```
/                   <- IMMUTABLE (composefs overlay, read-only)
├── usr/            <- Read-only (OS binaries, libraries)
├── etc/            <- MUTABLE + PERSISTENT (3-way merge on upgrade)
├── var/            <- MUTABLE + PERSISTENT (shared across deployments, NOT rolled back)
├── home/           <- MUTABLE + PERSISTENT (user data)
├── tmp/            <- MUTABLE (tmpfs, cleared on reboot)
└── ostree/repo/    <- OSTree deployment repository (multiple versions stored here)
    ├── deployment-1/  (currently booted)
    └── deployment-2/  (previous / rollback)
```

### The Three Key Directories

| Directory | Behaviour | Upgraded? | Rolled back? |
|---|---|---|---|
| `/usr` | Immutable - read-only | Yes (new image replaces it) | Yes |
| `/etc` | Mutable - 3-way merge | Yes (your changes + image changes merged) | Yes |
| `/var` | Mutable - shared between deployments | **No** (your changes survive upgrades) | **No** |

### The `/etc` Three-Way Merge Explained

```
Old image /etc/sshd_config  (version A)
New image /etc/sshd_config  (version B - different from A)
Your local /etc/sshd_config (version C - you modified from A)

bootc merge result:
  - If you didn't change it:  new image version (B) is used
  - If you changed it:        your version (C) is kept
  - If both changed:          conflict flagged for review
```

> This is conceptually identical to `git merge`. Your local `/etc` changes survive OS upgrades.

### The `/var` Directory: Important Limitations

```bash
# /var contents are set from the image at FIRST INSTALL ONLY
# Subsequent upgrades and rollbacks do NOT change /var

# This means:
# - Database files in /var/lib/mysql survive upgrades (good!)
# - If you write to /var/www/html in a Containerfile, it only appears at first install
# - bootc rollback does NOT revert /var/lib/mysql schema changes
```

> You cannot push changes to `/var` content via image upgrades. Only `/etc` and `/usr` are updated. To update `/var` content you must do it directly on the running system.

### Verifying the Immutable Root

```bash
# The root filesystem shows composefs as the filesystem type
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# composefs       8.4M  8.4M    0  100% /       <- 100% is normal! It's an overlay
# /dev/sda3       9.0G  1.9G  7.2G  21% /sysroot <- actual storage

# Attempting to write to /usr fails (read-only)
touch /usr/testfile
# touch: cannot touch '/usr/testfile': Read-only file system
```

---

## 7. The `bootc upgrade` Output Explained

```
bootc upgrade output:

layers already present: 69    <- unchanged layers (not re-downloaded)
layers needed: 2 (34.5 MB)   <- only the changed layers are fetched

Fetching layers  2/2
Fetched layers: 32.95 MiB in 36 seconds

Queued for next boot: registry.example.com/user/myos
Version: 10.20250116.0
Digest: sha256:dc6a...27bc

Total new layers: 71    Size: 739.4 MB  <- complete new OS size
Removed layers:  2      Size: 26.9 MB   <- layers no longer needed
Added layers:    2      Size: 34.5 MB   <- new layers fetched
```

> **Only changed layers are downloaded.** Like `dnf update` only downloads changed packages, `bootc upgrade` only downloads image layers that changed. Updating one package in the Containerfile downloads only that layer, not the entire OS image.

---

## Complete Workflow: Updating an Image Mode System

```bash
# On the BUILD workstation:
# 1. Edit the Containerfile
vim Containerfile   # e.g. add: RUN dnf -y install vim-enhanced mod_ssl && dnf clean all

# 2. Rebuild the image
podman build --squash -t registry.example.com:5000/user/myos:latest .

# 3. Push to registry
podman push registry.example.com:5000/user/myos:latest

# On each TARGET server:
# 4. Stage the update (downloads only changed layers)
bootc upgrade

# 5. Verify what is staged
bootc status

# 6. Reboot to apply
reboot

# 7. After reboot - verify the new image is running
bootc status
rpm -q vim-enhanced mod_ssl   # verify the new packages are installed

# If something went wrong:
# 8. Roll back to previous version
bootc rollback
reboot
```

---

## Quick Reference: All Commands

```bash
# --- Build ---
podman build --squash -t REGISTRY/IMAGE:TAG .    # Build bootable image
podman push REGISTRY/IMAGE:TAG                    # Push to registry

# --- bootc (on deployed systems, requires root) ---
bootc status                    # Show staged/booted/rollback state
bootc upgrade                   # Stage update from registry (no reboot yet)
bootc upgrade --check           # Check for updates without applying
bootc upgrade --apply           # Stage AND reboot automatically
bootc rollback                  # Queue rollback for next boot
bootc update                    # Alias for bootc upgrade

# --- Automatic updates ---
systemctl status bootc-fetch-apply-updates.timer  # Check auto-update timer
systemctl mask bootc-fetch-apply-updates.timer    # Disable auto-updates

# --- Registry auth for bootc ---
podman login REGISTRY --authfile=/etc/ostree/auth.json

# --- Verify immutable root ---
df -h                           # Shows composefs at /
mount | grep " / "              # Shows ro on root
```

---

## Key Files and Paths

| Path | Purpose |
|---|---|
| `/ostree/repo/` | OSTree repository storing all available deployments |
| `/etc/` | Mutable, persistent, 3-way merged on upgrade |
| `/var/` | Mutable, persistent, shared across deployments, not rolled back |
| `/usr/` | Immutable OS content (read-only via composefs) |
| `/etc/ostree/auth.json` | Registry credentials for `bootc` automatic updates |
| `/usr/lib/systemd/system/bootc-fetch-apply-updates.timer` | Automatic update timer |

---

## Things to Remember

1. **Image mode replaces `dnf update` with a build-push-upgrade-reboot cycle.** You never install packages directly on a running image-mode system. All changes go through the Containerfile and the registry.

2. **The base image for image mode is `rhel-bootc`, not `ubi10/ubi`.** Using the wrong base creates an application container, not a bootable OS image.

3. **Use `systemctl enable` instead of `ENTRYPOINT` for services in bootable images.** In image mode, systemd is PID 1. `ENTRYPOINT` and `CMD` are ignored.

4. **A reboot is always required to apply an image mode update.** `bootc upgrade` stages the update but the running system is unaffected until reboot. There is no hot-apply for OS updates.

5. **Replace `%packages` with `ostreecontainer --url=` in Kickstart for image mode.** Do not use both. The `ostreecontainer` directive tells Anaconda to pull the bootable container image from the registry.

6. **`/var` is NOT upgraded and NOT rolled back.** Application data in `/var/lib/` survives both upgrades and rollbacks. Writing to `/var` in a Containerfile only affects the initial installation.

7. **`/etc` is merged during upgrades, not overwritten.** Your local configuration changes survive OS upgrades. The merge is like `git merge`: your changes + image changes, with conflict flagging.

8. **`bootc status` shows three states: staged, booted, and rollback.** Staged = downloaded, ready for next boot. Booted = currently running. Rollback = previous version, can be restored with `bootc rollback`.

9. **The composefs root filesystem shows 100% full in `df -h` - this is normal.** It is a read-only overlay. The actual storage usage is shown on `/sysroot`. The 100% figure is expected and not a disk space problem.

10. **`bootc rollback` does not revert `/var`.** If an upgrade migrated a database schema or wrote files to `/var`, rollback restores the old OS binaries but the `/var` changes remain. Plan schema changes carefully in image-mode database environments.
