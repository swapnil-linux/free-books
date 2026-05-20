# RH134 Chapter 12 - Controlling and Troubleshooting the Boot Process

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Manage how the system boots to control which services start, and troubleshoot and repair boot-time problems.

---

## Windows vs Linux: Boot Concepts Equivalents

| Windows Concept | Linux / RHEL Equivalent |
|---|---|
| BIOS / UEFI firmware | BIOS / UEFI firmware (same) |
| Windows Boot Manager | GRUB2 |
| `bcdedit` (boot config) | `grubby` command |
| `boot.ini` / BCD store | `/boot/loader/entries/*.conf` (UEFI) |
| Safe Mode (F8) | `rescue.target` (append to kernel line) |
| Safe Mode - Command Prompt | `emergency.target` |
| Normal boot | `graphical.target` or `multi-user.target` |
| Server Core (no GUI) | `multi-user.target` |
| Windows RE (WinPE recovery) | Boot from RHEL install media, rescue mode |
| `chkdsk /f` | `xfs_repair DEVICE` or `fsck.ext4 DEVICE` |

---

## 1. The RHEL 10 Boot Sequence

```
Power on
    |
    v
FIRMWARE (BIOS or UEFI)
  - BIOS: reads MBR from disk, loads GRUB2 core.img
  - UEFI: reads NVRAM, loads GRUB2 EFI app from /boot/efi/
    |
    v
GRUB2 BOOT LOADER
  - Presents boot menu (kernel selection)
  - Loads kernel (vmlinuz) + initramfs into RAM
  - Passes kernel command-line arguments
  - Hands control to the kernel
    |
    v
KERNEL (boots inside initramfs)
  - Initialises hardware using initramfs drivers
  - Starts /sbin/init from initramfs -> PID 1 (systemd)
  - Runs initrd.target units
  - Mounts real root filesystem at /sysroot
  - Pivots root from initramfs to /sysroot
  - Re-executes systemd from disk
    |
    v
SYSTEMD (from real disk)
  - Reads default.target
  - Starts units to reach that target
  - Resolves dependencies automatically
    |
    v
LOGIN PROMPT
  (text login for multi-user.target)
  (graphical login for graphical.target)
```

> **initramfs in RHEL 10:** The initramfs is a complete bootable Linux system in RAM. It contains drivers, scripts, and a running systemd unit - enough to set up LVM, LUKS, or RAID before the real root filesystem is mounted. Inspect it with `lsinitrd`.

---

## 2. GRUB2 Boot Loader

### Key Locations

| Path | Purpose |
|---|---|
| `/boot/grub2/grub.cfg` | Generated config - do NOT edit directly |
| `/boot/loader/entries/*.conf` | Per-kernel boot entries (UEFI/BLS format) |
| `/boot/efi/` | EFI system partition (UEFI only) |
| `/etc/default/grub` | User-editable settings (used to regenerate grub.cfg) |

> **Never edit `/boot/grub2/grub.cfg` directly.** It is auto-generated and will be overwritten by kernel updates. Use `grubby` to make persistent changes.

### Using the GRUB2 Menu at Boot Time (One-time Changes)

```
At the GRUB2 menu:
  Any key (not Enter)    Interrupt the countdown
  E                      Edit the currently selected entry
  Ctrl+X                 Boot with edited entry (changes are NOT persistent)
  Esc                    Discard edits, return to menu
```

On the kernel command line (the line starting with `linux`):
- Append options at the end of the line
- Changes made here apply to this boot only

### The `grubby` Command (Persistent Changes)

```bash
# List all boot entries with their index numbers
grubby --info=ALL

# View details of a specific entry by index
grubby --info 0
grubby --info 1

# Show the current default entry
grubby --default-index
grubby --default-kernel

# Change the default boot entry (by index)
grubby --set-default-index 0

# Change default by kernel path
grubby --set-default /boot/vmlinuz-6.12.0-55.12.1.el10_0.x86_64

# Add kernel arguments to a specific kernel
grubby --update-kernel /boot/vmlinuz-KERNELVERSION --args="rhgb quiet"

# Add argument to ALL kernels
grubby --update-kernel=ALL --args="console=ttyS0"

# Remove kernel arguments
grubby --update-kernel /boot/vmlinuz-KERNELVERSION --remove-args="rhgb quiet"

# Remove argument from ALL kernels
grubby --update-kernel=ALL --remove-args="rhgb quiet"
```

### Useful Kernel Command-Line Arguments

| Argument | Effect |
|---|---|
| `rhgb` | Red Hat Graphical Boot (splash screen) |
| `quiet` | Suppress most kernel boot messages |
| `systemd.unit=rescue.target` | Boot into rescue shell |
| `systemd.unit=emergency.target` | Boot into emergency shell |
| `selinux=0` | Disable SELinux for this boot (RHEL 10) |
| `rd.break` | Break into initramfs shell (used for root password reset - Chapter 13) |
| `console=ttyS0` | Send console output to serial port |
| `video=640x480` | Set console resolution (helpful in GRUB editor) |

```bash
# Verify current kernel arguments after changes
grubby --info $(grubby --default-kernel)
```

---

## 3. systemd Targets

Targets are groups of systemd units that define a system state. They are the replacement for SysV runlevels.

### Key Targets

| Target | SysV Equivalent | Description |
|---|---|---|
| `emergency.target` | Single-user (minimal) | Root filesystem read-only, almost nothing started. Minimal recovery environment. |
| `rescue.target` | Runlevel 1 | Single-user with most hardware and filesystems initialised. Read/write root. |
| `multi-user.target` | Runlevel 3 | Full multi-user, text login only. Normal server mode. |
| `graphical.target` | Runlevel 5 | Full multi-user with graphical login. |
| `reboot.target` | Runlevel 6 | System reboot. |
| `poweroff.target` | Runlevel 0 | System halt and power off. |

### Emergency vs Rescue - The Key Differences

| Feature | `emergency.target` | `rescue.target` |
|---|---|---|
| Root filesystem | Read-only (`ro`) | Read/write (`rw`) |
| Other filesystems | Not mounted | Mounted (as per fstab) |
| Services started | Minimal (almost none) | Most system services |
| Logging | No | Yes |
| Use when | fstab errors, critical boot failure | Configuration issues, service failures |

> **Practical rule:** Try `rescue.target` first. Use `emergency.target` when rescue also fails, or specifically for `/etc/fstab` repair (because root is read-only, no mounts have been attempted yet).

### Viewing and Changing Targets

```bash
# Show the current default boot target
systemctl get-default

# Change the default boot target (persistent)
systemctl set-default multi-user.target
systemctl set-default graphical.target

# Switch to a different target NOW (without rebooting)
systemctl isolate multi-user.target
systemctl isolate graphical.target

# Note: not all targets support isolate
# Only targets with AllowIsolate=yes can be isolated
```

### Booting to a Specific Target (One-time, at GRUB)

```
1. Reboot the system
2. At GRUB2 menu, press any key (not Enter) to interrupt countdown
3. Press E to edit the current entry
4. Navigate to the line starting with 'linux'
5. Press Ctrl+E or End to move to the end of that line
6. Append:  systemd.unit=rescue.target
       or:  systemd.unit=emergency.target
7. Press Ctrl+X to boot with this configuration
```

### Power Control Commands

```bash
systemctl reboot        # Graceful reboot (stops services, unmounts filesystems)
systemctl poweroff      # Graceful shutdown and power off
systemctl halt          # Halt (does not power off - brings to safe state)
reboot                  # Shortcut for systemctl reboot
poweroff                # Shortcut for systemctl poweroff
```

---

## 4. Repairing Filesystem Issues at Boot

### Why the Boot Process Stops

When systemd cannot mount a filesystem listed in `/etc/fstab`, it times out and drops to an emergency shell:

```
[ TIME ] Timed out waiting for device /dev/sda2.
[DEPEND] Dependency failed for /mnt/data
[DEPEND] Dependency failed for Local File Systems.
[ OK  ] Started Emergency Shell.
Give root password for maintenance
(or press Control-D to continue):
```

**Common causes:**

| Cause | Symptom | Fix |
|---|---|---|
| Wrong UUID in fstab | Device not found, timeout | Correct UUID in `/etc/fstab` |
| Nonexistent mount point directory | Mount point does not exist | Create the directory |
| Typo in device path | Device not found | Fix the device path |
| Corrupted filesystem | fsck fails at boot | `xfs_repair` or `fsck.ext4` |

### The Emergency Shell Recovery Workflow

```bash
# Step 1: Enter the emergency shell, log in as root with root password

# Step 2: Check current mount status
mount
# Look for root filesystem - it will show 'ro' (read-only)
# e.g: /dev/sda3 on / type xfs (ro,relatime,...)

# Step 3: Remount root filesystem read/write (CRITICAL STEP)
mount -o remount,rw /

# Step 4: Try to mount all fstab entries to identify the problem
mount -a
# OR:
mount --all
# The error message will identify which entry is failing

# Step 5: Fix the problem
vim /etc/fstab           # Correct wrong UUID, path, or mount point
# OR
mkdir /missing/mountpoint  # Create missing mount point directory

# Step 6: Tell systemd to reload fstab
systemctl daemon-reload

# Step 7: Test the fix
mount -a
# Should now complete without errors

# Step 8: Reboot to verify
systemctl reboot
```

### Getting the Correct UUID After Fstab Corruption

```bash
# In the emergency shell, find the right UUID
lsblk --fs              # Shows UUID for all partitions
blkid                   # Alternative UUID listing
ls -l /dev/disk/by-uuid/  # Symlinks from UUID to device
```

### Repairing Corrupted XFS Filesystems

```bash
# xfs_repair must run on an UNMOUNTED filesystem
# If the damaged filesystem is NOT root, unmount it first:
umount /dev/sdb1

# Run repair
xfs_repair /dev/sdb1

# If the filesystem has a dirty log (was not cleanly unmounted):
# xfs_repair may ask you to mount/unmount first to replay the log
# Mount it first to replay the journal:
mount /dev/sdb1 /mnt/temp
umount /dev/sdb1
# Then run xfs_repair again:
xfs_repair /dev/sdb1
```

> **`fsck.xfs` is a decoy.** It exists to satisfy the boot framework's `fsck.TYPE` convention but exits immediately without doing anything. The real XFS repair tool is `xfs_repair DEVICE`.

### Repairing Corrupted ext4 Filesystems

```bash
# Must also be unmounted
umount /dev/sdb1

# Repair with auto-fix for minor issues
fsck.ext4 -p /dev/sdb1

# Interactive repair
fsck.ext4 /dev/sdb1

# e2fsck is the same tool (fsck.ext4 is a hard link to e2fsck)
e2fsck -p /dev/sdb1
```

### The `nofail` Option - Testing Without Risk

Adding `nofail` to an `/etc/fstab` entry allows the system to boot even if that entry fails to mount:

```bash
UUID=...   /data   xfs   defaults,nofail   0   0
```

> **Use `nofail` for testing only, not production filesystems.** If `/data` contains application data and it fails to mount, the application may start against a missing filesystem with unpredictable consequences. Use `nofail` only for optional mounts (removable media, non-critical NFS).

---

## 5. The `mount -o remount,rw /` Command Explained

In the emergency target, the root filesystem is deliberately mounted read-only to prevent accidental damage during recovery. This means:

```bash
# Trying to edit files fails
vim /etc/fstab
# Error: E45: 'readonly' option is set (add ! to override)

# Fix: remount root read/write WITHOUT unmounting
mount -o remount,rw /

# Verify it worked
mount | grep " on / "
# /dev/sda3 on / type xfs (rw,relatime,...) <- rw now appears

# Now you can edit fstab
vim /etc/fstab
```

> `remount` changes the mount options on a live, already-mounted filesystem. It does not unmount and remount - it modifies the mount flags in place. This is the only way to make root writable in an emergency shell.

---

## Quick Reference: All Commands

```bash
# --- GRUB2 ---
grubby --info=ALL                          # List all boot entries
grubby --info 0                            # Info for entry 0
grubby --default-index                     # Show current default index
grubby --set-default-index 0              # Set default entry
grubby --update-kernel=ALL --args="quiet" # Add arg to all kernels
grubby --update-kernel=ALL --remove-args="rhgb quiet"  # Remove args
grubby --info $(grubby --default-kernel)  # Inspect current default

# --- Targets ---
systemctl get-default                      # Show default target
systemctl set-default multi-user.target    # Change default target
systemctl set-default graphical.target     # Set graphical as default
systemctl isolate multi-user.target        # Switch target now (no reboot)
systemctl isolate rescue.target            # Switch to rescue now

# --- Power ---
systemctl reboot                           # Reboot gracefully
systemctl poweroff                         # Power off gracefully

# --- Emergency recovery workflow ---
mount                                      # Check what is mounted (look for ro/rw)
mount -o remount,rw /                      # Make root writable (FIRST STEP)
mount -a                                   # Attempt all fstab mounts
lsblk --fs                                 # Find correct UUIDs
blkid                                      # Alternative UUID listing
vim /etc/fstab                             # Fix the fstab entry
systemctl daemon-reload                    # Reload after fstab edit
mount -a                                   # Test fix
systemctl reboot                           # Reboot to confirm

# --- Filesystem repair ---
xfs_repair /dev/sdb1                       # Repair XFS (unmounted only)
fsck.ext4 -p /dev/sdb1                     # Repair ext4 (unmounted, auto)
fsck.ext4 /dev/sdb1                        # Repair ext4 (interactive)

# --- Initramfs inspection ---
lsinitrd                                   # Inspect current initramfs contents
lsinitrd /boot/initramfs-KERNELVERSION.img # Inspect specific initramfs
```

---

## Key Paths and Files

| Path | Purpose |
|---|---|
| `/boot/grub2/grub.cfg` | Generated GRUB2 config - do not edit |
| `/boot/loader/entries/*.conf` | Per-kernel BLS boot entries (UEFI) |
| `/boot/efi/` | EFI system partition |
| `/etc/default/grub` | User-editable GRUB defaults |
| `/etc/systemd/system/default.target` | Symlink defining the default boot target |
| `/usr/lib/systemd/system/*.target` | Vendor-supplied target unit files |
| `/etc/fstab` | Filesystem mount configuration (source of many boot failures) |

---

## Things to Remember

1. **Never edit `/boot/grub2/grub.cfg` directly.** It is auto-generated. Use `grubby` for persistent changes. GRUB2 menu edits (pressing E) are one-time only and lost on the next boot.

2. **Changes in the GRUB2 editor (pressing E) apply to this boot only.** For persistent changes, use `grubby --update-kernel` from a running system.

3. **`rescue.target` mounts filesystems read/write. `emergency.target` does not.** Use rescue for most troubleshooting. Use emergency when you specifically need to fix fstab before anything tries to mount.

4. **In the emergency shell, root is mounted read-only.** You must run `mount -o remount,rw /` before you can edit any files. This is always the first step.

5. **`mount -a` in the emergency shell shows you exactly which fstab entry is failing.** The error message identifies the problem (nonexistent device, missing mount point, wrong UUID). Run it immediately after remounting root read/write.

6. **`xfs_repair` must run on an unmounted filesystem.** `fsck.xfs` does nothing - it is a placeholder. The real XFS tool is `xfs_repair DEVICE`.

7. **`systemctl daemon-reload` is required after editing `/etc/fstab` in the emergency shell.** Without it, systemd continues using the old (broken) mount unit configuration.

8. **`systemctl isolate TARGET` switches targets without rebooting.** Starting services in the new target and stopping services that conflict. Use it to test target changes before making them persistent.

9. **`systemctl set-default` creates a symlink at `/etc/systemd/system/default.target`.** The symlink points to the target unit file. You can verify it with `ls -l /etc/systemd/system/default.target`.

10. **The initramfs contains a complete bootable system in RAM.** Use `lsinitrd` to inspect its contents. Understand that the kernel boots twice: once from inside initramfs to find the real root, then again from the real disk under systemd's control.
