# Chapter 12 – Installing and Updating Software with RPM
## RH124 Student Quick Reference

---

## The Big Picture — How Linux Software Installation Works

Coming from Windows, you are used to downloading `.exe` or `.msi` files and running a setup wizard. Linux works very differently — and once you understand it, you will find it significantly better.

```
┌─────────────────────────────────────────────────────────┐
│              Repository (online package store)           │
│   Thousands of pre-built, tested, signed packages        │
└──────────────────────┬──────────────────────────────────┘
                       │  dnf install packagename
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  DNF (package manager)                   │
│  Resolves dependencies, downloads, verifies signatures  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│               RPM (low-level package tool)               │
│        Installs files, registers in local database       │
└─────────────────────────────────────────────────────────┘
```

> **Windows equivalent:** Think of DNF as a combination of Windows Update + Microsoft Store + winget. It finds software, downloads it, checks signatures, resolves what else is needed, and installs everything automatically.

---

## RPM vs DNF — When to Use Which

| Tool | What It Does | When to Use It |
|---|---|---|
| **DNF** | High-level — finds, downloads, resolves dependencies, installs | Day-to-day software management |
| **RPM** | Low-level — queries, verifies, installs single `.rpm` files | Inspecting packages, forensics, offline installs |

> **Rule of thumb:** Use `dnf` to install software. Use `rpm` to inspect and query what is installed.

---

## RPM Package Naming

```
coreutils  -  9.5  -  6.el10  .  x86_64  .rpm
    │           │        │           │
    │           │        │           └─ Architecture (x86_64 = 64-bit Intel/AMD)
    │           │        └─────────── Release (packager's build number)
    │           └──────────────────── Version (upstream software version)
    └──────────────────────────────── Package name
```

Common architectures:
- `x86_64` — 64-bit Intel/AMD (most servers and desktops)
- `aarch64` — 64-bit ARM (Raspberry Pi, AWS Graviton, Apple Silicon)
- `noarch` — architecture-independent (scripts, configs, documentation)

---

## DNF — Day-to-Day Package Management

### Finding Software

```bash
dnf search keyword                  # search by keyword in name and summary
dnf search all 'web server'         # search in name, summary AND description
dnf list 'http*'                    # list packages matching a pattern
dnf list installed                  # list all installed packages
dnf list available                  # list all available packages
dnf info packagename                # show full details about a package
dnf provides /usr/bin/vim           # find which package provides a specific file
dnf provides '*/nmap'               # find package that provides a command
```

### Installing and Removing

```bash
sudo dnf install packagename        # install a package and its dependencies
sudo dnf install pkg1 pkg2 pkg3     # install multiple packages
sudo dnf install ./package.rpm      # install a local .rpm file with dep resolution
sudo dnf remove packagename         # remove a package
sudo dnf remove pkg1 pkg2           # remove multiple packages
sudo dnf autoremove                 # remove packages no longer needed by anything
```

### Updating

```bash
sudo dnf upgrade                    # upgrade all installed packages
sudo dnf upgrade packagename        # upgrade a specific package only
sudo dnf check-update               # list available updates without installing
sudo dnf upgrade --security         # install security updates only
```

### Package Groups

Groups are collections of related packages installed together:

```bash
dnf group list                      # list available groups
dnf group list hidden               # include hidden groups
dnf group info 'Security Tools'     # show what a group contains
sudo dnf group install 'Development Tools'   # install a group
sudo dnf group remove 'Development Tools'    # remove a group
```

### History and Undo

```bash
dnf history                         # list all past transactions
dnf history info 5                  # details of transaction number 5
sudo dnf history undo 5             # undo transaction number 5
sudo dnf history rollback 3         # roll back to the state after transaction 3
```

> **This is a major advantage over Windows:** you can undo an installation completely, including all automatically installed dependencies, by reverting the transaction.

---

## RPM — Querying and Inspecting Packages

### Querying Installed Packages

```bash
rpm -qa                             # list ALL installed packages
rpm -qa | grep ssh                  # find installed packages matching ssh
rpm -q packagename                  # is this package installed? shows version
rpm -qi packagename                 # detailed info about installed package
rpm -ql packagename                 # list ALL files installed by this package
rpm -qc packagename                 # list only config files from this package
rpm -qd packagename                 # list only documentation files
rpm -q --scripts packagename        # show pre/post install scripts
rpm -q --changelog packagename      # show change log
rpm -qf /path/to/file               # which package owns this file?
rpm -q --requires packagename       # what does this package depend on?
rpm -q --provides packagename       # what does this package provide?
```

### Querying a Downloaded `.rpm` File (Before Installing)

Add `-p` to any query to inspect a file rather than the installed database:

```bash
rpm -qlp package.rpm                # list files it would install
rpm -qip package.rpm                # show info about the package file
rpm -q --scripts -p package.rpm     # inspect install scripts before running them
```

> **Security tip:** Always inspect third-party `.rpm` files before installing. The `--scripts` option shows scripts that run as root during install. Malicious packages can and do abuse this.

### Useful Forensic Queries

```bash
# Which package owns a file you are investigating?
rpm -qf /usr/bin/bash

# Was this binary modified after installation? (detects tampering)
rpm -V packagename

# Verify ALL installed packages (slow but thorough)
rpm -Va

# Find all files not owned by any package (potential rogue files)
rpm -Va 2>/dev/null | grep "^?.........."
```

### Understanding `rpm -V` Output

```
S.5....T.  /etc/ssh/sshd_config
│││││││││
││││││││└─ T = timestamp changed
│││││││└── . = (reserved)
││││││└─── . = (reserved)
│││││└──── . = (user ownership OK)
││││└───── . = (group ownership OK)
│││└────── . = (mode / permissions OK)
││└─────── . = (link count OK)
│└──────── 5 = MD5 checksum FAILED (file was modified)
└───────── S = file size changed
```

A dot `.` means that check passed. A letter means something changed.

---

## Repository Management

### Key Locations

| Location | Purpose |
|---|---|
| `/etc/yum.repos.d/` | Repository configuration files (`.repo` files) |
| `/etc/dnf/dnf.conf` | Main DNF configuration |
| `/var/cache/dnf/` | Downloaded package cache |
| `/var/log/dnf.rpm.log` | Full history of installed/removed packages |

### Listing and Managing Repositories

```bash
dnf repolist                        # list enabled repositories
dnf repolist all                    # list all repos including disabled ones
dnf repoinfo repo-id                # detailed info about a specific repo

sudo dnf config-manager --enable repo-id     # enable a repository
sudo dnf config-manager --disable repo-id    # disable a repository

# Temporarily enable/disable for one command
sudo dnf install package --enablerepo=repo-id
sudo dnf upgrade --disablerepo=repo-id
```

### `.repo` File Format

```ini
[repo-id]                           # unique identifier in square brackets
name=Friendly Name                  # human-readable name
baseurl=https://example.com/repo/   # URL to the repository
enabled=1                           # 1=enabled, 0=disabled
gpgcheck=1                          # 1=verify package signatures
gpgkey=file:///etc/pki/rpm-gpg/KEY  # path to the GPG public key
```

### Creating a Custom Repository File

```bash
sudo vim /etc/yum.repos.d/myrepo.repo
```

```ini
[myrepo]
name=My Custom Repository
baseurl=http://internal-server/repo/
enabled=1
gpgcheck=0
```

---

## GPG Signatures — Why They Matter

Every Red Hat package is signed with a GPG key. When you install a package, DNF verifies the signature to confirm:
- The package came from Red Hat (authenticity)
- The package has not been tampered with (integrity)

```bash
# Import a GPG key (always do this before adding a third-party repo)
sudo rpm --import https://example.com/RPM-GPG-KEY-example

# List imported GPG keys
rpm -qa gpg-pubkey*
rpm -qi gpg-pubkey-XXXXXXXX        # details of a specific key

# Never do this in production — bypasses all signature checking
sudo dnf install package --nogpgcheck    # ← dangerous
```

> **Security note:** `--nogpgcheck` is equivalent to downloading and running an `.exe` without checking that it is genuinely from the vendor. Only use it in isolated lab environments.

---

## RHEL Repositories — BaseOS and AppStream

RHEL 10 splits packages across two main repositories:

| Repository | Contains | Lifecycle |
|---|---|---|
| **BaseOS** | Core OS components — kernel, glibc, systemd | Same as RHEL major version (10 years) |
| **AppStream** | Applications, languages, databases, tools | May be shorter than the OS lifecycle |

> On Ubuntu/Debian the equivalent split is `main` (core) and `universe`/`multiverse` (community/extras).

---

## EPEL — Extra Packages for Enterprise Linux

EPEL is a community repository maintained by the Fedora project that provides thousands of extra packages not in RHEL's official repos:

```bash
# Install the EPEL repository
sudo rpm --import https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-10
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

# Then install packages from EPEL normally
sudo dnf install htop
sudo dnf install fail2ban
```

> EPEL packages are community-maintained and not officially supported by Red Hat. Fine for tools and utilities — use caution for production workloads.

---

## Extracting Files from an RPM Without Installing

Useful for recovering a config file or inspecting contents:

```bash
# Extract all files from an RPM
rpm2cpio package.rpm | cpio -idv

# List files in an RPM without extracting
rpm2cpio package.rpm | cpio -tv

# Extract one specific file
rpm2cpio httpd.rpm | cpio -id "./etc/httpd/conf/httpd.conf"
```

---

## Transaction Log and Audit Trail

```bash
# Full history of every package installed, upgraded, or removed
cat /var/log/dnf.rpm.log

# Summarised transaction history
dnf history

# What packages were installed on a specific date?
grep "2026-01-15" /var/log/dnf.rpm.log

# Snapshot all installed packages (useful before major changes)
rpm -qa --qf "%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n" | sort > /tmp/packages-before.txt
```

---

## Windows Comparison

| Windows | Linux (RHEL/DNF) | Notes |
|---|---|---|
| Windows Update | `dnf upgrade` | Update all packages |
| Microsoft Store / winget | `dnf install` | Install software |
| Add/Remove Programs | `dnf remove` | Remove software |
| `.exe` / `.msi` installer | `.rpm` package | Package format |
| Setup wizard with dependency check | `dnf` auto-resolves deps | No wizard needed |
| Windows Installer database | RPM database (`/var/lib/rpm/`) | Tracks installed software |
| Software publisher certificate | GPG signature | Verifies package authenticity |
| Windows Update history | `dnf history` | Full transaction log |
| Undo Windows Update | `dnf history undo N` | Rollback a transaction |
| `winget search term` | `dnf search term` | Find available software |
| `winget show package` | `dnf info package` | Show package details |
| WSUS (internal update server) | Custom `.repo` file | Internal repository |

---

## Things to Remember

- **Always use `dnf`, not `rpm` directly** for installing software — `rpm` does not resolve dependencies
- **`dnf history undo`** is a superpower — you can completely reverse any installation including all its dependencies
- **GPG signatures matter** — never use `--nogpgcheck` on a production system
- **`rpm -qf /path/to/file`** is invaluable for troubleshooting — instantly tells you which package owns any file
- **`rpm -V packagename`** detects if package files have been modified — essential for incident response
- **`/var/log/dnf.rpm.log`** is your audit trail — records every package change with timestamp
- **`yum` still works** on RHEL 9+ — it is just a symlink to `dnf`, not a separate tool
- On **Ubuntu/Debian**, the equivalent of `dnf` is `apt` — most concepts transfer, the commands differ
- **Third-party packages run scripts as root during install** — always inspect with `rpm -q --scripts -p file.rpm` first
