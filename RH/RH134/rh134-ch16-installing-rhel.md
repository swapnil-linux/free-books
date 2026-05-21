# RH134 Chapter 16 - Installing Red Hat Enterprise Linux

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Install Red Hat Enterprise Linux in package mode, either interactively or by using Kickstart for full automation.

---

## Windows vs Linux: Installation Equivalents

| Windows Concept | Linux / RHEL Equivalent |
|---|---|
| Windows Setup (setup.exe) | Anaconda installer |
| Windows Deployment Services (WDS) | Kickstart over HTTP/FTP/NFS |
| Unattended answer file (`unattend.xml`) | Kickstart file (`kickstart.cfg`) |
| Sysprep + WIM image | Kickstart + ISO + inst.ks= parameter |
| MDT / SCCM OSD Task Sequence | Kickstart `%pre`, `%packages`, `%post` sections |
| `setup /quiet /norestart` | `inst.ks=http://server/kickstart.cfg` |
| Group Policy applied at first login | Kickstart `%post` script |
| Windows ADK (validation tool) | `ksvalidator` (from `pykickstart` package) |
| Installation answer file syntax check | `ksvalidator kickstart.cfg` |

---

## Part 1: Interactive Installation with Anaconda

### Installation Media Types

| Media | Contents | Network needed? |
|---|---|---|
| Binary DVD ISO | Anaconda + full BaseOS + AppStream repos | No |
| Boot ISO | Anaconda only | Yes - packages downloaded over network |

### Minimum Requirements (x86-64)

| Resource | Minimum |
|---|---|
| Disk space | 10 GiB |
| RAM | 1.5 GiB (text mode) / 3 GiB (graphical) |

### Anaconda Installation Summary Screen

Anaconda uses a hub-and-spoke model. All configuration items appear on the Installation Summary screen. Items with an orange warning triangle are mandatory before installation can begin.

| Item | What to configure |
|---|---|
| Keyboard | Layout(s) for the system |
| Language Support | Additional language packs |
| Time & Date | Timezone + NTP configuration |
| Connect to Red Hat | Register with subscription and Red Hat Insights |
| Installation Source | Where to get packages (DVD, HTTP, FTP, NFS) |
| Software Selection | Base environment and additional package groups |
| Installation Destination | Disk selection and partitioning |
| KDUMP | Kernel crash dump (enable/disable) |
| Network & Host Name | NIC configuration and hostname |
| Root Account | Enable/disable root; set password |
| User Creation | Create a non-root admin user |

### RHEL 10 Security Changes in the Installer

| Change | RHEL 9 behaviour | RHEL 10 behaviour |
|---|---|---|
| Root account | Enabled by default | **Disabled by default** |
| Root SSH with password | Allowed if root enabled | **Disabled by default** even when root enabled |
| Admin access | Root or sudo | Non-root user in `wheel` group (mandatory if root disabled) |
| Remote installation GUI | VNC | **RDP** (Remote Desktop Protocol) |

> **Root account disabled by default** means you must either enable root during installation or ensure a non-root user with sudo/wheel membership is created. You cannot log in as root after installation if you skip both steps.

### Remote Installation with RDP (New in RHEL 10)

To perform a graphical installation remotely, add `inst.rdp` to the kernel command line at the GRUB boot menu:

```
# At the GRUB boot menu, press Tab or E to edit
# Append to the 'linux' line:
inst.rdp
```

Then connect from a Windows machine (Remote Desktop), Remmina, or any RDP client.

### The Post-Installation Kickstart File

After every interactive installation, Anaconda saves a Kickstart file capturing all choices made:

```bash
# Location on the newly installed system
cat /root/anaconda-ks.cfg
```

> **This is the recommended starting point for creating Kickstart files.** Install one system interactively, review `/root/anaconda-ks.cfg`, clean and templatise it, then use it for all subsequent deployments. Never start from scratch.

---

## Part 2: Automated Installation with Kickstart

### What Kickstart Does

A Kickstart file contains all answers to the Anaconda installer's questions in a text format. Anaconda reads the file and installs the system without any human interaction. Any question not answered in the Kickstart file causes Anaconda to either stop or prompt interactively.

### Kickstart File Structure

```bash
# version=RHEL10

# ================================================
# COMMAND SECTION (no section header - comes first)
# ================================================
# Installation mode, network, storage, user, auth settings

# ================================================
# %packages SECTION
# ================================================
# List of packages/groups to install

%packages
...
%end

# ================================================
# %addon SECTION (optional)
# ================================================
# Configure specific Anaconda add-ons (e.g. kdump)

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

# ================================================
# %pre SECTION (optional)
# ================================================
# Shell script that runs BEFORE installation begins
# Runs in minimal initramfs environment

%pre
...
%end

# ================================================
# %post SECTION (optional)
# ================================================
# Shell script that runs AFTER packages are installed
# Runs chrooted into the newly installed system

%post
...
%end
```

### Kickstart Section Summary

| Section | When it runs | Environment | Common uses |
|---|---|---|---|
| Command section | Drives installer | N/A (directives) | Storage, network, users, packages |
| `%packages` | During package install | N/A (declarative list) | Specify packages and groups |
| `%addon` | During install | N/A (directives) | Configure Anaconda add-ons |
| `%pre` | Before partitioning | Minimal initramfs bash | Detect hardware, set variables |
| `%post` | After package install | New system (chrooted) | Config files, extra packages, motd |

> Every section must end with `%end`.

### Common Kickstart Commands

```bash
# Installation mode
graphical          # Graphical installer (default)
text               # Text mode (lower resource use)

# Keyboard and language
keyboard --vckeymap=us --xlayouts='us'
lang en_US.UTF-8

# Timezone
timezone Australia/Sydney --utc

# Network
network --bootproto=dhcp --device=enp1s0 --ipv6=auto --activate
network --hostname=server1.example.com

# Installation source
url --url="http://content.example.com/rhel10.0/x86_64/dvd/"

# Storage
zerombr                        # Zero the Master Boot Record
clearpart --all --initlabel    # Remove all existing partitions
autopart                       # Auto-create partitions (LVM by default)
ignoredisk --only-use=vda      # Only use this disk (ignore others)

# Root account
rootpw --iscrypted HASHEDPASSWORD        # Encrypted password
rootpw --iscrypted --allow-ssh HASH      # Also allow root SSH

# User creation
user --groups=wheel --name=admin --password=HASH --iscrypted --gecos="Admin User"

# Post-install
reboot             # Reboot automatically after install (no manual press needed)
```

### `%packages` Section Syntax

```bash
%packages
# Install a single package
vim-enhanced
wget
httpd

# Install a package group (use @ prefix)
@^minimal-environment     # Base environment
@standard                 # Standard group

# Remove a package (use - prefix)
-hyperv*                  # Remove all Hyper-V related packages

# Remove a package group
-@graphical-server-environment
%end
```

> `@^` prefix = environment group (base environment). `@` prefix = package group. No prefix = individual package. `-` prefix = exclude/remove.

### `%post` Section Examples

```bash
%post
# Write an installation timestamp to motd
echo "Deployed via Kickstart on $(date)" > /etc/motd

# Update the man page index
mandb

# Create a custom configuration file
cat > /etc/sysconfig/myapp << 'EOF'
APP_ENV=production
APP_PORT=8080
EOF

# Set SELinux booleans
setsebool -P httpd_can_network_connect on

# Install additional packages (requires network)
dnf install -y tmux htop

# Copy SSH authorized key
mkdir -p /home/admin/.ssh
echo "ssh-rsa AAAAB3..." > /home/admin/.ssh/authorized_keys
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
chown -R admin:admin /home/admin/.ssh
%end
```

### Generating an Encrypted Password for Kickstart

```bash
# Generate a SHA-512 hashed password for use in rootpw or user directives
python3 -c "import crypt; print(crypt.crypt('yourpassword', crypt.mksalt(crypt.METHOD_SHA512)))"

# Or use openssl
openssl passwd -6 'yourpassword'
```

### Validating a Kickstart File

```bash
# Install the validation tool
dnf install pykickstart

# Validate the file syntax
ksvalidator kickstart.cfg
# No output = no errors

# Compare syntax between RHEL versions (useful when migrating from RHEL 8/9)
ksverdiff --from RHEL9 --to RHEL10
```

> **Always validate before deploying.** Syntax errors cause the installation to fail mid-process on a remote server. `ksvalidator` catches most errors before they happen.

### Publishing the Kickstart File

The Kickstart file must be accessible to the Anaconda installer. Common methods:

| Method | Example URL |
|---|---|
| HTTP server | `http://server/ks/kickstart.cfg` |
| FTP server | `ftp://server/ks/kickstart.cfg` |
| NFS share | `nfs:server:/ks/kickstart.cfg` |
| Local hard disk | `hd:sda1:/kickstart.cfg` |
| USB / CD-ROM | `cdrom:/kickstart.cfg` |

```bash
# Quick HTTP publishing with Apache
cp kickstart.cfg /var/www/html/ks/kickstart.cfg
systemctl enable --now httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Verify accessibility
curl http://servera/ks/kickstart.cfg
```

### Starting a Kickstart Installation

At the GRUB boot menu on the target system:

```
1. Boot from RHEL installation media (DVD or boot ISO)
2. At the GRUB menu, select "Install Red Hat Enterprise Linux 10"
3. Press E (or Tab on BIOS) to edit the kernel command line
4. Navigate to the 'linux' line
5. Append:  inst.ks=http://server/ks/kickstart.cfg
6. Press Ctrl+X to boot
```

```bash
# inst.ks= format examples
inst.ks=http://server/dir/file.cfg
inst.ks=ftp://server/dir/file.cfg
inst.ks=nfs:server:/dir/file.cfg
inst.ks=hd:sda1:/file.cfg
inst.ks=cdrom:/file.cfg
```

### Anaconda Troubleshooting During Installation

```
Ctrl+Alt+F1    Switch to Anaconda graphical log (if in graphical mode)
Ctrl+Alt+F2    Switch to a shell (for debugging mid-install)
Ctrl+Alt+F3    Installation log
Ctrl+Alt+F4    Storage log
Ctrl+Alt+F5    Programme log
Ctrl+Alt+F6    Return to main installer screen
```

---

## Complete Kickstart File Example

```bash
# version=RHEL10

# Installation mode
text

# Keyboard and language
keyboard --vckeymap=us --xlayouts='us'
lang en_US.UTF-8

# Timezone
timezone Australia/Sydney --utc

# Network
network --bootproto=dhcp --device=enp1s0 --ipv6=auto --activate
network --hostname=newserver.example.com

# Installation source
url --url="http://content.example.com/rhel10.0/x86_64/dvd/"

# Storage
zerombr
clearpart --all --initlabel
autopart
ignoredisk --only-use=vda

# Root account (enable with SSH access)
rootpw --iscrypted --allow-ssh $y$j9T$HASHEDPASSWORD

# Admin user
user --groups=wheel --name=admin --password=$y$j9T$HASHEDPASSWORD --iscrypted

# Post-install: automatic reboot
reboot

# Packages
%packages
@^minimal-environment
vim-enhanced
wget
-hyperv*
%end

# kdump add-on
%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

# Post-installation script
%post
echo "Deployed by Kickstart on $(date)" > /etc/motd
mandb
%end
```

---

## Quick Reference

```bash
# --- Validate Kickstart ---
dnf install pykickstart
ksvalidator kickstart.cfg
ksverdiff --from RHEL9 --to RHEL10

# --- Generate encrypted password ---
python3 -c "import crypt; print(crypt.crypt('password', crypt.mksalt(crypt.METHOD_SHA512)))"
openssl passwd -6 'password'

# --- Publish via HTTP ---
cp kickstart.cfg /var/www/html/ks/
systemctl enable --now httpd
firewall-cmd --permanent --add-service=http && firewall-cmd --reload
curl http://servera/ks/kickstart.cfg

# --- Read the post-install Kickstart (on newly installed system) ---
cat /root/anaconda-ks.cfg

# --- Use Kickstart Generator ---
# https://access.redhat.com/labs/kickstartconfig

# --- Boot parameter for Kickstart ---
# Append to 'linux' line at GRUB boot menu:
inst.ks=http://server/ks/kickstart.cfg

# --- Remote graphical install (RHEL 10) ---
# Append to 'linux' line:
inst.rdp
```

---

## Key Paths and URLs

| Path / URL | Purpose |
|---|---|
| `/root/anaconda-ks.cfg` | Post-install Kickstart record of all choices made |
| `https://access.redhat.com/labs/kickstartconfig` | Red Hat Kickstart Generator |
| `https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/htmlsingle/automatically_installing_rhel/` | Full Kickstart reference |

---

## Things to Remember

1. **Root is disabled by default in RHEL 10.** If you do not enable root during installation, you cannot log in as root. Always create a `wheel` group user when leaving root disabled.

2. **Root SSH password authentication is disabled by default even when root is enabled.** Use `--allow-ssh` in the `rootpw` Kickstart directive, or configure key-based SSH for root post-install.

3. **`/root/anaconda-ks.cfg` is the best starting point for a Kickstart file.** Install one system interactively, clean up this file, and use it as your template. Never start from scratch.

4. **Every Kickstart section must end with `%end`.** A missing `%end` causes the installer to fail partway through, sometimes without a clear error message.

5. **`ksvalidator` catches syntax errors before deployment.** Run it every time you modify a Kickstart file. Install it with `dnf install pykickstart`.

6. **Missing required Kickstart commands cause interactive prompts.** If a mandatory item (like `Installation Destination`) is absent, Anaconda stops and waits for human input - defeating the purpose of automation.

7. **`autopart` creates LVM-based partitions by default.** The resulting system has a `/boot` partition and LVM logical volumes for `/`, `/home`, and swap. These can be extended later with `lvextend`.

8. **`%post` runs inside the newly installed system (chrooted).** Commands like `dnf install` in `%post` install packages into the new system, not the installer environment. Network must be available for repository access.

9. **The Kickstart Generator at access.redhat.com produces a valid starting template.** Use it when building a Kickstart file for a new scenario you have not configured before.

10. **`reboot` at the end of a Kickstart file removes the need to press a button.** Without it, the installer displays a "Reboot" button and waits. Always include `reboot` in fully automated deployments.
