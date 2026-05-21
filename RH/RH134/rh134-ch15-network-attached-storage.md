# RH134 Chapter 15 - Accessing Network-attached Storage

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Access network-attached storage that the NFS protocol provides, either manually or by using the automounter.

---

## Windows vs Linux: Network Storage Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Map Network Drive (GUI) | `mount -t nfs server:/share /mountpoint` |
| Persistent mapped drive (Group Policy / fstab) | `/etc/fstab` NFS entry |
| DFS (Distributed File System) | autofs with indirect maps |
| `net use Z: \\server\share` | `mount -t nfs server:/share /mnt` |
| `net use /persistent:yes` | `/etc/fstab` entry |
| SMB/CIFS file shares (Windows native) | `mount -t cifs //server/share /mnt` |
| NFS client for Windows | NFS is native on Linux - no client install needed |
| AutoConnect on login | autofs (mounts on demand, unmounts when idle) |
| `showmount` equivalent | `showmount -e server` or browse NFSv4 root |

---

## Chapter Overview: Three Mounting Methods

| Method | Persistence | Mount timing | Use case |
|---|---|---|---|
| `mount -t nfs` | None (lost on reboot) | Immediate | Testing, one-off access |
| `/etc/fstab` | Permanent (always mounted) | At boot | Always-needed shared storage |
| `autofs` | Permanent config, on-demand mount | When first accessed | Home dirs, occasional shared data |

> **The `/etc/fstab` boot hang risk:** If an NFS server is unreachable at boot and the share is in `/etc/fstab`, the system waits up to 90 seconds per mount before giving up. Ten NFS mounts = potential 15-minute boot delay. Use `nofail` in options for optional mounts, or use `autofs` for large deployments.

---

## Part 1: Mounting NFS Filesystems

### Required Package

```bash
dnf install nfs-utils
```

### Discovering NFS Exports

```bash
# List all exports on a server (uses portmapper - may not work on NFSv4-only servers)
showmount -e serverb

# Alternative for NFSv4: mount the server root and browse
mount serverb:/ /mnt
ls /mnt
umount /mnt
```

### Manual Mount (Temporary)

```bash
# Basic NFS mount (uses NFSv4.2 by default on RHEL 10)
mount -t nfs serverb:/shares /mnt

# Mount with specific options
mount -t nfs -o rw,sync serverb:/shares /mnt
mount -t nfs -o ro serverb:/shares/docs /mnt/docs

# Verify the mount
mount | grep nfs
df -h /mnt

# Unmount
umount /mnt
```

### Common NFS Mount Options

| Option | Meaning |
|---|---|
| `rw` | Read/write (default) |
| `ro` | Read-only |
| `sync` | Writes confirmed by server before returning to client (safer, slower) |
| `async` | Writes buffered client-side (faster, risk of data loss if server crashes) |
| `hard` | Hang client process if server unreachable (default - safe for data integrity) |
| `soft` | Return error to client if server unreachable (can corrupt writes) |
| `nofail` | Do not fail boot if mount fails (use in fstab for optional NFS) |
| `vers=4.2` | Force specific NFS version |
| `sec=sys` | Use standard Unix UID/GID security |
| `_netdev` | Mark as network device (systemd waits for network before mounting) |

> **`hard` vs `soft`:** Red Hat recommends `hard` (the default) for production data. A `soft` mount returns an error if the server is unreachable, which can cause data corruption if a write was in progress. Use `soft` only for read-only reference data where an error is preferable to a hang.

### Persistent Mount via `/etc/fstab`

```bash
# NFS entry format in /etc/fstab
SERVER:/EXPORT   MOUNTPOINT   nfs   OPTIONS   0 0

# Basic examples
serverb:/shares         /mnt/shares   nfs   defaults,_netdev    0 0
serverb:/shares/docs    /mnt/docs     nfs   ro,sync,_netdev     0 0
serverb:/shares/data    /mnt/data     nfs   rw,sync,hard,_netdev  0 0

# Optional mount (will not halt boot if unreachable)
serverb:/shares         /mnt/shares   nfs   defaults,nofail,_netdev  0 0
```

```bash
# After adding to fstab
systemctl daemon-reload

# Test mount all fstab entries (verify before rebooting)
mount -a

# Verify
mount | grep nfs
df -h
```

> **Always use `_netdev` for NFS entries in `/etc/fstab`.** This tells systemd to wait until networking is available before attempting the mount. Without it, the mount attempt may occur before the network interface is up, causing failure at boot.

---

## Part 2: Automounting with autofs

### Why autofs?

- Mounts NFS shares **on demand** when a user accesses the path
- **Unmounts automatically** after an idle timeout (default 5 minutes)
- Works for **non-root users** (regular users cannot run `mount`)
- **No boot delay** if NFS server is unreachable - the mount only happens when accessed
- **Always uses current config** - not the stale config from last boot like fstab

### Required Packages

```bash
dnf install autofs nfs-utils
```

### autofs Configuration Architecture

```
/etc/auto.master.d/myconfig.autofs    <- master map (drop-in file, .autofs extension)
         |
         v
/etc/auto.mymap                       <- map file (list of mount points)
         |
         v
NFS server exports (mounted on demand)
```

### The Master Map File

Place drop-in files in `/etc/auto.master.d/`. Each file must have the `.autofs` extension.

```bash
# Master map format:
BASEDIR   MAPFILE

# Direct map: /- means the map file contains absolute paths
/- /etc/auto.direct

# Indirect map: /shares is the base directory; map file has relative paths
/shares /etc/auto.indirect

# Indirect map: /remote is the base; map file uses wildcard
/remote /etc/auto.shares
```

### Direct Maps

A direct map mounts a specific NFS export to a known, absolute mount point. The full path is in the map file.

```bash
# Master map entry (/etc/auto.master.d/direct.autofs):
/- /etc/auto.direct

# Map file (/etc/auto.direct):
MOUNTPOINT   OPTIONS   SERVER:/EXPORT
/mnt/docs   -rw,sync   serverb:/shares/docs
/mnt/data   -rw,sync   serverb:/shares/data
/public     -ro,sync   serverb:/public
```

> **Direct map mount points are always visible** as empty directories in the filesystem, even when nothing is mounted. autofs manages them.

### Indirect Maps

An indirect map uses a base directory. The map file contains relative subdirectory names that are appended to the base directory.

```bash
# Master map entry (/etc/auto.master.d/shares.autofs):
/shares /etc/auto.indirect

# Map file (/etc/auto.indirect):
SUBDIRNAME   OPTIONS   SERVER:/EXPORT
docs        -rw,sync   serverb:/shares/docs
data        -rw,sync   serverb:/shares/data
```

- Accessing `/shares/docs` mounts `serverb:/shares/docs` at `/shares/docs`
- Accessing `/shares/data` mounts `serverb:/shares/data` at `/shares/data`
- autofs **creates and removes the subdirectories** automatically

> **Indirect map subdirectories do NOT appear** with `ls /shares` until accessed. They are created on demand.

### Wildcard Indirect Maps

When an NFS server exports multiple subdirectories, use the `*` (wildcard) and `&` (substitution) special characters to handle all of them with one line.

```bash
# Map file using wildcard (/etc/auto.shares):
*   -rw,sync   serverb:/shares/&
```

- `*` matches any subdirectory name typed by the user
- `&` substitutes the matched name into the source path
- Accessing `/remote/management` mounts `serverb:/shares/management`
- Accessing `/remote/production` mounts `serverb:/shares/production`
- New exports on the server are automatically available - no config update needed

```bash
# Full example:
# /etc/auto.master.d/shares.autofs:
/remote /etc/auto.shares

# /etc/auto.shares:
*   -rw,sync,fstype=nfs4   serverb:/shares/&

# Result: accessing /remote/ANYTHING mounts serverb:/shares/ANYTHING
```

### Map File Options Format

```bash
# Options start with a dash (-) and are comma-separated (no spaces)
MOUNTPOINT   -OPTION1,OPTION2,OPTION3   SERVER:/EXPORT

# Examples:
work          -rw,sync                serverb:/shares/work
docs          -ro,sync                serverb:/shares/docs
data          -rw,sync,fstype=nfs4    serverb:/shares/data
*             -rw,sync                serverb:/shares/&
/mnt/secure   -ro,hard,sec=krb5      kerb.example.com:/secured
```

### Starting and Enabling autofs

```bash
# Start and enable at boot
systemctl enable --now autofs

# Verify service is running
systemctl status autofs

# After changing map files, reload without full restart
systemctl reload autofs
# OR restart if needed
systemctl restart autofs
```

### Verifying autofs

```bash
# Check nothing is mounted yet (expected)
mount | grep nfs

# Trigger an automount by accessing the path
ls /remote/management

# Now check - the NFS mount appears
mount | grep nfs

# After idle timeout (default 5 minutes), check again - it unmounts
mount | grep nfs   # empty again
```

### The systemd Alternative to autofs

For simpler cases (direct maps only), add `x-systemd.automount` to an `/etc/fstab` NFS entry:

```bash
# /etc/fstab:
serverb:/shares/docs   /mnt/docs   nfs   defaults,_netdev,x-systemd.automount   0 0

# After adding the fstab entry:
systemctl daemon-reload

# Enable the automount unit (name is based on mount path)
systemctl start mnt-docs.automount
# (hyphens replace / in the unit name; /mnt/docs -> mnt-docs.automount)
```

> Limitation: systemd automount only supports absolute path mount points (like direct maps). It does not support the wildcard indirect map pattern. Use `autofs` when dynamic subdirectory mounting is needed.

---

## Complete Workflow Examples

### Workflow 1: Manual NFS mount (testing)

```bash
# Install required package
dnf install nfs-utils

# Discover what the server exports
showmount -e serverb

# Create mount point and mount
mkdir /mnt/test
mount -t nfs serverb:/shares /mnt/test

# Verify and use
mount | grep nfs
ls /mnt/test

# Unmount when done
umount /mnt/test
```

### Workflow 2: Persistent NFS mount

```bash
# Add to /etc/fstab
vim /etc/fstab
# serverb:/shares   /mnt/shares   nfs   rw,sync,hard,_netdev   0 0

# Reload systemd and test
systemctl daemon-reload
mount -a

# Verify
mount | grep nfs
df -h /mnt/shares
```

### Workflow 3: autofs indirect map with wildcard

```bash
# Step 1: Install packages
dnf install autofs nfs-utils

# Step 2: Create master map drop-in file
cat > /etc/auto.master.d/shares.autofs << 'EOF'
/remote /etc/auto.shares
EOF

# Step 3: Create map file with wildcard
cat > /etc/auto.shares << 'EOF'
* -rw,sync,fstype=nfs4 serverb:/shares/&
EOF

# Step 4: Start and enable autofs
systemctl enable --now autofs

# Step 5: Verify - nothing mounted yet
mount | grep nfs

# Step 6: Trigger a mount by accessing the path
ls /remote/management

# Step 7: Confirm it mounted
mount | grep nfs
```

---

## Quick Reference: All Commands

```bash
# --- Packages ---
dnf install nfs-utils autofs

# --- Discover exports ---
showmount -e SERVERNAME

# --- Manual mount/unmount ---
mount -t nfs SERVER:/EXPORT /MOUNTPOINT
mount -t nfs -o rw,sync SERVER:/EXPORT /MOUNTPOINT
umount /MOUNTPOINT

# --- View mounts ---
mount | grep nfs
df -h
findmnt -t nfs

# --- autofs service ---
systemctl enable --now autofs
systemctl status autofs
systemctl reload autofs       # After changing map files

# --- Verify autofs ---
mount | grep nfs             # Should be empty before access
ls /AUTOMOUNT_PATH           # Trigger the mount
mount | grep nfs             # Mount now appears

# --- fstab testing ---
systemctl daemon-reload
mount -a
findmnt --verify
```

---

## Key Files and Paths

| Path | Purpose |
|---|---|
| `/etc/auto.master` | Legacy master map file (avoid editing directly) |
| `/etc/auto.master.d/` | Drop-in master map files (preferred - use `.autofs` extension) |
| `/etc/auto.NAME` | Map files defining individual automounts |
| `/etc/fstab` | Persistent mounts including NFS entries |
| `/var/lib/nfs/` | NFS client state files |

---

## Direct vs Indirect Map Summary

| Feature | Direct Map | Indirect Map |
|---|---|---|
| Master map entry | `/- /etc/auto.direct` | `/base/dir /etc/auto.indirect` |
| Map file paths | Absolute (`/mnt/docs`) | Relative (`docs`) |
| Mount location | Exactly as specified | base/dir + relative name |
| Directories visible before access | Yes (empty) | No |
| Wildcard `*` support | No | Yes |
| Best for | Fixed known paths | Dynamic sets of exports |

---

## Things to Remember

1. **autofs mounts on demand and unmounts when idle.** Nothing is mounted until a user or process accesses the path. After the idle timeout (default 5 minutes), the mount disappears. This is by design, not a failure.

2. **Regular users can access autofs mounts without needing root.** autofs is a kernel-assisted service that mounts on behalf of any user. `mount` requires root; autofs does not.

3. **`_netdev` is required in `/etc/fstab` NFS entries.** Without it, systemd may attempt the mount before the network interface is ready, causing boot failure.

4. **`nofail` in fstab prevents NFS boot hangs.** If the NFS server is unreachable at boot, `nofail` allows the system to continue booting instead of waiting for timeouts.

5. **Master map drop-in files must have the `.autofs` extension.** Files in `/etc/auto.master.d/` without the `.autofs` extension are silently ignored.

6. **After changing autofs map files, `systemctl reload autofs`.** Changes to map files take effect after a reload. A restart also works but causes any currently mounted autofs shares to be briefly unmounted.

7. **In indirect map files, options start with `-` and have no spaces.** `-rw,sync` is correct. `-rw, sync` (with space) fails. The format is `MOUNTPOINT -OPTIONS SERVER:/EXPORT`.

8. **The `*` wildcard and `&` substitution only work in indirect maps.** You cannot use these in direct maps. The `&` in the source replaces with whatever `*` matched.

9. **`hard` NFS mounts (the default) hang if the server is unreachable.** The client process cannot be killed normally during the hang. For critical production data this is correct behaviour - it preserves data integrity. For non-critical reads, `soft` returns an error instead.

10. **`mount | grep nfs` is the quickest way to see active NFS mounts.** Use it before accessing a path to confirm nothing is mounted, access the path, then run it again to confirm autofs triggered the mount correctly.
