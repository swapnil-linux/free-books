# RH134 Chapter 8 - Transferring Files

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Securely transfer files to and from remote systems, and efficiently synchronise directory content between systems.

---

## Windows vs Linux: File Transfer Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| WinSCP (GUI SFTP client) | `sftp` command (CLI) |
| FileZilla (SFTP mode) | `sftp` command |
| `robocopy` (copy with sync) | `rsync` |
| `xcopy /s` (recursive copy) | `scp -r` or `rsync -r` |
| Windows FTP client (port 21) | `sftp` (port 22, inside SSH - no extra port) |
| `pscp` (PuTTY SCP) | `scp` |
| DFS Replication | `rsync` with cron or systemd timer |
| OneDrive / SharePoint sync | `rsync` to a remote host |

> **Key difference from Windows FTP:** SFTP is not FTP. It runs entirely inside an SSH connection on port 22. No extra daemon, no extra firewall rule, no separate certificate. If SSH works, SFTP works.

---

## Tool Selection Guide

| Tool | Best for | Interactive? | Preserves attributes? |
|---|---|---|---|
| `sftp` | Interactive browsing and transfer | Yes | No (by default) |
| `scp` | Quick non-interactive file copy | No | No (by default) |
| `rsync` | Efficient sync, backups, automation | No | Yes (with `-a -A -X`) |

> **RHEL 10 change:** `scp` now uses the SFTP protocol internally (not the old, vulnerable SCP protocol). The `-O` flag reverts to the old protocol, but this is not recommended. Prefer `sftp` or `rsync` for new scripts and automation.

---

## Part 1: SFTP - Interactive File Transfer

### Starting an SFTP Session

```bash
# Connect to remote host as the same user
sftp remotehost

# Connect as a specific user
sftp remoteuser@remotehost

# Connect to a specific path (non-interactive download only)
sftp remoteuser@remotehost:/remote/path/file.txt
```

### The Two-World Mental Model

Inside an sftp session, every command operates on the REMOTE host by default.
Prefix a command with `l` to run it on the LOCAL host instead.

| Remote command | Local equivalent | What it does |
|---|---|---|
| `pwd` | `lpwd` | Print working directory |
| `cd path` | `lcd path` | Change directory |
| `ls` | `lls` | List directory contents |
| `mkdir dir` | `lmkdir dir` | Create a directory |

### SFTP Session Commands

```
help              Show all available commands
pwd               Print current directory on remote host
lpwd              Print current directory on local host
cd PATH           Change directory on remote host
lcd PATH          Change directory on local host
ls                List files on remote host
lls               List files on local host
mkdir DIRNAME     Create directory on remote host
rmdir DIRNAME     Remove directory on remote host
get FILE          Download file from remote to local current directory
get -r DIR        Download directory recursively
put FILE          Upload file from local to remote current directory
put -r DIR        Upload directory recursively
rm FILE           Delete file on remote host
rename OLD NEW    Rename file on remote host
exit / bye / quit Exit the sftp session
```

### SFTP Workflow Examples

```bash
# Start an interactive session
sftp student@servera

# --- Inside the sftp session ---

# Check where you are (remote and local)
sftp> pwd
Remote working directory: /home/student
sftp> lpwd
Local working directory: /home/user

# Navigate locally to where you want downloads to land
sftp> lcd /home/user/downloads/

# Browse the remote host
sftp> cd /etc/ssh
sftp> ls

# Download a single file
sftp> get sshd_config

# Download a whole directory recursively
sftp> get -r /etc/ssh

# Navigate to a target directory on the remote
sftp> mkdir /home/student/uploads
sftp> cd /home/student/uploads

# Upload a file
sftp> put /etc/hosts

# Upload a directory recursively
sftp> put -r /home/user/scripts

# Exit
sftp> exit
```

---

## Part 2: SCP - Non-Interactive File Copy

### Basic Syntax

```bash
# Local to remote
scp SOURCE remoteuser@remotehost:DESTINATION

# Remote to local
scp remoteuser@remotehost:SOURCE DESTINATION

# Remote to remote (via local machine)
scp user1@host1:/path/file user2@host2:/path/
```

### SCP Examples

```bash
# Copy a single file to a remote host's home directory
scp /etc/hosts student@servera:

# Copy a single file to a specific remote path
scp /etc/hosts student@servera:/tmp/hosts-backup

# Copy a single file FROM a remote host to local
scp student@servera:/etc/hostname /tmp/

# Copy multiple files to remote
scp /etc/hosts /etc/hostname student@servera:/tmp/

# Copy a directory recursively (-r)
scp -r /etc/ssh student@servera:/tmp/ssh-backup/

# Copy a directory recursively FROM a remote host
scp -r student@servera:/etc/ssh ~/serverbackup/
```

### Common SCP Options

| Option | Purpose |
|---|---|
| `-r` | Recursive - copy directories |
| `-p` | Preserve file timestamps and permissions |
| `-P PORT` | Use a non-standard SSH port (capital P) |
| `-q` | Quiet - suppress progress output |
| `-C` | Compress data during transfer |
| `-O` | Use old SCP protocol (not recommended) |

> **Note on RHEL 10:** If `/etc/ssh/disable_scp` exists on the remote server, the legacy SCP protocol is blocked and `-O` will not work. The default SFTP-backed `scp` will still work.

---

## Part 3: rsync - Efficient Synchronisation

### How rsync Works

rsync transfers only the differences between source and destination. On the first run, it copies everything. On subsequent runs, it detects what has changed and transfers only those changes. This makes it dramatically faster than `scp` for keeping directories in sync.

```
First sync:  Transfers 500 MB of /var/log/ (everything)
Second sync: Transfers only 2 MB of new log entries (delta only)
```

### Basic Syntax

```bash
rsync OPTIONS SOURCE DESTINATION

# Remote locations use user@host:path format
rsync -av /local/dir/ user@remotehost:/remote/dir/
rsync -av user@remotehost:/remote/dir/ /local/dir/
```

### The Most Important rsync Options

| Option | Long form | Purpose |
|---|---|---|
| `-n` | `--dry-run` | Simulate - show what WOULD happen without doing it |
| `-v` | `--verbose` | Show files being transferred |
| `-a` | `--archive` | Archive mode - preserves most file attributes (see below) |
| `-r` | `--recursive` | Recurse into directories |
| `-l` | `--links` | Preserve symbolic links |
| `-p` | `--perms` | Preserve permissions |
| `-t` | `--times` | Preserve modification timestamps |
| `-g` | `--group` | Preserve group ownership |
| `-o` | `--owner` | Preserve file owner (requires root) |
| `-A` | `--acls` | Preserve POSIX Access Control Lists |
| `-X` | `--xattrs` | Preserve extended attributes (includes SELinux contexts) |
| `-z` | `--compress` | Compress data during transfer (useful on slow links) |
| `-P` | | Equivalent to `--partial --progress` (resume + show progress) |
| `--delete` | | Delete files from destination that no longer exist at source |
| `--exclude` | | Exclude files matching a pattern |

> **`-a` archive mode** is shorthand for `-rlptgoD` (recursive, links, permissions, times, group, owner, devices). It does NOT include `-A` (ACLs) or `-X` (SELinux). Add these separately when syncing system directories.

### rsync Examples

```bash
# Synchronise /var/log from local to remote
rsync -av /var/log student@servera:/tmp/

# Synchronise /etc from a remote host to local /backup/
rsync -av root@servera:/etc /backup/

# Always do a dry run first with -n
rsync -avn /var/log/ student@servera:/tmp/logs/

# Full system-level sync preserving SELinux and ACLs
rsync -av -A -X /etc/ root@servera:/configsync/etc/

# Compress during transfer (useful on slow WAN links)
rsync -avz /var/log/ student@servera:/tmp/logs/

# Show progress for large transfers
rsync -av -P /large/dataset/ student@servera:/backup/

# Exclude specific files or patterns
rsync -av --exclude='*.log' --exclude='*.tmp' /source/ /dest/

# Mirror (delete files in dest that are gone from source)
# ALWAYS dry-run first before using --delete
rsync -avn --delete /source/ /dest/       # dry run first
rsync -av --delete /source/ /dest/        # then do it for real

# Local to local sync (no network involved)
rsync -av /var/log/ /mnt/backup/logs/
```

---

## The Trailing Slash Rule

This is the most important thing to understand about rsync.

### Without trailing slash on source: copies the DIRECTORY ITSELF

```bash
rsync -av /var/log hosta:/tmp

# Result: creates /tmp/log/ on hosta
# All log files are inside /tmp/log/
```

### With trailing slash on source: copies the DIRECTORY CONTENTS

```bash
rsync -av /var/log/ hosta:/tmp

# Result: files go directly into /tmp/ on hosta
# No /tmp/log/ subdirectory is created
```

### Visual Comparison

```
SOURCE                  DESTINATION (no slash)    DESTINATION (with slash)
/var/log/               /tmp/log/                 /tmp/
  messages                messages                  messages
  secure                  secure                    secure
  cron                    cron                      cron
```

> **Analogy:** Without the slash, you are copying the folder itself. With the slash, you are copying everything inside the folder. When in doubt, use `-n` (dry run) to check before committing.

> **Bash tab-completion** automatically adds a trailing slash to directory names. This is helpful but can catch you out - be deliberate about whether you want the slash.

---

## SFTP vs SCP vs rsync Decision Guide

```
Do you need to browse the remote filesystem interactively?
    YES -> sftp

Is this a one-off copy of a file or directory?
    YES -> scp (simple, non-interactive)

Are you syncing directories that change over time?
    YES -> rsync (only transfers changes - much faster on repeat runs)

Do you need to preserve SELinux contexts and ACLs?
    YES -> rsync -av -A -X  (or tar --selinux --acls from Chapter 7)

Are you writing a backup script?
    YES -> rsync -av -A -X with cron or systemd timer
```

---

## Quick Reference: All Commands

```bash
# --- sftp ---
sftp remoteuser@remotehost                  # Start interactive session
sftp remoteuser@remotehost:/path/file       # Non-interactive download
# Inside sftp:
# pwd / lpwd       - remote / local working directory
# cd / lcd         - change remote / local directory
# ls / lls         - list remote / local
# get FILE         - download file
# get -r DIR       - download directory
# put FILE         - upload file
# put -r DIR       - upload directory recursively
# exit             - quit session

# --- scp ---
scp FILE user@host:DEST                     # Upload file
scp user@host:FILE DEST                     # Download file
scp -r DIR user@host:DEST                   # Upload directory
scp -r user@host:DIR DEST                   # Download directory
scp -p FILE user@host:DEST                  # Preserve timestamps/perms

# --- rsync ---
rsync -avn SOURCE DEST                      # Dry run FIRST - always
rsync -av SOURCE user@host:DEST             # Push to remote
rsync -av user@host:SOURCE DEST             # Pull from remote
rsync -av -A -X SOURCE DEST                 # With SELinux and ACLs
rsync -avz SOURCE user@host:DEST            # With compression
rsync -av --delete SOURCE DEST              # Mirror (removes extras)
rsync -av --exclude='*.tmp' SOURCE DEST     # Exclude pattern
rsync -av -P SOURCE DEST                    # Show progress + resume
```

---

## Key Configuration Files

| Path | Purpose |
|---|---|
| `/etc/ssh/sshd_config` | SSH server config (controls SFTP subsystem availability) |
| `/etc/ssh/disable_scp` | If this file exists, legacy SCP protocol is blocked (RHEL 10) |

---

## Things to Remember

1. **SFTP runs inside SSH on port 22 - no extra ports or firewall rules needed.** If `ssh user@host` works, `sftp user@host` works. They use the same authentication (password or key pair).

2. **Inside an sftp session, prefix commands with `l` to run them locally.** `pwd` = remote directory. `lpwd` = local directory. `cd` = change remote directory. `lcd` = change local directory.

3. **`scp` now uses SFTP under the hood in RHEL 10.** The old SCP wire protocol had a security vulnerability. The `scp` command still works but is backed by SFTP. Use `sftp` or `rsync` for new scripts.

4. **rsync only transfers what has changed.** The first run is as slow as a full copy. Every subsequent run is much faster because only modified blocks are transferred. This is the key advantage over `scp`.

5. **Always do a dry run before using rsync.** `rsync -avn SOURCE DEST` shows exactly what would be transferred without touching anything. This is non-negotiable before any `--delete` operation.

6. **The trailing slash on the rsync source changes behaviour completely.** `rsync -av /var/log host:/tmp` creates `/tmp/log/`. `rsync -av /var/log/ host:/tmp` puts files directly into `/tmp/`. When in doubt, dry run.

7. **`-a` archive mode does NOT include ACLs or SELinux contexts.** Add `-A` and `-X` explicitly when syncing system directories that need their security labels preserved.

8. **To preserve file ownership at the destination, you must run rsync as root.** If syncing to a remote destination, authenticate as root on the remote host. A non-root rsync will transfer files but cannot set the owner.

9. **`--delete` makes rsync a true mirror - files removed from source are removed from destination.** This is powerful for backups but dangerous if source and destination are confused. Always dry run first.

10. **Use `rsync -avz` over slow network links.** The `-z` flag compresses data in transit. Over a fast LAN it adds CPU overhead with little benefit; over a WAN or VPN it can significantly reduce transfer time.
