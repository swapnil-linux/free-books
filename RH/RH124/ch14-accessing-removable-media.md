# Chapter 14 – Accessing Removable Media
## RH124 Student Quick Reference

---

## The Big Picture — How Linux Handles Storage

Coming from Windows, you expect drives to appear as letters (`C:\`, `D:\`). In Linux, **storage is attached to the filesystem tree as a directory** — a process called mounting.

```
Windows:                          Linux:
C:\ ──► OS and programs           /     ──► root filesystem (on /dev/sda3)
D:\ ──► Data drive                /boot ──► boot files (on /dev/sda2)
E:\ ──► USB stick                 /home ──► user data (on /dev/sdb1)
                                  /mnt/usb ──► USB stick (on /dev/sdc1)
```

Everything is reachable from `/`. Storage devices are not separate entities — they are directories in the tree.

---

## Filesystem Types

| Filesystem | Where Used | Notes |
|---|---|---|
| **XFS** | Default on RHEL | High performance, good for large files, default since RHEL 7 |
| **ext4** | Also supported on RHEL | Mature, widely used across all Linux distros |
| **exFAT** | Removable media | Supported from RHEL 10 — good for USB sticks shared with Windows |
| **vfat / FAT32** | USB sticks, EFI boot | Universal compatibility with Windows and macOS |
| **iso9660** | CD/DVD/ISO files | Read-only optical disc format |
| **tmpfs** | RAM-backed (`/tmp`, `/run`) | Exists in memory only — lost on reboot |
| **NTFS** | Windows disks | Readable on Linux with `ntfs-3g` — not a native Linux format |

> **Windows interoperability:** For USB sticks shared between Windows and Linux, format as **exFAT** (RHEL 10+) or **FAT32** (universal). NTFS works but is not ideal. Linux native formats (XFS, ext4) are not readable by Windows without third-party software.

---

## Block Devices — How Linux Sees Disks

Every storage device appears as a file in `/dev/`:

| Device Type | Naming Pattern | Example |
|---|---|---|
| SATA / SAS / USB drives | `/dev/sda`, `/dev/sdb`, … | `/dev/sda` = first disk, `/dev/sdb` = second |
| NVMe SSDs | `/dev/nvme0`, `/dev/nvme1`, … | `/dev/nvme0n1` = first NVMe drive |
| VM paravirtual (virtio-blk) | `/dev/vda`, `/dev/vdb`, … | Common in KVM/QEMU VMs |
| SD / eMMC cards | `/dev/mmcblk0`, `/dev/mmcblk1`, … | Raspberry Pi, embedded devices |

### Partitions

Partitions are numbered suffixes on the device name:

```
/dev/sda     ← entire first disk
/dev/sda1    ← first partition on first disk
/dev/sda2    ← second partition on first disk
/dev/sdb3    ← third partition on second disk
/dev/nvme0n1p1  ← first partition on first NVMe drive
```

> **Windows equivalent:** `/dev/sda` = Disk 0 in Disk Management. `/dev/sda1` = the first volume/partition on that disk.

---

## Viewing Disks and Filesystems

```bash
lsblk                           # tree view of all block devices and partitions
lsblk -f                        # include filesystem types and labels
lsblk -fp                       # full paths + UUID + mount points (most useful)

df                              # disk space usage of all mounted filesystems
df -h                           # human-readable sizes (GB, MB)
df -H                           # human-readable using SI units (1000 not 1024)
df -hT                          # include filesystem type in output
df /home                        # space usage for a specific filesystem

du -h /var/log                  # disk usage of a directory and its contents
du -sh /var/log                 # total size summary only
du -sh /*                       # size of each top-level directory
du -h --max-depth=1 /var        # one level deep only
```

### Understanding `df` Output

```
Filesystem      Size  Used  Avail  Use%  Mounted on
/dev/sda3        20G   6G    14G   30%   /
/dev/sda2       200M   9M   191M    5%   /boot/efi
tmpfs           2.8G   84K  2.8G    1%   /dev/shm
```

---

## Mounting Filesystems Manually

### The Workflow

```bash
# 1 — Identify the device
lsblk -fp

# 2 — Create a mount point (an empty directory)
sudo mkdir /mnt/mydata

# 3 — Mount the device
sudo mount /dev/sdb1 /mnt/mydata          # by device name
sudo mount UUID="xxxx-xxxx" /mnt/mydata   # by UUID (more reliable)

# 4 — Use the filesystem
ls /mnt/mydata

# 5 — Unmount when done
sudo umount /mnt/mydata
```

### Why Use UUID Instead of Device Name?

Device names like `/dev/sdb` are assigned by **detection order** — if you add or remove a disk, `/dev/sdb` might point to a different disk next time. UUIDs are permanent identifiers tied to the filesystem itself.

```bash
# Find the UUID of a device
lsblk -fp /dev/sdb
blkid /dev/sdb1

# Mount using UUID
sudo mount UUID="2dfe4a17-d529-480a-a53e-9c28ff49545e" /mnt/data
```

---

## Unmounting

```bash
sudo umount /mnt/mydata             # unmount by mount point
sudo umount /dev/sdb1               # unmount by device name

# If umount says "target is busy"
lsof /mnt/mydata                    # find which processes are using it
fuser -m /mnt/mydata                # alternative: list PIDs using the mount

# The most common cause — your shell is inside the directory
cd /                                # move out first, then unmount
sudo umount /mnt/mydata
```

> ⚠️ **Always unmount before physically unplugging a device.** Linux buffers writes in memory — unplugging without unmounting can corrupt data, exactly like removing a USB stick from Windows without "safely removing" it first.

---

## Automatic Mounting on the Desktop

On systems with a GNOME desktop, plugging in a USB stick mounts it automatically at:

```
/run/media/USERNAME/LABEL
```

For example: `/run/media/student/MyUSB`

To safely remove: right-click the drive in the file manager and select Eject, or:

```bash
umount /run/media/student/MyUSB
```

---

## Persistent Mounting — `/etc/fstab`

The `/etc/fstab` file defines filesystems that mount automatically at boot. Editing this file is beyond RH124 scope but important context:

```bash
cat /etc/fstab
```

Each line follows this format:
```
UUID=xxxx   /mount/point   fstype   options   dump   pass
```

Example:
```
UUID=2dfe4a17   /mnt/data   xfs   defaults   0   0
```

> **Warning:** A typo in `/etc/fstab` can prevent the system from booting. Always test with `sudo mount -a` before rebooting, and always edit with `sudo`.

---

## Locating Files — `locate` vs `find`

| | `locate` | `find` |
|---|---|---|
| **Speed** | Very fast (uses a database index) | Slower (searches in real time) |
| **Accuracy** | May miss recently created files | Always current |
| **Requires update** | `sudo updatedb` to refresh | No — always live |
| **Search by** | Filename only | Filename, size, owner, permissions, date, type, and more |
| **Best for** | Quick name lookups | Complex, precise searches |

---

## `locate` — Fast File Name Search

```bash
locate filename                 # find all paths containing "filename"
locate passwd                   # find all paths containing "passwd"
locate -i filename              # case-insensitive search
locate -n 5 passwd              # limit to first 5 results
locate '*.conf' | grep /etc     # find .conf files under /etc

sudo updatedb                   # refresh the locate database (run as root)
```

> On RHEL, `locate` comes from the `mlocate` or `plocate` package. The database is updated automatically by a daily cron job, but run `updatedb` manually after creating files you need to find immediately.

---

## `find` — Powerful Real-Time Search

### By Name

```bash
find / -name sshd_config                    # exact name match
find /etc -name '*.conf'                    # wildcard — quote it
find /etc -iname '*network*'               # case-insensitive
find /home -name '*.txt' -type f            # files only, not directories
```

### By Type

```bash
find /etc -type d                           # directories only
find /etc -type f                           # regular files only
find / -type l                              # symbolic links only
find /dev -type b                           # block devices only
```

### By Owner and Group

```bash
find /home -user alice                      # owned by user alice
find /var -group mail                       # owned by group mail
find / -user root -group mail               # owned by root AND group mail
find / -nouser                              # files with no valid owner (orphaned)
find / -nogroup                             # files with no valid group
```

### By Permissions

```bash
find /home -perm 644                        # exact permission match
find /home -perm -644                       # at least these permissions set
find /home -perm /u=w                       # owner has write (any match)
find / -perm -4000                          # setuid files (potential security concern)
find / -perm -2000                          # setgid files
find / -perm -4000 -type f 2>/dev/null      # setuid executables system-wide
```

### By Size

```bash
find / -size 10M                            # exactly 10 MB (rounds up)
find / -size +100M                          # larger than 100 MB
find / -size -10k                           # smaller than 10 KB
find /var -size +50M -size -1G              # between 50 MB and 1 GB
```

Size units: `k` = kilobytes, `M` = megabytes, `G` = gigabytes (note case sensitivity)

### By Modification Time

```bash
find / -mmin -60                            # modified in the last 60 minutes
find / -mmin +120                           # modified more than 120 minutes ago
find / -mtime -1                            # modified in the last 24 hours
find / -mtime +30                           # modified more than 30 days ago
find /var/log -mtime +7 -name '*.log'       # log files older than 7 days
```

### Combining Multiple Criteria

```bash
# AND (default — both must match)
find /home -user alice -name '*.txt'

# OR
find /home -name '*.txt' -o -name '*.log'

# NOT
find /home -not -user alice
find /home ! -name '*.txt'

# Complex example — find large old log files
find /var/log -type f -name '*.log' -size +10M -mtime +30
```

### Taking Action on Results

```bash
# List results with details (like ls -l)
find /home -name '*.txt' -ls

# Execute a command on each result
find /tmp -name '*.tmp' -delete             # delete all .tmp files
find /home -type f -exec chmod 640 {} \;   # change perms on found files
find /var/log -mtime +30 -exec ls -lh {} \;  # list old log files

# Pass results to another command
find /home -name '*.txt' | xargs wc -l     # count lines in all .txt files
```

> The `{}` is a placeholder for the found file. The `\;` ends the `-exec` expression.

---

## Real-World Search Examples

```bash
# Find files modified in the last hour (incident response)
find / -mmin -60 -type f 2>/dev/null

# Find all setuid binaries (security audit)
find / -type f -perm -4000 2>/dev/null

# Find files not owned by any user (post-userdel cleanup)
find / -nouser 2>/dev/null

# Find large files consuming disk space
find / -type f -size +500M 2>/dev/null | sort

# Find config files modified today
find /etc -type f -mtime -1

# Find world-writable files (security concern)
find / -type f -perm -002 2>/dev/null

# Find all .log files older than 90 days and delete them
find /var/log -name '*.log' -mtime +90 -delete
```

---

## Windows Comparison

| Windows | Linux | Notes |
|---|---|---|
| Disk letters (C:, D:) | Mount points (`/`, `/home`, `/mnt/data`) | Linux uses directories, not letters |
| Disk Management | `lsblk`, `fdisk`, `parted` | View and manage disks/partitions |
| File Explorer "This PC" | `df -h` | See all mounted drives and free space |
| Folder size (Properties) | `du -sh /path` | Total size of a directory tree |
| "Safely Remove Hardware" | `umount /dev/sdX` | Flush and detach a device |
| Autoplay (USB insert) | `/run/media/user/label` | GNOME auto-mounts removable media |
| Windows Search (Explorer) | `locate` (fast) or `find` (precise) | Locate searches an index; find searches live |
| `dir /s /b *.txt` | `find / -name '*.txt'` | Recursive file search by name |
| Search by date modified | `find / -mtime -1` | Files modified in the last 24 hours |
| File size filter | `find / -size +100M` | Files larger than 100 MB |

---

## Things to Remember

- **Always `umount` before unplugging** — data corruption is real if you do not
- **Use UUID not device name** for reliable mounting — device names can change between reboots
- **`lsblk -fp`** is your go-to command for understanding the current disk layout
- **`locate` is fast but stale** — always run `sudo updatedb` after creating files you need to find
- **`find` is live and precise** — use it when freshness matters or when searching by anything other than name
- **Quote wildcards in `find`** — `find / -name '*.conf'` not `find / -name *.conf` (the shell expands the latter before `find` sees it)
- **`2>/dev/null`** in `find` commands silences the many "Permission denied" errors when searching system directories as a regular user
- **`-mtime` uses days, `-mmin` uses minutes** — easy to confuse
- **Unmount fails with "target is busy"** — use `lsof /mountpoint` to find what is holding it open; the most common cause is your shell's current directory being inside the mount point
