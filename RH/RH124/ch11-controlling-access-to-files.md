# Chapter 11 – Controlling Access to Files
## RH124 Student Quick Reference

---

## The Big Picture — How Linux Controls Access

Every file and directory in Linux has:
- An **owner** (one user)
- An **owning group** (one group)
- **Permission bits** for three audiences: owner, group, and everyone else

Linux checks these in order: **owner first, then group, then other**. The first match wins — it does not continue checking.

> **Windows equivalent:** Like NTFS permissions, but simpler. Linux uses a fixed three-tier model (owner/group/other) rather than per-user ACLs. For most situations this is sufficient. For complex multi-user access, tools like ACLs (`setfacl`) extend the model beyond what this chapter covers.

---

## Reading Permission Strings

```bash
ls -l file.txt
-rwxr-x---  1  alice  developers  4096  Jan 15 10:00  file.txt
```

```
- rwx r-x ---
↑ ↑↑↑ ↑↑↑ ↑↑↑
│  │   │   └── Other  (everyone else): no permissions
│  │   └────── Group  (developers): read + execute
│  └────────── Owner  (alice): read + write + execute
└───────────── File type: - = file, d = directory, l = symlink
```

### Permission Meanings

| Symbol | Value | On a File | On a Directory |
|---|---|---|---|
| `r` | 4 | Read the file contents | List the contents (`ls`) |
| `w` | 2 | Modify / write to the file | Create, delete, rename files inside |
| `x` | 1 | Execute the file as a program | Enter the directory (`cd`) and access its contents |
| `-` | 0 | Permission not granted | Permission not granted |

> ⚠️ **Directory execute (`x`) is critical.** Without `x` on a directory you cannot `cd` into it or access anything inside it, even if you have `r`. This trips up Windows users constantly.

---

## Common Permission Patterns

| Octal | Symbolic | Meaning | Typical Use |
|---|---|---|---|
| `777` | `rwxrwxrwx` | Everyone can do everything | Avoid — very insecure |
| `755` | `rwxr-xr-x` | Owner: full, others: read+execute | Public directories, scripts |
| `750` | `rwxr-x---` | Owner: full, group: read+execute, other: none | Group-accessible directories |
| `700` | `rwx------` | Owner only — full control | Private directories |
| `644` | `rw-r--r--` | Owner: read+write, others: read only | Public config files, web content |
| `640` | `rw-r-----` | Owner: read+write, group: read, other: none | Protected config files |
| `600` | `rw-------` | Owner read+write only | Private files, SSH keys |
| `664` | `rw-rw-r--` | Owner+group: read+write, other: read | Shared project files |
| `660` | `rw-rw----` | Owner+group: read+write, other: none | Private shared files |

---

## The Octal Method — Quick Reference

```
Read  = 4
Write = 2
Execute = 1

7 = 4+2+1 = rwx
6 = 4+2   = rw-
5 = 4+  1 = r-x
4 = 4     = r--
0 = 0     = ---
```

**Memory trick:** Start from 7 (full) and subtract what you want to remove.
- Remove execute: 7-1 = 6 (`rw-`)
- Remove write: 7-2 = 5 (`r-x`)
- Remove write+execute: 7-3 = 4 (`r--`)

---

## `chmod` — Change Permissions

### Symbolic Method (Recommended for Changes)

```bash
chmod u+x script.sh             # add execute for owner
chmod g+w shared.txt            # add write for group
chmod o-r private.txt           # remove read from other
chmod a+x script.sh             # add execute for everyone (a = all)
chmod u+x,g-w file.txt          # multiple changes at once
chmod g=rx directory/           # set group to EXACTLY r-x (replaces existing)
chmod o= file.txt               # set other to nothing (removes all other perms)
chmod -R g+rX directory/        # recursive: add r for group, x only for dirs
```

**Who:**
- `u` = user (owner)
- `g` = group
- `o` = other
- `a` = all (u+g+o)

**Operation:**
- `+` = add
- `-` = remove
- `=` = set exactly (replaces all existing for that category)

### Octal Method (Recommended for Setting All Permissions at Once)

```bash
chmod 755 script.sh             # rwxr-xr-x
chmod 644 config.txt            # rw-r--r--
chmod 600 id_rsa                # rw------- (SSH private key)
chmod 750 /home/shared/         # rwxr-x---
chmod -R 755 /var/www/html/     # recursive
```

---

## `chown` — Change Ownership

Only root can change file ownership. The file owner (or root) can change group ownership.

```bash
sudo chown alice file.txt               # change owner to alice
sudo chown alice:developers file.txt    # change owner AND group
sudo chown :developers file.txt         # change group only
sudo chown -R alice:web /var/www/       # recursive — owner and group
sudo chgrp developers file.txt          # change group only (alternative)
```

---

## Viewing Permissions

```bash
ls -l                           # list files with permissions
ls -la                          # include hidden files
ls -ld /path/to/directory/      # show directory itself, not its contents
stat file.txt                   # detailed info including octal permissions
```

### Understanding `ls -l` Output in Full

```
drwxrwsr-t.  2  alice  developers  4096  Jan 15  shared/
│└┬┘└┬┘└┬┘│  │   │         │        │      │      │
│ │  │  │ │  │   │         │        │      │      └─ filename
│ │  │  │ │  │   │         │        │      └─ date modified
│ │  │  │ │  │   │         │        └─ file size (bytes)
│ │  │  │ │  │   │         └─ group owner
│ │  │  │ │  │   └─ user owner
│ │  │  │ │  └─ hard link count
│ │  │  │ └─ SELinux context indicator (.)
│ │  │  └─ other permissions
│ │  └─ group permissions
│ └─ owner permissions
└─ file type (d=dir, -=file, l=symlink)
```

---

## Special Permissions

Three additional bits beyond the standard nine:

| Name | Octal | Symbolic | On a File | On a Directory |
|---|---|---|---|---|
| **setuid** | 4 | `u+s` | Runs as the file's **owner** (not the user who ran it) | No effect |
| **setgid** | 2 | `g+s` | Runs as the file's **group** | New files inherit the **directory's group** |
| **sticky bit** | 1 | `o+t` | No effect | Users can only delete **their own files** |

### Recognising Special Permissions in `ls -l`

```
-rwsr-xr-x    ← setuid:  lowercase 's' in owner execute position
-rwxr-sr-x    ← setgid:  lowercase 's' in group execute position
drwxrwxrwt    ← sticky:  lowercase 't' in other execute position

-rwSr--r--    ← setuid SET but execute NOT set: uppercase 'S' (warning sign)
drwxrwxrwT    ← sticky SET but other execute NOT set: uppercase 'T'
```

### Real-world Examples

```bash
ls -l /usr/bin/passwd
-rwsr-xr-x  root  root  /usr/bin/passwd
# setuid: passwd runs as root even when a regular user runs it
# this is how a regular user can change their own password (which modifies /etc/shadow)

ls -ld /tmp
drwxrwxrwt  root  root  /tmp
# sticky bit: everyone can create files, but only the owner can delete their own files
```

### Setting Special Permissions

```bash
# Symbolic
chmod u+s file.sh               # set setuid
chmod g+s directory/            # set setgid
chmod o+t directory/            # set sticky bit
chmod g+s,o+t directory/        # set both at once

# Octal (4-digit — leading digit is the special bit)
chmod 4755 file.sh              # setuid + rwxr-xr-x
chmod 2770 directory/           # setgid + rwxrwx---
chmod 1777 /tmp                 # sticky + rwxrwxrwx
chmod 3770 directory/           # setgid + sticky + rwxrwx---

# Removing special permissions (must explicitly zero the leading digit)
chmod 0755 file.sh              # remove setuid, keep rwxr-xr-x
chmod 00770 directory/          # remove setgid, keep rwxrwx---
```

---

## `umask` — Default Permissions for New Files

When a file or directory is created, Linux starts with maximum permissions and **subtracts** the umask.

```
New file starts at:     0666  (rw-rw-rw-)
New directory starts at: 0777  (rwxrwxrwx)
```

**Files never start with execute** — you must always add it explicitly. This is a deliberate security feature.

### umask Calculation

```
umask 0022:
  File:       0666 - 0022 = 0644  (rw-r--r--)
  Directory:  0777 - 0022 = 0755  (rwxr-xr-x)

umask 0027:
  File:       0666 - 0027 = 0640  (rw-r-----)
  Directory:  0777 - 0027 = 0750  (rwxr-x---)

umask 0077:
  File:       0666 - 0077 = 0600  (rw-------)
  Directory:  0777 - 0077 = 0700  (rwx------)
```

### umask Commands

```bash
umask                           # show current umask
umask 0027                      # set temporarily (for current shell session only)
umask -S                        # show in symbolic format (e.g. u=rwx,g=rx,o=)
```

### Making umask Permanent

Edit the user's shell config file:

```bash
# For a single user
echo "umask 0027" >> ~/.bashrc

# System-wide (affects all new users)
# Edit /etc/login.defs — UMASK setting
# Or edit /etc/bashrc
```

---

## Common Scenarios and Solutions

### Shared Collaborative Directory

Goal: members of `devteam` group can read/write/create files; others cannot access:

```bash
sudo mkdir /shared/project
sudo chown :devteam /shared/project
sudo chmod 770 /shared/project          # devteam: rwx, other: ---
sudo chmod g+s /shared/project          # new files inherit devteam group
```

### Shared Directory Where Users Cannot Delete Each Other's Files

Add the sticky bit on top:

```bash
sudo chmod 1770 /shared/project         # or: chmod o+t /shared/project
# Result: drwxrws--T (setgid + sticky)
```

### Private SSH Key (Must Be 600 or SSH Refuses to Use It)

```bash
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh/                       # directory also needs to be private
```

### Web Server Content (Apache/Nginx Needs to Read)

```bash
sudo chown -R root:apache /var/www/html/
sudo chmod -R 755 /var/www/html/        # apache user can read + execute
sudo chmod 644 /var/www/html/*.html     # files: read only for apache
```

---

## Windows Comparison

| Windows (NTFS) | Linux | Notes |
|---|---|---|
| File Properties → Security tab | `ls -l` / `stat` | View permissions |
| Full Control | `rwx` (7) | All permissions |
| Modify | `rw-` (6) | Read + Write (no execute) |
| Read & Execute | `r-x` (5) | Read + Execute (no write) |
| Read | `r--` (4) | Read only |
| No Access | `---` (0) | Deny all |
| `icacls file /grant user:F` | `chmod u+rwx file` | Grant permissions |
| `icacls file /deny user:R` | `chmod o-r file` | Remove permissions |
| `takeown /f file` | `chown user file` | Take/change ownership |
| Inherited permissions | `chmod -R` | Recursive apply |
| ACL (per-user granular) | `setfacl` (beyond this chapter) | Advanced permissions |
| "Run as administrator" | `setuid` bit | Run as file owner |
| Folder inheritance | `setgid` bit | Group inherits from directory |
| Shared folder with delete protection | `sticky bit` | Users delete own files only |

---

## Things to Remember

- Linux checks **owner → group → other** in that order — first match wins, it stops there
- **Directory `x` is not optional** — without it you cannot `cd` in or access any files inside
- `chmod =` **replaces** the entire set for that category — use `+` and `-` to make incremental changes
- **`chmod -R`** with symbolic `X` (capital) is safer than `x` — it only sets execute on directories, not all files
- **Sticky bit** (`/tmp` is the classic example) — everyone can write, but you can only delete your own files
- **setuid** is a security risk if misused — any setuid root binary is a potential privilege escalation target
- **umask subtracts from the default** — it cannot add permissions that were not there to begin with
- `chown owner.group` (with a dot) works but use `chown owner:group` (colon) — dots are valid in usernames
- SSH private keys **must** be `600` — SSH will refuse to use them if group or other can read them
- Root ignores permission checks for read and write — only execute still applies to root
