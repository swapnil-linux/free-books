# Chapter 16 – Controlling Services and Daemons
## RH124 Student Quick Reference

---

## What Is systemd?

**systemd** is the init system and service manager that runs as PID 1 on modern Linux. It is the first process started by the kernel at boot and the last to run at shutdown. Everything else on the system is eventually traced back to systemd.

```
Boot sequence:
  kernel → systemd (PID 1) → starts all services in parallel → system ready
```

> **Windows equivalent:** systemd is like **Windows Service Control Manager (SCM)** combined with **Task Scheduler** and **Event Trigger** rolled into one. The `systemctl` command is the Linux equivalent of the `services.msc` GUI or the `sc.exe` command.

---

## What Is a Daemon?

A **daemon** is a background process that runs without a controlling terminal, usually started at boot and kept running until shutdown. Common examples:

| Daemon | Purpose |
|---|---|
| `sshd` | SSH server (remote access) |
| `httpd` / `nginx` | Web servers |
| `chronyd` | NTP time synchronisation |
| `rsyslog` | System logging |
| `crond` | Scheduled task runner |
| `firewalld` | Firewall management |

> By convention, daemon names end in **`d`** — `sshd`, `httpd`, `chronyd`, `crond`. This is a Unix tradition: the trailing `d` stands for "daemon".

---

## systemd Units — Everything Is a Unit

A **unit** is anything systemd manages. Units are defined in configuration files (called **unit files**) and grouped by type using file extensions.

| Unit Type | Extension | What It Represents |
|---|---|---|
| **Service** | `.service` | A system service or daemon (e.g. `sshd.service`) |
| **Socket** | `.socket` | A socket for on-demand service activation |
| **Path** | `.path` | Triggers a service when a file changes |
| **Timer** | `.timer` | Scheduled tasks (modern replacement for cron) |
| **Target** | `.target` | A group of units (like runlevels) |
| **Mount** | `.mount` | A filesystem mount point |
| **Device** | `.device` | A device file |

When you run a command without specifying the type, systemd assumes `.service`:

```bash
systemctl status sshd             # same as systemctl status sshd.service
```

---

## The Everyday systemctl Commands

### Check Service Status

```bash
systemctl status sshd              # detailed info — is it running? PID? recent logs?
systemctl is-active sshd           # just: active or inactive
systemctl is-enabled sshd          # just: enabled or disabled
systemctl is-failed sshd           # did it fail?
```

### Starting and Stopping

```bash
sudo systemctl start sshd          # start now
sudo systemctl stop sshd           # stop now
sudo systemctl restart sshd        # stop then start (PID changes)
sudo systemctl reload sshd         # reload config WITHOUT stopping (PID stays same)
sudo systemctl reload-or-restart sshd   # reload if possible, otherwise restart
```

### Boot-Time Configuration

```bash
sudo systemctl enable sshd         # start at boot (not now)
sudo systemctl disable sshd        # do not start at boot (but leave running)

# The --now shortcut does both in one command
sudo systemctl enable --now sshd   # start now AND enable at boot
sudo systemctl disable --now sshd  # stop now AND disable at boot
```

> ⚠️ **Enable does not start. Disable does not stop.** These commands affect boot behaviour only. Use `--now` when you want both.

---

## The Four States of a Service

Understanding these states saves hours of confusion:

|  | Running Now? | Starts at Boot? |
|---|---|---|
| `systemctl start` only | ✅ Yes | ❌ No — will not survive reboot |
| `systemctl enable` only | ❌ No | ✅ Yes — on next boot |
| `systemctl enable --now` | ✅ Yes | ✅ Yes |
| `systemctl disable --now` | ❌ No | ❌ No |

---

## Masking — The "Really Disabled" State

**Disable** prevents a service from auto-starting at boot, but you can still start it manually. **Mask** goes further — it makes the service impossible to start at all, even manually, even as a dependency of another service.

```bash
sudo systemctl mask httpd          # cannot be started by anyone or anything
sudo systemctl unmask httpd        # lift the mask
```

### When to Mask

- **Port conflicts** — you have `nginx` installed and want to prevent `httpd` from ever starting and grabbing port 80
- **Security hardening** — services you want to guarantee can never run (telnet, rsh)
- **Preventing accidental starts** — a service another admin might re-enable by accident

> Masking creates a symbolic link from the unit file to `/dev/null` — systemd sees the link and refuses to load the unit.

---

## Listing Services

### Currently Loaded and Active

```bash
systemctl                                    # show ALL active units of any type
systemctl list-units                         # same
systemctl list-units --type=service          # only services
systemctl list-units --type=service --all    # include inactive/dead services
systemctl list-units --type=service --state=failed   # only failed services
systemctl list-units --type=socket           # list socket units
```

### All Installed Units (Even Disabled Ones)

```bash
systemctl list-unit-files                    # every installed unit
systemctl list-unit-files --type=service     # services only
systemctl list-unit-files --state=enabled    # currently enabled services
systemctl list-unit-files --state=masked     # masked services
```

### The Difference

- `list-units` — shows units **loaded in memory** (active or recently active)
- `list-unit-files` — shows **all installed unit files on disk**, whether loaded or not

---

## Understanding `systemctl status` Output

```
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
   Active: active (running) since Thu 2025-05-22 14:47:33 UTC; 2h 48min ago
     Docs: man:sshd(8)
 Main PID: 2212 (sshd)
    Tasks: 1 (limit: 35724)
   Memory: 3.2M (peak: 6.4M)
   CGroup: /system.slice/sshd.service
           └─2212 sshd: /usr/sbin/sshd -D [listener]

May 22 17:33:34 workstation sshd[5124]: Connection closed by 192.168.1.5
```

| Field | Meaning |
|---|---|
| `●` (green) / `○` (white) / `×` (red) | Running / inactive / failed |
| `Loaded` | Path to unit file, and whether enabled at boot |
| `Active` | Current state and how long it has been in that state |
| `Docs` | Where to find more info (man pages, URLs) |
| `Main PID` | Process ID of the main daemon |
| `Tasks` | Number of threads/processes spawned by the service |
| `Memory` | Current and peak memory use |
| `CGroup` | Control group — groups related processes together |
| (last lines) | Recent log entries from this service |

### Status Indicators

| Symbol | Meaning |
|---|---|
| `●` (dot) | Active — running normally |
| `○` (circle) | Inactive (dead) — stopped |
| `×` (cross) | Failed — tried to start but could not |

---

## Dependency Management

```bash
# What does this service depend on?
systemctl list-dependencies sshd

# What depends on this service? (reverse lookup)
systemctl list-dependencies --reverse sshd

# Include dependencies of dependencies (recursive)
systemctl list-dependencies --all sshd
```

> Useful when troubleshooting: if service X fails, `list-dependencies` shows what it was waiting for; `--reverse` shows what else will now fail.

---

## Unit File Locations

systemd searches for unit files in this order (later locations take precedence):

| Location | Purpose |
|---|---|
| `/usr/lib/systemd/system/` | Shipped by RPM packages — **do not edit directly** |
| `/etc/systemd/system/` | Local admin overrides — **edit here** |
| `/run/systemd/system/` | Runtime units (temporary, lost on reboot) |

### Viewing a Unit File

```bash
systemctl cat sshd                 # show the effective unit file + overrides
systemctl show sshd                # show all properties as key=value
systemctl edit sshd                # create/edit an override file for this unit
systemctl edit --full sshd         # edit the whole unit file
```

After editing a unit file, always reload systemd's configuration:

```bash
sudo systemctl daemon-reload       # reread all unit files
sudo systemctl restart sshd        # then restart the affected service
```

---

## Viewing Logs for a Service

systemd includes a centralised logging system called **journald**. To see logs for a specific service:

```bash
journalctl -u sshd                          # all logs for sshd
journalctl -u sshd -f                       # follow (tail -f equivalent)
journalctl -u sshd --since "1 hour ago"     # recent logs only
journalctl -u sshd --since today            # today's logs
journalctl -u sshd -p err                   # error priority or higher
journalctl -u sshd -n 50                    # last 50 lines
journalctl -u sshd -r                       # reverse order (newest first)
journalctl -u sshd -b                       # logs from current boot only
```

> `journalctl` replaces the traditional need to grep through multiple files in `/var/log/`. It is one of the most useful systemd tools for troubleshooting.

---

## System Power and Boot Control

```bash
sudo systemctl reboot              # reboot (same as: reboot)
sudo systemctl poweroff            # power off (same as: shutdown -h now)
sudo systemctl halt                # halt CPU but leave power on
sudo systemctl suspend             # suspend to RAM
sudo systemctl hibernate           # hibernate to disk
systemctl get-default              # show the default target (runlevel)
sudo systemctl set-default multi-user.target   # boot to CLI, not GUI
```

### Common Targets (Roughly Equivalent to Runlevels)

| Target | Purpose | Old Runlevel |
|---|---|---|
| `poweroff.target` | Halt the system | 0 |
| `rescue.target` | Single-user rescue mode | 1 |
| `multi-user.target` | Multi-user text mode | 3 |
| `graphical.target` | Multi-user with GUI | 5 |
| `reboot.target` | Reboot | 6 |

---

## Real-World Workflow Examples

### Installing and Starting a New Service

```bash
sudo dnf install nginx                 # install
sudo systemctl enable --now nginx      # start and enable at boot
sudo systemctl status nginx            # verify
```

### Applying a Configuration Change

```bash
sudo vim /etc/ssh/sshd_config          # edit config
sudo sshd -t                           # test config syntax first (service-specific)
sudo systemctl reload sshd             # apply without disconnecting existing sessions
```

### Troubleshooting a Failed Service

```bash
systemctl status httpd                 # quick overview
journalctl -u httpd -n 50              # recent logs
journalctl -u httpd --since "10 min ago"   # logs around the failure
systemctl list-dependencies httpd      # what it depends on
sudo systemctl reset-failed httpd      # clear failed state
sudo systemctl restart httpd           # try again
```

### Safely Removing a Service

```bash
sudo systemctl disable --now unwanted-service    # stop and disable
sudo systemctl mask unwanted-service             # lock it out permanently
sudo dnf remove unwanted-package                 # uninstall the package
```

---

## Windows Comparison

| Windows | Linux (systemd) | Notes |
|---|---|---|
| `services.msc` | `systemctl list-units --type=service` | List all services |
| `Start-Service svc` / `net start svc` | `systemctl start svc` | Start a service |
| `Stop-Service svc` / `net stop svc` | `systemctl stop svc` | Stop a service |
| Set to Automatic (Properties) | `systemctl enable svc` | Start at boot |
| Set to Disabled (Properties) | `systemctl disable svc` | Do not start at boot |
| `Restart-Service svc` | `systemctl restart svc` | Restart a service |
| `Get-Service svc` | `systemctl status svc` | Check service status |
| Event Viewer | `journalctl -u svc` | View service logs |
| Dependencies tab | `systemctl list-dependencies svc` | Show dependencies |
| `sc.exe create` | systemd unit file in `/etc/systemd/system/` | Define a new service |
| Services.msc → Disabled | `systemctl mask svc` | Prevent any start |
| Task Scheduler | systemd timer units (`.timer`) | Scheduled tasks |
| `shutdown /r` | `systemctl reboot` | Reboot |
| `shutdown /s` | `systemctl poweroff` | Shut down |

---

## Things to Remember

- **`enable` ≠ `start`** — enable affects next boot; start affects right now. Use `--now` for both.
- **`disable` does not stop** a running service — use `--now` to stop as well.
- **`mask` is stronger than `disable`** — mask prevents any start attempt; disable only prevents auto-start.
- **`reload` is better than `restart`** when available — no downtime, existing connections stay alive. Not all services support reload.
- **`restart` changes the PID**, `reload` does not — useful way to verify which one actually happened.
- **Always `daemon-reload`** after editing a unit file — systemd caches unit files in memory.
- **`journalctl -u servicename`** is usually the fastest way to find out why a service failed.
- **The `● ○ ×` symbols** at the start of status output tell you at a glance: green dot = running, white circle = stopped, red cross = failed.
- **Unit files in `/etc/systemd/system/`** override those in `/usr/lib/systemd/system/` — never edit the latter directly, your changes will be wiped on package updates.
- **`systemctl cat sshd`** shows the effective configuration including any overrides, without needing to find the files yourself.
