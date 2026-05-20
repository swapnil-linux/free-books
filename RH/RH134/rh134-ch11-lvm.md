# RH134 Chapter 11 - Managing Storage with Logical Volume Manager

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Use Logical Volume Manager (LVM) to manage logical volumes that contain file systems and swap spaces.

---

## Windows vs Linux: LVM Equivalents

| Windows Concept | Linux / LVM Equivalent |
|---|---|
| Physical disk | Physical Volume (PV) |
| Storage Pool (Storage Spaces) | Volume Group (VG) |
| Virtual Disk (Storage Spaces) | Logical Volume (LV) |
| Dynamic disk (spanned/striped) | LVM LV across multiple PVs |
| Extend volume (diskmgmt.msc) | `lvextend` + `xfs_growfs` |
| New Volume Wizard | `pvcreate` + `vgcreate` + `lvcreate` |
| Disk signature / Volume GUID | LVM UUID (stored in PV metadata) |
| `diskpart extend` | `lvextend -r` |

---

## 1. The LVM Architecture

LVM adds a virtualisation layer between physical storage and filesystems. This enables resizing, spanning multiple disks, and live disk replacement without downtime.

```
┌─────────────────────────────────────────────────┐
│               FILE SYSTEM (XFS, ext4)           │  <-- df, mount, /etc/fstab
├─────────────────────────────────────────────────┤
│           LOGICAL VOLUME (LV)                   │  lvcreate, lvextend, lvremove
│   /dev/vg01/lv01  or  /dev/mapper/vg01-lv01    │
├──────────────────────────────────────────────────┤
│           VOLUME GROUP (VG)                     │  vgcreate, vgextend, vgreduce
│   Pool of physical extents from all PVs         │
├──────────┬──────────┬───────────────────────────┤
│  PV      │  PV      │  PV  (Physical Volumes)   │  pvcreate, pvmove, pvremove
│ /dev/sdb1│ /dev/sdb2│ /dev/sdc                  │
├──────────┴──────────┴───────────────────────────┤
│     Physical Disks or Partitions                │  parted, lsblk
│     /dev/sdb, /dev/sdc, whole disks OK too      │
└─────────────────────────────────────────────────┘
```

### Physical Extents (PE)

LVM divides PVs into equal-sized chunks called Physical Extents (PEs). The default PE size is 4 MiB. An LV is a collection of PEs drawn from the VG. When you specify `-L 300M`, LVM calculates how many PEs are needed (75 PEs × 4 MiB = 300 MiB).

---

## 2. Inspecting LVM Components

### Quick Summary View (one line per item)

```bash
pvs          # All physical volumes
vgs          # All volume groups
lvs          # All logical volumes
```

### Detailed View

```bash
pvdisplay              # All PVs (detailed)
pvdisplay /dev/sdb1    # Specific PV

vgdisplay              # All VGs (detailed)
vgdisplay vg01         # Specific VG

lvdisplay              # All LVs (detailed)
lvdisplay /dev/vg01/lv01  # Specific LV
```

### Key Fields to Check

| Command | Key fields |
|---|---|
| `pvdisplay` | PV Size, PE Size, Free PE, VG Name |
| `vgdisplay` | VG Size, Free PE / Size (how much space is left to allocate) |
| `lvdisplay` | LV Path, LV Size, LV Status |

```bash
# Check free space before extending an LV
vgdisplay vg01 | grep -i free
# Free PE / Size   179 / 716.00 MiB   <-- this is what you can use

# Quick overview of everything
pvs && vgs && lvs
```

---

## 3. Creating LVM Storage

### Full Creation Workflow

```bash
# Step 1: Create a partition with lvm flag set (optional - can use whole disk)
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 1MiB 769MiB
parted /dev/sdb set 1 lvm on
udevadm settle

# Step 2: Initialise as a Physical Volume
pvcreate /dev/sdb1
# Multiple PVs at once:
pvcreate /dev/sdb1 /dev/sdb2

# Step 3: Create a Volume Group from PVs
vgcreate VGNAME PV1 PV2...
vgcreate vg01 /dev/sdb1 /dev/sdb2

# Step 4: Create a Logical Volume from the VG
lvcreate -n LVNAME -L SIZE VGNAME
lvcreate -n lv01 -L 300M vg01

# Step 5: Format the LV with a filesystem
mkfs.xfs /dev/vg01/lv01
# or:
mkfs -t xfs /dev/vg01/lv01

# Step 6: Mount the LV
mkdir /data
mount /dev/vg01/lv01 /data

# Step 7: Make it persistent (/etc/fstab)
# Use the device mapper path or LVM path - both work
# UUID is also found with: lsblk --fs /dev/vg01/lv01
echo "/dev/vg01/lv01  /data  xfs  defaults  0  0" >> /etc/fstab
systemctl daemon-reload
```

### `lvcreate` Size Options

| Option | Example | Meaning |
|---|---|---|
| `-L SIZE` | `-L 300M` | Fixed size in human-readable units |
| `-L +SIZE` | `-L +500M` | Add this amount to existing size (used with `lvextend`) |
| `-l EXTENTS` | `-l 75` | Number of physical extents (75 × 4 MiB = 300 MiB) |
| `-l 100%FREE` | `-l 100%FREE` | Use all remaining free space in the VG |
| `-l 50%VG` | `-l 50%VG` | Use 50% of total VG space |

> **`-L` (capital) = human size. `-l` (lowercase) = extent count or percentage.**

### LV Device Paths

LVM creates two equivalent paths to every logical volume:

```bash
/dev/vg01/lv01            # Symlink (user-friendly)
/dev/mapper/vg01-lv01     # Actual device mapper device

# Both refer to the same block device
ls -l /dev/vg01/lv01      # Shows -> /dev/mapper/vg01-lv01

# df -h shows the /dev/mapper/ path
df -h /data
# /dev/mapper/vg01-lv01  300M  ...
```

---

## 4. Extending LVM Storage

### When to Extend

- LV is running out of space (`df -h` shows high usage)
- VG has free space: check with `vgdisplay | grep -i free`
- If VG is also full, first add a new PV to the VG

### Extend the VG First (if needed)

```bash
# Add a new PV
pvcreate /dev/sdc
# OR pvcreate a new partition on an existing disk
parted /dev/sdb mkpart secondary 769MiB 1025MiB
parted /dev/sdb set 2 lvm on
udevadm settle
pvcreate /dev/sdb2

# Add the new PV to an existing VG
vgextend vg01 /dev/sdc
vgextend vg01 /dev/sdb2

# Verify the VG now has more free space
vgdisplay vg01 | grep -i free
```

### Extend the LV and Filesystem

```bash
# Method 1: Two separate steps (explicit, good for understanding)
lvextend -L +500M /dev/vg01/lv01       # Extend LV by 500 MiB
xfs_growfs /data                        # Grow XFS to fill LV (use mount point)

# Method 2: One step with -r (recommended for production)
lvextend -L +500M -r /dev/vg01/lv01   # Extend LV AND resize filesystem together

# Extend to an absolute size
lvextend -L 1G /dev/vg01/lv01          # Set total size to 1 GiB

# Extend using all remaining free space in VG
lvextend -l +100%FREE -r /dev/vg01/lv01

# Verify the extension
df -h /data
lvdisplay /dev/vg01/lv01
```

### Filesystem Resize Commands

| Filesystem | Resize command | Takes | Notes |
|---|---|---|---|
| XFS | `xfs_growfs MOUNTPOINT` | Mount point | Grow only - cannot shrink |
| ext4 | `resize2fs /dev/vg01/lv01` | Device path | Can grow and shrink |

```bash
# XFS - must be mounted, takes mount point
xfs_growfs /data

# ext4 - can be unmounted, takes device path
resize2fs /dev/vg01/lv01
```

> **XFS can only grow, never shrink.** If you need to reduce an XFS LV, back up data, remove the LV, create a smaller one, and restore. This is a deliberate XFS design decision.

### Extending Swap LVs (Different Process)

Swap LVs must be taken offline before extending:

```bash
swapoff /dev/vg01/swap              # Deactivate swap
lvextend -L +300M /dev/vg01/swap   # Extend the LV
mkswap /dev/vg01/swap              # Re-initialise swap header
swapon /dev/vg01/swap              # Reactivate swap
```

---

## 5. Replacing Physical Volumes

Use `pvmove` to migrate data off a PV (failing disk, decommission, reallocation) while the system runs.

```bash
# Move all data off /dev/sdb1 to other PVs in the same VG
pvmove /dev/sdb1
# Shows percentage progress - data is accessible throughout

# Move with automatic metadata backup
pvmove -A y /dev/sdb1

# After pvmove completes, remove the now-empty PV from the VG
vgreduce vg01 /dev/sdb1

# Remove the LVM label from the partition
pvremove /dev/sdb1

# The partition is now free for other use
```

> **Always back up data before `pvmove`.** An unexpected power loss during the operation can leave the VG in an inconsistent state and cause data loss.

---

## 6. Removing LVM Storage

**Removal order is the exact reverse of creation order:**

```
CREATE: physical -> PV -> VG -> LV -> mkfs -> mount
REMOVE: umount -> lvremove -> vgremove -> pvremove
```

```bash
# Step 1: Unmount the filesystem
umount /data

# Step 2: Remove the fstab entry (edit /etc/fstab)
vim /etc/fstab   # delete the LV's line

# Step 3: Remove the LV (prompts for confirmation)
lvremove /dev/vg01/lv01

# Step 4: Remove the VG
vgremove vg01

# Step 5: Remove the PV labels
pvremove /dev/sdb1 /dev/sdb2

# The partitions are now available for other use
```

> **`lvremove`, `vgremove`, and `pvremove` are destructive and irreversible.** There is no undo. Confirm the correct LV/VG/PV name before running. `lvremove` prompts for confirmation; `vgremove` and `pvremove` do not always.

---

## 7. Viewing LVM in Context

```bash
# See LVM volumes alongside regular partitions
lsblk
# NAME          MAJ:MIN  RM  SIZE RO TYPE MOUNTPOINTS
# sda             8:0     0   10G  0 disk
# sdb             8:16    0    5G  0 disk
# ├─sdb1          8:17    0  768M  0 part
# │ └─vg01-lv01  253:0   0  300M  0 lvm  /data
# └─sdb2          8:18    0  256M  0 part
#   └─vg01-swap  253:1   0  244M  0 lvm  [SWAP]

# Show filesystem info for all devices including LVM
lsblk --fs

# Check free space on mounted LVs
df -h

# Confirm LV details (path, size, VG)
lvs
lvdisplay /dev/vg01/lv01
```

---

## Quick Reference: All Commands

```bash
# --- Inspect ---
pvs                                    # Summary of all PVs
vgs                                    # Summary of all VGs
lvs                                    # Summary of all LVs
pvdisplay /dev/sdb1                    # Detailed PV info
vgdisplay vg01                         # Detailed VG info (check Free PE)
lvdisplay /dev/vg01/lv01               # Detailed LV info
lsblk                                  # Block device tree view

# --- Create ---
pvcreate /dev/sdb1                     # Initialise a PV
vgcreate vg01 /dev/sdb1 /dev/sdb2     # Create VG from PVs
lvcreate -n lv01 -L 300M vg01         # Create LV (human size)
lvcreate -n lv01 -l 75 vg01           # Create LV (75 PEs)
lvcreate -n lv01 -l 100%FREE vg01     # Create LV using all free space
mkfs.xfs /dev/vg01/lv01               # Format LV with XFS
mkdir /data && mount /dev/vg01/lv01 /data  # Mount

# --- Extend ---
pvcreate /dev/sdc                      # Prep new PV
vgextend vg01 /dev/sdc                 # Add PV to VG
lvextend -L +500M /dev/vg01/lv01      # Extend LV by 500 MiB
lvextend -L 1G /dev/vg01/lv01         # Extend LV to 1 GiB total
lvextend -l +100%FREE /dev/vg01/lv01  # Use all free space
lvextend -L +500M -r /dev/vg01/lv01   # Extend LV AND resize FS (best!)
xfs_growfs /data                       # Grow XFS (mount point)
resize2fs /dev/vg01/lv01              # Grow ext4 (device path)

# --- Replace PV ---
pvmove /dev/sdb1                       # Move data off a PV (live)
vgreduce vg01 /dev/sdb1               # Remove empty PV from VG
pvremove /dev/sdb1                     # Remove LVM label from partition

# --- Remove (in order!) ---
umount /data                           # Unmount filesystem first
lvremove /dev/vg01/lv01               # Remove LV (asks confirmation)
vgremove vg01                          # Remove VG
pvremove /dev/sdb1                     # Remove PV labels
```

---

## Key Files and Paths

| Path | Purpose |
|---|---|
| `/dev/VGNAME/LVNAME` | Symlink to the LV (user-friendly path) |
| `/dev/mapper/VGNAME-LVNAME` | Actual device mapper path (shown in `df`) |
| `/etc/lvm/backup/` | Automatic VG metadata backups |
| `/etc/lvm/devices/system.devices` | LVM device filter |

---

## Common Mistakes and Fixes

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot `xfs_growfs` after `lvextend` | `df -h` still shows old size | Run `xfs_growfs MOUNTPOINT` |
| Used `lvextend -L 500M` instead of `-L +500M` | LV shrinks to 500M instead of growing by 500M | Use `+` to mean "add this much"; without `+` it sets the absolute size |
| Tried to shrink XFS | Error: XFS does not support shrinking | Back up, recreate smaller, restore |
| VG has no free space | `lvextend` fails | `pvcreate` new device + `vgextend` first |
| Wrong removal order | `vgremove` fails because LV still exists | Unmount and `lvremove` first |
| `lvremove` fails | LV is still mounted | `umount MOUNTPOINT` first |

---

## Things to Remember

1. **LVM has three layers: PV -> VG -> LV.** Physical Volumes pool into a Volume Group. Logical Volumes are carved out of the VG. The filesystem sits on top of the LV. Understand the layers before touching the commands.

2. **`-L` (capital) takes human-readable sizes. `-l` (lowercase) takes PE counts or percentages.** `-L 300M` and `-l 75` (with default 4 MiB PEs) are equivalent. `-l 100%FREE` is the easiest way to use all remaining VG space.

3. **Use `lvextend -r` to extend the LV and resize the filesystem in one step.** Forgetting to run `xfs_growfs` after `lvextend` is the most common mistake in this chapter. The `-r` flag prevents it.

4. **`xfs_growfs` takes a mount point; `resize2fs` takes a device path.** And XFS can only grow, never shrink. These two differences between XFS and ext4 resize commands are frequently tested.

5. **Always check `vgdisplay | grep -i free` before `lvextend`.** If the VG has no free space, `lvextend` fails. Add a PV with `pvcreate + vgextend` first.

6. **The LV appears at both `/dev/VGNAME/LVNAME` and `/dev/mapper/VGNAME-LVNAME`.** They are the same device. `df -h` shows the `/dev/mapper/` path. Either can be used in `/etc/fstab`.

7. **Removal order is the reverse of creation order.** umount -> lvremove -> vgremove -> pvremove. Trying to remove a VG that still contains LVs will fail.

8. **`pvmove` migrates data live while the LV is mounted and in use.** Use it to drain a failing disk or decommission a partition without downtime. Always back up data before running `pvmove`.

9. **`lvremove`, `vgremove`, and `pvremove` are irreversible.** They do not move data to a recycle bin. Confirm the correct names before running, especially in scripts.

10. **A whole disk can be used as a PV without partitioning it first.** `pvcreate /dev/sdc` (not `/dev/sdc1`) is valid and simpler. The partitioning step is optional and only needed if you want to mix LVM and non-LVM use on the same disk.
