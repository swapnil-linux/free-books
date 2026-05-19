# RH134 Chapter 7 - Archiving Files

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Create compressed archives of files for backup and transfer to other systems.

---

## Windows vs Linux: Archive Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| `.zip` file (WinZip, 7-Zip) | `tar` + `gzip` (`.tar.gz`) |
| `.7z` file (7-Zip) | `tar` + `xz` (`.tar.xz`) - similar compression ratio |
| Right-click "Send to Compressed folder" | `tar -czf archive.tar.gz folder/` |
| WinRAR / 7-Zip extract | `tar -xf archive.tar.gz` |
| "Open archive to see contents" | `tar -tf archive.tar.gz` |
| `Compress-Archive` (PowerShell) | `tar -czf` or `zip` |
| `Expand-Archive` (PowerShell) | `tar -xf` |
| `robocopy` for sync/backup | `rsync` (Chapter 8) |

> **Key difference:** On Linux, archiving (combining files) and compression (reducing size) are separate steps. `tar` handles archiving; `gzip`, `bzip2`, and `xz` handle compression. The `tar` command can invoke them together in one step.

---

## 1. The `tar` Command

### Three Core Operations

Every `tar` command uses exactly one of these action flags:

| Flag | Long form | Action |
|---|---|---|
| `-c` | `--create` | Create a new archive |
| `-t` | `--list` | List contents of an archive |
| `-x` | `--extract` | Extract files from an archive |

### Common Options

| Flag | Long form | Purpose |
|---|---|---|
| `-f` | `--file` | Specify the archive filename (must be followed immediately by the filename) |
| `-v` | `--verbose` | Show files being processed |
| `-p` | `--preserve-permissions` | Preserve original file permissions on extract |
| `-a` | `--auto-compress` | Detect compression from file extension |
| `--selinux` | | Preserve SELinux file contexts |
| `--acls` | | Preserve POSIX Access Control Lists |
| `--xattrs` | | Preserve extended attributes (includes SELinux and ACLs) |

### Compression Options

| Flag | Algorithm | Extension | Speed | Ratio |
|---|---|---|---|---|
| `-z` | gzip | `.tar.gz` or `.tgz` | Fastest | Good |
| `-j` | bzip2 | `.tar.bz2` | Medium | Better |
| `-J` | xz | `.tar.xz` | Slowest | Best |
| `-a` | Auto-detect from extension | any | - | - |

### Real-World Comparison (archiving `/etc`):

| Format | Size | Use case |
|---|---|---|
| `.tar` (no compression) | ~22 MB | When speed matters more than size |
| `.tar.gz` | ~5.5 MB | General use - fastest, most portable |
| `.tar.bz2` | ~4.7 MB | When size matters more than speed |
| `.tar.xz` | ~4.1 MB | Best compression - archival, distribution |

> **Tip:** Previously compressed files (JPEGs, MP4s, RPMs, already-compressed archives) do not compress further significantly. The ratio above applies to typical text/config data like `/etc`.

---

## 2. Creating Archives

### Basic Syntax

```bash
tar -OPTIONS ARCHIVE_FILE SOURCE...
```

> **Critical rule:** `-f` must always be followed immediately by the archive filename. Group other flags before or after `-f`, but the filename must come directly after `-f`.

### Create Examples

```bash
# Uncompressed archive of /etc
tar -cf /tmp/etc.tar /etc

# With verbose output (shows each file as it is added)
tar -cvf /tmp/etc.tar /etc

# gzip compressed
tar -czf /tmp/etc.tar.gz /etc

# bzip2 compressed
tar -cjf /tmp/etc.tar.bz2 /etc

# xz compressed (best ratio)
tar -cJf /tmp/etc.tar.xz /etc

# Auto-detect compression from filename extension (modern approach)
tar -caf /tmp/etc.tar.gz /etc    # uses gzip because of .gz
tar -caf /tmp/etc.tar.xz /etc    # uses xz because of .xz

# Archive multiple sources into one archive
tar -czf /tmp/backup.tar.gz /etc /var/log /home

# Archive specific files only
tar -czf /tmp/logs.tar.gz /var/log/messages /var/log/secure

# Archive with SELinux contexts and ACLs preserved
tar --selinux --acls -czf /tmp/etc-full.tar.gz /etc

# Archive preserving all extended attributes
tar --xattrs -czf /tmp/etc-full.tar.gz /etc
```

> **"Removing leading `/` from member names"** - this message is NOT an error. It is a safety feature. tar strips the leading `/` so extracted files use relative paths. This prevents accidentally overwriting live system files when extracting.

---

## 3. Listing Archive Contents

```bash
# List contents of any archive (tar auto-detects compression)
tar -tf /tmp/etc.tar
tar -tf /tmp/etc.tar.gz     # no need to specify -z
tar -tf /tmp/etc.tar.bz2    # no need to specify -j
tar -tf /tmp/etc.tar.xz     # no need to specify -J

# List with verbose detail (permissions, owner, size, date)
tar -tvf /tmp/etc.tar.gz

# Filter listing to find a specific file
tar -tf /tmp/etc.tar.gz | grep passwd

# Check how many files are in the archive
tar -tf /tmp/etc.tar.gz | wc -l
```

### Checking Uncompressed Size Before Extracting

```bash
# Check expansion size of a gzip archive before extracting
gzip -l /tmp/etc.tar.gz
# Output: compressed  uncompressed  ratio  uncompressed_name
#          5638124     22302720     74.7%  /tmp/etc.tar

# Check expansion size of an xz archive
xz -l /tmp/etc.tar.xz
# Output shows: Compressed, Uncompressed, Ratio, Filename
```

> **Good habit:** Always check the uncompressed size before extracting to a filesystem that may be running low on space. A 200 MB compressed archive could expand to 2 GB.

---

## 4. Extracting Archives

### Basic Extract

```bash
# Extract to the CURRENT DIRECTORY (archive paths are relative)
tar -xf /tmp/etc.tar.gz
tar -xvf /tmp/etc.tar.gz     # verbose - shows each file as extracted

# Extract to a SPECIFIC DIRECTORY
mkdir /restore
tar -xf /tmp/etc.tar.gz -C /restore
# Result: /restore/etc/...

# tar auto-detects compression - no need to specify -z/-j/-J
tar -xf /tmp/etc.tar.gz      # works
tar -xf /tmp/etc.tar.bz2     # works
tar -xf /tmp/etc.tar.xz      # works
```

### Preserve Permissions on Extract

```bash
# For regular users: use -p to preserve original permissions
# (root has -p enabled by default)
tar -xpf /tmp/etc.tar.gz -C /restore

# Without -p: the user's umask modifies extracted file permissions
# With -p: original archived permissions are used exactly
```

### Extract Specific Files Only

```bash
# Extract a single file from the archive (specify path as stored inside)
tar -xf /tmp/etc.tar.gz etc/ssh/sshd_config

# Extract multiple specific files
tar -xf /tmp/etc.tar.gz etc/ssh/sshd_config etc/hosts

# Extract a specific directory from the archive
tar -xf /tmp/etc.tar.gz etc/ssh/

# Dry run - list what would be extracted without extracting
tar -tvf /tmp/etc.tar.gz etc/ssh/
```

---

## 5. Common Workflow: Backup and Restore

### Backing Up a Configuration Directory

```bash
# Create a dated backup of /etc
tar -czf /tmp/etc-backup-$(date +%Y%m%d).tar.gz /etc

# Create with SELinux and ACL preservation for a complete restore
tar --selinux --acls --xattrs -czf \
    /backup/etc-$(date +%Y%m%d).tar.gz /etc

# Verify the backup immediately after creation
tar -tf /tmp/etc-backup-$(date +%Y%m%d).tar.gz | head -20
echo "Archive contains $(tar -tf /tmp/etc-backup-$(date +%Y%m%d).tar.gz | wc -l) files"
```

### Restoring to a Test Location First

```bash
# ALWAYS restore to a test directory first to verify
mkdir /restore-test
tar -xf /tmp/etc-backup-20250620.tar.gz -C /restore-test

# Verify key files are present
ls /restore-test/etc/
ls /restore-test/etc/ssh/

# Compare with live system
diff /restore-test/etc/hosts /etc/hosts
```

### Moving Files Between Systems

```bash
# Pack, ship, unpack pattern
# On source system:
tar -czf /tmp/webfiles.tar.gz /var/www/html/

# Copy to destination (Chapter 8 covers scp/rsync)
scp /tmp/webfiles.tar.gz user@destserver:/tmp/

# On destination system:
mkdir /var/www/html/
tar -xf /tmp/webfiles.tar.gz -C /
# Extracts to /var/www/html/ because paths inside archive are relative
```

---

## 6. Stand-Alone Compression Commands

These compress or decompress a SINGLE file (not multiple files into one archive):

| Compress | Decompress | Extension |
|---|---|---|
| `gzip file` | `gunzip file.gz` | `.gz` |
| `bzip2 file` | `bunzip2 file.bz2` | `.bz2` |
| `xz file` | `unxz file.xz` | `.xz` |

```bash
# Compress a single file (REPLACES the original by default)
gzip largefile.log           # creates largefile.log.gz, removes largefile.log
bzip2 largefile.log          # creates largefile.log.bz2
xz largefile.log             # creates largefile.log.xz

# Keep the original file while compressing
gzip -k largefile.log        # -k = keep original

# Decompress
gunzip largefile.log.gz      # restores largefile.log
bunzip2 largefile.log.bz2
unxz largefile.log.xz

# Check compressed size vs original (without decompressing)
gzip -l largefile.log.gz
xz -l largefile.log.xz
```

> **Stand-alone compression vs `tar`:** Stand-alone `gzip`/`bzip2`/`xz` compress one file to one compressed file. They cannot bundle multiple files into one. To create a compressed bundle of multiple files or a directory, always use `tar` with a compression flag.

---

## 7. `-f` Flag Placement - The Most Common Mistake

```bash
# CORRECT - -f immediately before the archive filename
tar -czf /tmp/archive.tar.gz /etc
tar -c -z -f /tmp/archive.tar.gz /etc   # long form

# WRONG - compression flag after -f
tar -cfz /tmp/archive.tar.gz /etc
# tar treats 'z' as the archive filename, then /tmp/archive.tar.gz as a source to archive
# Result: error or unexpected behaviour
```

> **Rule:** `-f` is special. It must be the last flag in the group, and the archive filename must come directly after it. Think of `-f FILENAME` as an inseparable pair.

---

## Quick Reference: Command Summary

```bash
# --- CREATE ---
tar -cf archive.tar DIR/              # uncompressed
tar -czf archive.tar.gz DIR/          # gzip
tar -cjf archive.tar.bz2 DIR/         # bzip2
tar -cJf archive.tar.xz DIR/          # xz
tar -caf archive.tar.gz DIR/          # auto-detect from extension
tar -czf archive.tar.gz DIR1/ DIR2/   # multiple sources
tar --selinux --acls -czf archive.tar.gz DIR/  # with SELinux + ACLs

# --- LIST ---
tar -tf archive.tar.gz                # list contents
tar -tvf archive.tar.gz               # list with details
tar -tf archive.tar.gz | grep ssh     # search inside
gzip -l archive.tar.gz                # check uncompressed size
xz -l archive.tar.xz                  # check uncompressed size

# --- EXTRACT ---
tar -xf archive.tar.gz                # extract to current dir
tar -xvf archive.tar.gz               # extract verbose
tar -xf archive.tar.gz -C /restore/   # extract to specific dir
tar -xpf archive.tar.gz -C /restore/  # extract preserving permissions
tar -xf archive.tar.gz etc/hosts      # extract single file only

# --- STANDALONE COMPRESSION ---
gzip file                             # compress (replaces original)
gzip -k file                          # compress (keep original)
gunzip file.gz                        # decompress
xz file                               # compress with xz
unxz file.xz                          # decompress xz
```

---

## Things to Remember

1. **`-f` must always be followed immediately by the archive filename.** Group other flags before `-f`, but the filename comes right after. `tar -czf archive.tar.gz /etc` is correct. `tar -cfz archive.tar.gz /etc` is wrong.

2. **`tar` auto-detects compression when listing and extracting.** You do not need to specify `-z`, `-j`, or `-J` when using `-t` or `-x`. `tar -xf archive.tar.xz` works regardless of compression type.

3. **"Removing leading `/` from member names" is a safety message, not an error.** tar strips the leading `/` so extracted files use relative paths. This prevents overwriting live system files during extraction.

4. **SELinux contexts and ACLs are NOT preserved by default.** For a complete, restorable backup use `--selinux --acls --xattrs` when creating the archive. A backup that loses these attributes may not restore correctly on a hardened system.

5. **Extract to a test directory first before overwriting live files.** `tar -xf backup.tar.gz -C /restore-test/` then verify the contents before committing to a live restore.

6. **`-p` preserves permissions for regular users; root has it by default.** Without `-p`, a non-root user's umask modifies extracted permissions. Always use `-p` when the correct permissions matter.

7. **Stand-alone `gzip`/`bzip2`/`xz` compress one file only.** They cannot bundle a directory. For multi-file archives always use `tar` with a compression flag. Running `gzip` on a directory is an error.

8. **`gzip -l` and `xz -l` show uncompressed size without extracting.** Check these before extracting a large archive onto a filesystem with limited free space.

9. **`-a` (auto-compress) detects the algorithm from the filename extension.** `tar -caf backup.tar.gz` uses gzip; `tar -caf backup.tar.xz` uses xz. This makes scripts more readable and easier to change.

10. **When extracting as root, paths inside the archive are applied relative to `-C` or the current directory.** Extracting a backup of `/etc` to `/restore/` gives `/restore/etc/`, not `/etc/`. To restore in-place as root, extract with `-C /` - but be very sure before doing so.
