# RH134 Chapter 10 - Managing Basic Storage

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Manage storage devices by creating partitions, file systems, and swap spaces from the command line.

---

## Windows vs Linux: Storage Management Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Disk Management (diskmgmt.msc) | `parted`, `lsblk` |
| Disk initialise (MBR / GPT) | `parted /dev/sdb mklabel msdos\|gpt` |
| New Simple Volume Wizard | `parted mkpart` + `mkfs.xfs` + `mount` |
| Format Volume (NTFS/exFAT) | `mkfs.xfs` / `mkfs.ext4` |
| Drive letter (C:, D:) | Mount point directory (`/data`, `/backup`) |
| Volume label | XFS label (`-L NAME` in `mkfs.xfs`) |
| "Always mount this drive" | Entry in `/etc/fstab` |
| Page file | Swap space |
| `diskmgmt` > New Simple Volume | `parted mkpart` + `mkfs.xfs` + `fstab` entry |
| Disk Signature (MBR) / GUID (GPT) | Partition UUID (stored in filesystem superblock) |
| `diskpart` (CLI) | `parted` (CLI) |

---

## Chapter Overview: The Storage Stack

```
Physical disk (/dev/sdb)
        |
        v
Partition table (GPT or MBR)
        |
   +---------+----------+
   |         |          |
Partition1  Partition2  Partition3
(/dev/sdb1) (/dev/sdb2) (/dev/sdb3)
        |         |           |
   Filesystem  Filesystem   Swap space
    (XFS)      (ext4)     (mkswap)
        |
   Mount point
   (/data)
        |
   /etc/fstab entry
   (persistent)
```

---

## Part 1: Partition Schemes

### MBR vs GPT

| Feature | MBR (msdos) | GPT |
|---|---|---|
| Firmware required | BIOS | UEFI (preferred) |
| Max partitions | 4 primary (15 with extended) | 128 |
| Max disk size | 2 TiB | 8 ZiB |
| Partition names | No (type only: primary/extended/logical) | Yes (each partition has a name) |
| Redundancy | Single partition table | Primary + backup copy |
| Error detection | None | CRC32 checksum |
| Use today | Legacy hardware | All modern systems |

> **RHEL 10 default:** GPT on UEFI systems. MBR (`msdos`) for BIOS compatibility only.

---

## Part 2: Partitioning with `parted`

### Critical Safety Rule

> **`parted` writes changes to disk immediately.** Unlike `fdisk`, there is no staging area and no "write" command to confirm. Every subcommand takes effect the moment you press Enter. `mklabel` wipes the existing partition table instantly. Always double-check the device name before running any `parted` command.

### Viewing Disks and Partitions

```bash
# List all block devices and their partitions
lsblk

# List with filesystem info (UUID, type, label)
lsblk --fs
lsblk -f          # same

# View partition table on a specific disk
parted /dev/sdb print

# View partition table in sectors
parted /dev/sdb unit s print

# View all disks
parted -l
```

### Writing a Partition Table Label (Initialising a Disk)

```bash
# Write MBR (BIOS / legacy)
parted /dev/sdb mklabel msdos

# Write GPT (UEFI / modern)
parted /dev/sdb mklabel gpt
```

> **Warning:** `mklabel` immediately wipes the existing partition table. All existing partitions and their data become inaccessible.

### Creating Partitions - Non-Interactive (Recommended)

```bash
# MBR: create a primary XFS partition from 2048s to 1000MB
parted /dev/sdb mkpart primary xfs 2048s 1000MB

# GPT: create a named XFS partition
parted /dev/sdb mkpart data xfs 2048s 1000MB

# GPT: create a named swap partition
parted /dev/sdb mkpart swap1 linux-swap 1000MB 1512MB

# Always run udevadm settle after creating a partition
udevadm settle
```

### Creating Partitions - Interactive Mode

```bash
parted /dev/sdb
```

```
# Inside the parted prompt:

print              # Show current partition table
help               # Show available commands
help mkpart        # Show mkpart syntax

mkpart             # Start interactive partition creation
  Partition type? primary/extended? primary     # MBR only
  # OR
  Partition name? []? data                      # GPT only
  File system type? [ext2]? xfs
  Start? 2048s
  End? 1000MB

rm 1               # Delete partition number 1
quit               # Exit parted
```

### Partition Start Position

```bash
# Always start at sector 2048 for correct alignment
# 2048 sectors * 512 bytes = 1,048,576 bytes = 1 MiB alignment
# This aligns with modern disk physical block boundaries

# After first partition, use the END of the previous partition as the new START
# Example: partition 1 ends at 1000MB -> partition 2 starts at 1000MB
parted /dev/sdb mkpart swap1 linux-swap 1000MB 1512MB
parted /dev/sdb mkpart swap2 linux-swap 1512MB 2024MB
```

### The FS-TYPE Label vs the Actual Filesystem

```
parted mkpart ... xfs ...     <- Sets a LABEL in partition table only
                                 Does NOT create a filesystem!

mkfs.xfs /dev/sdb1            <- Creates the actual XFS filesystem
```

> The `xfs` or `linux-swap` in `mkpart` is metadata that tells tools what the partition is intended for. No actual filesystem exists until `mkfs` is run.

### Deleting Partitions

```bash
# Non-interactive
parted /dev/sdb rm 1          # Delete partition number 1

# Interactive
parted /dev/sdb
(parted) print                # Find the partition number
(parted) rm 1
(parted) quit
```

---

## Part 3: Creating Filesystems

### XFS (RHEL Default)

```bash
# Format a partition with XFS
mkfs.xfs /dev/sdb1

# Format with a label (label can be referenced in fstab)
mkfs.xfs -L mydata /dev/sdb1

# Force overwrite of existing filesystem (use with care!)
mkfs.xfs -f /dev/sdb1

# Verify the filesystem was created
lsblk --fs /dev/sdb1
```

### ext4 (Alternative)

```bash
# Format with ext4
mkfs.ext4 /dev/sdb1
mkfs.ext4 -L mydata /dev/sdb1
```

### Common mkfs Variants

| Command | Filesystem |
|---|---|
| `mkfs.xfs` | XFS (RHEL default, recommended) |
| `mkfs.ext4` | ext4 |
| `mkfs.vfat` | FAT32 (USB drives, EFI partitions) |
| `mkfs -t xfs` | Equivalent to `mkfs.xfs` |

---

## Part 4: Mounting Filesystems

### Temporary Mount (Lost on Reboot)

```bash
# Create the mount point directory first
mkdir /data

# Mount the partition
mount /dev/sdb1 /data

# Mount with specific options
mount -o ro /dev/sdb1 /data       # Read-only
mount -o noexec /dev/sdb1 /data   # Prevent execution of files

# Unmount
umount /data
umount /dev/sdb1       # Can specify either mount point or device

# View all currently mounted filesystems
mount
mount | grep sdb
findmnt
```

### Persistent Mount via `/etc/fstab`

The `/etc/fstab` file is read at boot. Each line mounts one device.

#### `/etc/fstab` Field Format

```
DEVICE          MOUNTPOINT   FSTYPE   OPTIONS    DUMP  FSCK
UUID=...        /data        xfs      defaults   0     0
/dev/sdb1       /data        xfs      defaults   0     0
LABEL=mydata    /data        xfs      defaults   0     0
```

| Field | Position | Description |
|---|---|---|
| Device | 1 | UUID, device path, or LABEL |
| Mount point | 2 | Directory path (use `swap` for swap entries) |
| FS type | 3 | `xfs`, `ext4`, `swap`, `nfs`, etc. |
| Options | 4 | Mount options (use `defaults` to start) |
| Dump | 5 | `0` = no backup with dump (almost always 0) |
| fsck order | 6 | `0` = no fsck, `1` = root partition, `2` = others |

#### Common Mount Options

| Option | Meaning |
|---|---|
| `defaults` | Standard options: rw, suid, dev, exec, auto, nouser, async |
| `ro` | Read-only |
| `rw` | Read-write (included in defaults) |
| `noexec` | Prevent execution of binaries on this mount |
| `nosuid` | Ignore setuid/setgid bits |
| `noatime` | Do not update file access timestamps (performance) |
| `auto` | Mount automatically at boot (included in defaults) |
| `nofail` | Do not fail boot if this device is absent |
| `x-systemd.automount` | Mount only when first accessed |

#### A Typical `/etc/fstab` Entry

```bash
# Get the UUID first
lsblk --fs /dev/sdb1
# OR
blkid /dev/sdb1

# Add the entry to /etc/fstab
UUID=8a90ce0d-4a6f-4b12-9abc-1234567890ab  /data  xfs  defaults  0  0
```

### After Editing `/etc/fstab`

```bash
# ALWAYS verify the file before rebooting
findmnt --verify

# Test: mount all entries in fstab (catches most errors)
mount -a

# Tell systemd to reload fstab
systemctl daemon-reload
```

> **Warning:** A bad `/etc/fstab` entry can make the system fail to boot and drop to an emergency shell. Always run `findmnt --verify` and `mount -a` after every edit. Never reboot without testing first.

### Getting UUIDs

```bash
# From lsblk (preferred - clear format)
lsblk --fs
lsblk --fs /dev/sdb1

# From blkid
blkid
blkid /dev/sdb1

# From /dev/disk/by-uuid/ (symlinks to devices)
ls -l /dev/disk/by-uuid/
```

> **Always use UUIDs in `/etc/fstab`, not device names.** Device names like `/dev/sdb1` can change between reboots (especially on VMs) if disks are detected in a different order. UUIDs are permanent and stored in the filesystem superblock.

---

## Part 5: Managing Swap Space

### What Is Swap?

Swap is disk space used as overflow when physical RAM is fully utilised. The kernel moves inactive memory pages to swap, freeing RAM for active processes. Swap is significantly slower than RAM but prevents out-of-memory crashes.

### Recommended Swap Sizes (Red Hat Guidelines)

| System RAM | Recommended Swap | With Hibernation |
|---|---|---|
| Less than 2 GB | 2x RAM | 3x RAM |
| 2-8 GB | Equal to RAM | 2x RAM |
| 8-64 GB | At least 4 GB | 1.5x RAM |
| More than 64 GB | Workload-dependent (min 4 GB) | Hibernation impractical |

### Creating Swap Space

```bash
# Step 1: Create a partition with linux-swap type label
parted /dev/sdb mkpart swap1 linux-swap 1000MB 1512MB
udevadm settle

# Step 2: Initialise the swap space
mkswap /dev/sdb2
# Output shows UUID - note it for fstab
# Setting up swapspace version 1, size = 476 MiB
# no label, UUID=39e2667a-9458-42fe-9665-c5c854605881

# Step 3: Activate temporarily (until reboot)
swapon /dev/sdb2

# Verify it is active
swapon --show
free -h

# Deactivate
swapoff /dev/sdb2
```

### Making Swap Persistent in `/etc/fstab`

```bash
# Swap fstab entry format:
UUID=39e2667a-9458-42fe-9665-c5c854605881  swap  swap  defaults  0  0

# With priority (higher number = preferred first):
UUID=39e2667a-...  swap  swap  pri=10  0  0
UUID=54a08491-...  swap  swap  pri=20  0  0
# (swap2 with pri=20 is used first, swap1 with pri=10 used when swap2 is full)

# After editing fstab:
systemctl daemon-reload

# Activate all swap spaces listed in fstab
swapon -a

# Verify
swapon --show
```

### Swap Priority Behaviour

| Priority | Behaviour |
|---|---|
| Higher number (e.g. `pri=20`) | Used first (exhausted before lower priority) |
| Lower number (e.g. `pri=10`) | Used after higher priority is full |
| Equal priority | Kernel stripes across all (round-robin - good for performance) |
| Default (`pri=-2`) | Added last in the order, lowest priority |

```
Swap available: sdb2 pri=20, sdb3 pri=10
Kernel fills sdb2 first -> when sdb2 is full -> starts using sdb3
```

### Viewing Swap

```bash
swapon --show         # Detailed view with priorities
free -h               # Summary of RAM + swap
cat /proc/swaps       # Raw swap info
```

---

## Quick Reference: Complete Workflow

### Workflow 1: New partition + XFS filesystem + persistent mount

```bash
# 1. Initialise disk (if new)
parted /dev/sdb mklabel gpt

# 2. Create partition
parted /dev/sdb mkpart data xfs 2048s 10GB

# 3. Wait for device file
udevadm settle

# 4. Verify partition was created
lsblk
parted /dev/sdb print

# 5. Format with XFS
mkfs.xfs /dev/sdb1

# 6. Get UUID
lsblk --fs /dev/sdb1

# 7. Create mount point
mkdir /data

# 8. Add to /etc/fstab (use UUID from step 6)
echo "UUID=PASTE-UUID-HERE  /data  xfs  defaults  0  0" >> /etc/fstab

# 9. Verify fstab syntax
findmnt --verify

# 10. Test mount
mount -a

# 11. Reload systemd
systemctl daemon-reload

# 12. Verify mounted
mount | grep /data
df -h /data
```

### Workflow 2: Swap partition

```bash
# 1. Create swap partition
parted /dev/sdb mkpart swap1 linux-swap 10GB 10.5GB
udevadm settle

# 2. Initialise as swap (note the UUID in output)
mkswap /dev/sdb2

# 3. Get UUID
lsblk --fs /dev/sdb2

# 4. Add to /etc/fstab
echo "UUID=PASTE-UUID-HERE  swap  swap  pri=10  0  0" >> /etc/fstab

# 5. Reload systemd
systemctl daemon-reload

# 6. Activate all fstab swap entries
swapon -a

# 7. Verify
swapon --show
```

---

## All Commands at a Glance

```bash
# --- Inspection ---
lsblk                          # List block devices
lsblk --fs                     # Include UUID, fstype, label
lsblk --fs /dev/sdb1           # Single device detail
blkid                          # Show UUIDs and types
blkid /dev/sdb1                # Single device
findmnt                        # Show mount tree
findmnt --verify               # Verify /etc/fstab
parted /dev/sdb print          # Show partition table
parted -l                      # Show all disks

# --- Partitioning ---
parted /dev/sdb mklabel gpt          # Initialise GPT
parted /dev/sdb mklabel msdos        # Initialise MBR
parted /dev/sdb mkpart NAME TYPE START END   # Create partition (GPT)
parted /dev/sdb mkpart primary TYPE START END # Create partition (MBR)
parted /dev/sdb rm PARTNUM           # Delete partition
udevadm settle                       # Wait for device files

# --- Filesystems ---
mkfs.xfs /dev/sdb1             # Format XFS
mkfs.xfs -L LABEL /dev/sdb1   # Format XFS with label
mkfs.ext4 /dev/sdb1            # Format ext4

# --- Mounting ---
mkdir /mountpoint              # Create mount point first
mount /dev/sdb1 /mountpoint    # Temporary mount
mount -a                       # Mount all fstab entries
umount /mountpoint             # Unmount
df -h                          # Show disk usage

# --- Swap ---
mkswap /dev/sdb2               # Initialise swap
swapon /dev/sdb2               # Activate swap (temporary)
swapon -a                      # Activate all fstab swap entries
swapoff /dev/sdb2              # Deactivate swap
swapon --show                  # Show active swap
free -h                        # Show RAM + swap summary

# --- After fstab changes ---
findmnt --verify               # Check syntax
mount -a                       # Test all entries
systemctl daemon-reload        # Tell systemd about changes
```

---

## Key Files

| Path | Purpose |
|---|---|
| `/etc/fstab` | Persistent mount and swap configuration |
| `/dev/sda`, `/dev/sdb` ... | Whole disk block device files |
| `/dev/sda1`, `/dev/sdb1` ... | Partition block device files |
| `/dev/disk/by-uuid/` | Symlinks to devices by UUID (reliable identifiers) |
| `/proc/swaps` | Currently active swap spaces |

---

## Things to Remember

1. **`parted` writes changes immediately - there is no undo.** Every command takes effect the instant you press Enter. `mklabel` wipes the partition table instantly. Always double-check the device name before running any `parted` command.

2. **The FS-TYPE in `parted mkpart` is a label, not a filesystem.** Specifying `xfs` in `mkpart` does NOT create an XFS filesystem. You must run `mkfs.xfs` separately after partitioning.

3. **Always run `udevadm settle` after creating a partition.** Without it, the `/dev/sdbN` device file may not exist yet when you try to run `mkfs`. `settle` blocks until `udev` has registered the new partition.

4. **Always use UUIDs in `/etc/fstab`, not device names.** Device names (`/dev/sdb1`) can change between reboots. UUIDs are stored in the filesystem and never change unless you reformat.

5. **Test `/etc/fstab` before rebooting.** A bad entry can prevent the system from booting. Run `findmnt --verify` to check syntax, then `mount -a` to test all entries. Run `systemctl daemon-reload` after any change.

6. **The six `/etc/fstab` fields in order:** Device, Mount point, FS type, Options, Dump (0), fsck order (0 for swap/0 for most, 1 for root, 2 for others).

7. **Swap entries use `swap` as both the mount point field and the filesystem type field.** The line looks like: `UUID=...  swap  swap  defaults  0  0`.

8. **Higher swap priority number = used first.** `pri=20` is preferred over `pri=10`. When two swaps have equal priority, the kernel stripes across them in round-robin.

9. **`mkswap` replaces the filesystem on a partition.** If you accidentally run `mkswap` on a data partition, the filesystem is gone. There is no recovery without a backup. Confirm the device name before running.

10. **`mount -a` does not unmount anything.** It only mounts entries that are not already mounted. It is safe to run on a live system to activate new `/etc/fstab` entries without affecting existing mounts.
