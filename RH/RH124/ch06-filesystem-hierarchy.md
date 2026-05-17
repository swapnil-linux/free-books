# Chapter 6 – The Linux Filesystem Hierarchy
## RH124 Student Quick Reference

---

## The Biggest Mindset Shift from Windows

**Windows:** Multiple drives with letters — `C:\`, `D:\`, `E:\`
**Linux:** One single tree starting at `/` (called "root")

Everything in Linux — files, folders, drives, USB sticks, printers, network shares — is somewhere in this one tree.

```
/                        ← root of everything (like C:\ but for the whole system)
├── etc/                 ← configuration files
├── home/                ← user home directories
│   └── student/         ← your home directory
├── var/                 ← logs, databases, variable data
│   └── log/             ← system log files
├── usr/                 ← programs and libraries
│   ├── bin/             ← user commands
│   └── sbin/            ← admin commands
├── tmp/                 ← temporary files (cleared on reboot)
├── dev/                 ← device files (disks, terminals, etc.)
├── boot/                ← boot loader and kernel
└── root/                ← home directory of the root (admin) user
```

---

## Key Directories

| Directory | What Is In It | Windows Equivalent |
|---|---|---|
| `/` | Root of the entire tree | `C:\` |
| `/boot` | Kernel and boot files | (part of `C:\Windows`) |
| `/dev` | Device files (disks, USB, terminals) | Device Manager |
| `/etc` | **All system config files** (plain text) | Registry + `C:\Windows\System32\config` |
| `/home` | Regular user home directories | `C:\Users\` |
| `/home/username` | Your personal files and settings | `C:\Users\yourname` |
| `/root` | Root user's home directory | `C:\Users\Administrator` |
| `/tmp` | Temporary files — cleared on reboot | `C:\Windows\Temp` |
| `/var/tmp` | Temp files that survive reboots (30 day purge) | (no direct equivalent) |
| `/usr` | Installed software and libraries | `C:\Windows\System32` + `C:\Program Files` |
| `/usr/bin` | Regular user commands (`ls`, `cp`, etc.) | `C:\Windows\System32\*.exe` |
| `/usr/sbin` | Admin commands (`fdisk`, `sshd`) | |
| `/usr/local` | Software you install manually | `C:\Program Files` |
| `/var` | Logs, databases, mail, print spools | |
| `/var/log` | **System log files** | Event Viewer |
| `/proc` | Virtual — live kernel and process info | Task Manager internals |
| `/sys` | Virtual — kernel hardware parameters | Device Manager internals |

> **Key insight:** In Linux, **all configuration is plain text files** in `/etc`. There is no registry. You can read, edit, back up, and version-control every config file with standard tools.

---

## Navigation Commands

```bash
pwd                         # where am I? (Print Working Directory)
ls                          # what is in here?
ls -la                      # everything, including hidden files, with details
cd /etc                     # go to /etc (absolute path)
cd Documents                # go to Documents subfolder (relative path)
cd ..                       # go up one level
cd ../..                    # go up two levels
cd ~                        # go to my home directory
cd                          # also goes to home directory
cd -                        # go back to where I just was (like Back button)
```

---

## Absolute vs Relative Paths

| Type | Starts With | Example |
|---|---|---|
| **Absolute** | `/` | `/home/student/documents/report.txt` |
| **Relative** | Anything else | `documents/report.txt` or `../report.txt` |

```bash
# These do the same thing if you are in /home/student
cat /home/student/documents/report.txt      # absolute — always works
cat documents/report.txt                    # relative — depends where you are
```

---

## Special Path Shortcuts

| Shortcut | Meaning | Example |
|---|---|---|
| `.` | Current directory | `cp /etc/hosts .` |
| `..` | Parent directory | `cd ..` |
| `~` | Your home directory | `ls ~` |
| `~username` | Another user's home | `ls ~root` |
| `-` | Previous directory | `cd -` |

---

## Hidden Files

- Any file or directory starting with `.` (dot) is hidden
- Hidden from `ls` by default — use `ls -a` to see them
- This is a naming convention, not a filesystem attribute (unlike Windows)

```bash
ls -a ~             # see ALL files including dotfiles in your home
```

Common hidden files in your home directory:
- `.bashrc` — shell configuration
- `.bash_history` — command history
- `.ssh/` — SSH keys and config

---

## Important: `/tmp` Security Note

`/tmp` is **world-writable** — any user can create files there.
This makes it useful, but also a common target for abuse.

- Files not accessed for **10 days** are automatically deleted
- `/var/tmp` is similar but purged after **30 days** and survives reboots
- On hardened servers, `/tmp` is often mounted with `noexec` to prevent running binaries from it

---

## Things to Remember

- There are **no drive letters** in Linux — everything is in one tree
- Drives and USB sticks are **mounted** as folders, not assigned letters
- **All config is plain text files in `/etc`** — readable with `cat`, editable with any text editor
- Case sensitivity is strict: `/etc/Hosts` ≠ `/etc/hosts`
- `/root` (admin home) is separate from `/home` by design — root can always log in even if `/home` is broken
- `/proc` and `/sys` look like directories but are virtual — they show live kernel data, nothing is stored on disk
