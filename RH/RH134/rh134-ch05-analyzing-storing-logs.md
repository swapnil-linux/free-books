# RH134 Chapter 5 - Analyzing and Storing Logs

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Locate and interpret system logs for troubleshooting purposes, and ensure accurate timestamps for log events.

---

## Windows vs Linux: Logging Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Windows Event Log | systemd journal + `/var/log/` text files |
| Event Viewer | `journalctl` |
| Event ID + Source | Syslog facility + priority |
| `wevtutil` | `journalctl` with filter options |
| Windows Event Forwarding | rsyslog forwarding to remote syslog server |
| `eventvwr.msc` / PowerShell `Get-EventLog` | `journalctl`, `grep` on `/var/log/` |
| Application / System / Security logs | `/var/log/messages`, `/var/log/secure`, `/var/log/cron` |
| Event Log size limits | `journald.conf` `SystemMaxUse`, logrotate config |
| Windows Time (w32tm) | `chronyd` + `timedatectl` |

---

## 1. System Log Architecture

Red Hat Enterprise Linux uses two complementary logging systems that work together:

```
Applications + Kernel
        |
        v
┌─────────────────────────────┐
│     systemd-journald        │  Structured, binary, indexed
│   /run/log/journal/         │  Volatile (lost on reboot) by default
│   (or /var/log/journal/)    │  Captures EVERYTHING from boot onwards
└─────────────┬───────────────┘
              | imjournal module
              v
┌─────────────────────────────┐
│         rsyslog             │  Text files, persistent across reboots
│      /var/log/              │  Sortable by facility + priority
│   /etc/rsyslog.conf         │  Can forward to remote syslog / SIEM
│   /etc/rsyslog.d/*.conf     │
└─────────────────────────────┘
```

> **Why both?** The journal provides fast, structured, indexed searching via `journalctl`. rsyslog writes human-readable text files that survive reboots without special configuration and can be forwarded to centralised log management systems (Splunk, SIEM, etc.).

### Important Log Files in `/var/log/`

| File | Contents |
|---|---|
| `/var/log/messages` | General system messages (most services) |
| `/var/log/secure` | Authentication and security messages (SSH, sudo, su) |
| `/var/log/cron` | Scheduled job (cron and at) activity |
| `/var/log/maillog` | Mail server messages |
| `/var/log/boot.log` | Console messages from system startup |
| `/var/log/dnf.log` | Package installation and update activity |
| `/var/log/audit/audit.log` | SELinux and kernel audit events |

---

## 2. Syslog Facilities and Priorities

Every syslog message is tagged with a **facility** (what type of process sent it) and a **priority** (how severe it is).

### Syslog Facilities

| Code | Facility | Description |
|---|---|---|
| 0 | `kern` | Kernel messages |
| 1 | `user` | User-level messages |
| 2 | `mail` | Mail system |
| 3 | `daemon` | System daemons |
| 4 | `auth` | Authentication and security |
| 5 | `syslog` | Internal syslog messages |
| 9 | `cron` | Clock daemon (cron/at) |
| 10 | `authpriv` | Non-system authorisation messages |
| 11 | `ftp` | FTP protocol |
| 16-23 | `local0`-`local7` | Custom application logging |

### Syslog Priorities (Most to Least Severe)

| Code | Priority | Meaning |
|---|---|---|
| 0 | `emerg` | System is unusable |
| 1 | `alert` | Action must be taken immediately |
| 2 | `crit` | Critical condition |
| 3 | `err` | Non-critical error |
| 4 | `warning` | Warning condition |
| 5 | `notice` | Normal but significant event |
| 6 | `info` | Informational message |
| 7 | `debug` | Debug-level message |

> **Memory tip:** "Every Angry Cat Eats Warm Noodles In Daytime" - Emerg, Alert, Crit, Err, Warning, Notice, Info, Debug.

---

## 3. Rsyslog Configuration

### How rsyslog Rules Work

Rules in `/etc/rsyslog.conf` and `/etc/rsyslog.d/*.conf` follow this format:

```
FACILITY.PRIORITY    DESTINATION
```

| Syntax | Meaning |
|---|---|
| `*.info` | All facilities, info priority and above |
| `kern.*` | All kernel messages, any priority |
| `authpriv.alert` | Auth messages, alert priority and above |
| `*.debug` | ALL messages (debug is the lowest priority) |
| `*.=debug` | ONLY debug priority messages (exact match) |
| `mail.none` | Exclude all mail messages |
| `*.info;mail.none;authpriv.none` | All info+, excluding mail and authpriv |
| `*.*` | Absolutely everything |

### Common rsyslog Destinations

```bash
# Write to a file
authpriv.*              /var/log/secure

# Write to a file without syncing after every write (faster, less safe)
authpriv.*              -/var/log/secure

# Forward to a remote syslog server (UDP)
*.* @192.168.1.100:514

# Forward to a remote syslog server (TCP - more reliable)
*.* @@siem.example.com:514

# Send to all logged-in users
emerg.*                 :omusrmsg:*

# Discard (drop) messages
mail.*                  ~
```

### Creating a Custom rsyslog Rule

```bash
# Create a drop-in config file in /etc/rsyslog.d/
# Name it to control load order (files load alphabetically)

vim /etc/rsyslog.d/debug.conf
```

```bash
# /etc/rsyslog.d/debug.conf
# Log all debug and higher messages to a separate file
*.debug /var/log/messages-debug
```

```bash
# /etc/rsyslog.d/auth-errors.conf
# Log auth alert messages to a dedicated file
authpriv.alert /var/log/auth-errors
```

```bash
# Always restart rsyslog after changing config
systemctl restart rsyslog

# Test by sending a message with logger
logger -p authpriv.alert "Test auth alert message"
logger -p user.debug "Test debug message"

# Verify the message appeared where expected
tail /var/log/auth-errors
tail /var/log/messages-debug
```

### The `logger` Command

Use `logger` to manually send messages to the syslog system - useful for testing rsyslog rules and for adding log entries from scripts.

```bash
# Default: user.notice priority
logger "This is a test message"

# Specify facility and priority
logger -p auth.warning "Suspicious login attempt detected"
logger -p local7.notice "Application started"
logger -p authpriv.alert "Authentication alert test"

# Add a tag to the message
logger -t myapp "Application encountered an error"

# Send from a script with a meaningful priority
logger -p kern.err "Kernel module load failed"
```

---

## 4. The `journalctl` Command

### Basic Usage

```bash
# View all journal entries (opens in less - use q to quit)
journalctl

# View last N entries (default 10)
journalctl -n
journalctl -n 20

# Follow live (like tail -f)
journalctl -f
```

### Filtering by Priority

```bash
# Show errors and above (err, crit, alert, emerg)
journalctl -p err

# Show warnings and above
journalctl -p warning

# Priority names: debug info notice warning err crit alert emerg
# Or by number:   7     6    5      4       3   2    1     0
journalctl -p 3     # same as -p err
```

### Filtering by Time

```bash
# Since a specific date/time
journalctl --since "2025-06-01 09:00:00"
journalctl --since "2025-06-01"         # defaults to 00:00:00

# Until a specific time
journalctl --until "2025-06-01 17:00:00"

# Named values
journalctl --since today
journalctl --since yesterday
journalctl --until tomorrow

# Relative time (from now)
journalctl --since "-1 hour"
journalctl --since "-30 min"
journalctl --since "-2 days"

# Combine since and until
journalctl --since "2025-06-01 09:00" --until "2025-06-01 17:00"
```

### Filtering by Unit or Boot

```bash
# Filter by systemd unit
journalctl -u sshd.service
journalctl -u httpd.service
journalctl -u crond.service

# Current boot only
journalctl -b

# Previous boot
journalctl -b -1

# Boot before that
journalctl -b -2

# List all recorded boot sessions
journalctl --list-boots
```

### Filtering by Journal Fields

```bash
# By specific fields (verbose output reveals available fields)
journalctl _SYSTEMD_UNIT=sshd.service
journalctl _UID=1000
journalctl _PID=1234
journalctl _COMM=sudo

# Combine multiple fields (AND logic)
journalctl _SYSTEMD_UNIT=sshd.service _PID=2188

# Combine unit filter with time filter
journalctl --since "09:00:00" _SYSTEMD_UNIT="sshd.service"
```

### Output Formats

```bash
# Verbose output - shows all journal fields for each entry
journalctl -o verbose

# Short format (default)
journalctl -o short

# JSON output (useful for scripting)
journalctl -o json

# Compact JSON, one line per entry
journalctl -o json-pretty
```

> **Colour coding in journalctl output:**
> - Bold text = notice or warning priority
> - Red text = err priority or higher
> - Normal text = info and below

### Practical journalctl Combinations

```bash
# Show all errors since last boot
journalctl -b -p err

# Follow SSH service live
journalctl -u sshd.service -f

# Show recent auth failures
journalctl -u sshd.service --since "-1 hour" | grep -i fail

# Show all warnings from today for a specific service
journalctl -u httpd.service --since today -p warning

# Check journal size limits
journalctl | grep -E 'Runtime Journal|System Journal'

# Show kernel messages only
journalctl -k

# Show kernel messages from previous boot
journalctl -k -b -1
```

---

## 5. Persistent Journal Configuration

By default, the journal is stored in RAM (`/run/log/journal/`) and is lost on reboot.

### Enabling Persistent Storage

```bash
# Step 1: Create the persistent storage directory
mkdir /var/log/journal

# Step 2: Flush current journal to the new location
journalctl --flush

# Step 3: Verify journal files were created
ls /var/log/journal/
ls /var/log/journal/<hex-directory>/
# Should see: system.journal  user-1000.journal
```

> **How it works:** When `/var/log/journal/` exists and the `Storage=auto` setting (default) is active in `journald.conf`, the journal automatically uses the persistent location. No service restart is needed after creating the directory.

### Configuring Journal Limits

The journal config file is at `/usr/lib/systemd/journald.conf` (vendor default). To customise, copy it first:

```bash
# Copy vendor config to admin-controlled location
cp /usr/lib/systemd/journald.conf /etc/systemd/journald.conf

# Edit the copy
vim /etc/systemd/journald.conf
```

Key settings in `journald.conf`:

| Setting | Default | Description |
|---|---|---|
| `Storage=auto` | auto | `auto` = use persistent if `/var/log/journal/` exists |
| `SystemMaxUse=` | 10% of FS | Max disk space for persistent journal |
| `SystemKeepFree=` | 15% of FS | Min free space to maintain |
| `SystemMaxFileSize=` | 1/8 of MaxUse | Max size of a single `.journal` file |
| `RuntimeMaxUse=` | 10% of FS | Max RAM for volatile journal |

```bash
# After editing journald.conf, restart the service
systemctl restart systemd-journald
```

### Journal Rotation

- Journal rotates automatically each month
- Also rotates when it hits 10% of the filesystem it is on
- Keeps at least 15% of the filesystem free
- View current limits: `journalctl | grep -E 'Runtime Journal|System Journal'`

---

## 6. Maintaining Synchronised Time

### Why Time Sync Matters for Logs

Log timestamps across multiple systems must be accurate for:
- Correlating events during incident investigation
- Meeting compliance requirements (IRAP, PCI DSS, ISO 27001)
- Kerberos authentication (fails if clocks are more than 5 minutes apart)
- TLS certificate validity checking

### `timedatectl` - View and Set Time

```bash
# View current time, timezone, and NTP status
timedatectl

# List all available timezones
timedatectl list-timezones

# Filter timezone list (useful for Australia)
timedatectl list-timezones | grep Australia
timedatectl list-timezones | grep Pacific

# Set the timezone
timedatectl set-timezone Australia/Sydney
timedatectl set-timezone UTC
timedatectl set-timezone America/Jamaica

# Enable NTP synchronisation (enables chronyd)
timedatectl set-ntp true

# Disable NTP (allows manual time setting)
timedatectl set-ntp false

# Set time manually (NTP must be disabled first)
timedatectl set-time "2025-06-01 14:30:00"
```

### `timedatectl` Output Explained

```
               Local time: Mon 2025-06-09 14:30:00 AEST
           Universal time: Mon 2025-06-09 04:30:00 UTC
                 RTC time: Mon 2025-06-09 04:30:00
                Time zone: Australia/Sydney (AEST, +1000)
System clock synchronised: yes
              NTP service: active
          RTC in local TZ: no
```

### `chronyd` - NTP Service

```bash
# Check chronyd service status
systemctl status chronyd

# View NTP sources and their status
chronyc sources
chronyc sources -v    # verbose - includes column headers

# View current tracking information (offset, stratum, etc.)
chronyc tracking

# Force immediate time sync
chronyc makestep
```

### `chronyc sources` Output

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* classroom.example.com         3   9   377   469  +107us[+130us] +/- 13ms
```

| Symbol | Meaning |
|---|---|
| `^` | Server (NTP server source) |
| `=` | Peer |
| `#` | Local reference clock |
| `*` | Currently selected best source |
| `+` | Acceptable source, combined with `*` |
| `-` | Not combined |
| `x` | May be in error |
| `?` | Unusable |

### NTP Stratum Explained

```
Stratum 0  = Atomic clock / GPS receiver (hardware reference)
Stratum 1  = Server directly connected to Stratum 0
Stratum 2  = Server syncing from Stratum 1
Stratum 3  = Server syncing from Stratum 2
...and so on (lower = closer to reference)
```

### `/etc/chrony.conf` Key Directives

```bash
# Use a specific NTP server
server 0.rhel.pool.ntp.org iburst

# Use a pool of NTP servers (recommended for redundancy)
pool 2.rhel.pool.ntp.org iburst

# Use multiple servers (configure 3+ for better accuracy)
server ntp1.example.com iburst
server ntp2.example.com iburst
server ntp3.example.com iburst

# Location of drift file (corrects for local clock inaccuracy)
driftfile /var/lib/chrony/drift
```

```bash
# After editing /etc/chrony.conf, restart chronyd
systemctl restart chronyd
```

---

## Quick Reference: All Commands

```bash
# --- rsyslog ---
systemctl status rsyslog
systemctl restart rsyslog              # Required after config changes
logger "Test message"                  # Send a test syslog message
logger -p auth.warning "Test"          # Specify facility.priority
tail -f /var/log/messages              # Watch live general messages
tail -f /var/log/secure                # Watch live auth messages

# --- journalctl ---
journalctl                             # View all journal entries
journalctl -n 20                       # Last 20 entries
journalctl -f                          # Follow live
journalctl -p err                      # Errors and above
journalctl -p warning                  # Warnings and above
journalctl -b                          # Current boot only
journalctl -b -1                       # Previous boot
journalctl --list-boots                # List all recorded boots
journalctl -u sshd.service             # Filter by unit
journalctl --since today               # Since midnight today
journalctl --since "-1 hour"           # Last hour
journalctl --since "2025-06-01 09:00" --until "2025-06-01 17:00"
journalctl -b -p err                   # Errors from current boot
journalctl -k                          # Kernel messages only
journalctl -o verbose                  # All journal fields

# --- persistent journal ---
mkdir /var/log/journal                 # Enable persistent storage
journalctl --flush                     # Flush to new location

# --- time ---
timedatectl                            # Show current time and NTP status
timedatectl list-timezones | grep Australia
timedatectl set-timezone Australia/Sydney
timedatectl set-ntp true
chronyc sources -v                     # NTP source status
chronyc tracking                       # Sync accuracy details
systemctl status chronyd
```

---

## Key Configuration Files Summary

| File | Purpose |
|---|---|
| `/etc/rsyslog.conf` | Main rsyslog config - do not edit directly for custom rules |
| `/etc/rsyslog.d/*.conf` | Drop-in custom rsyslog rules - create your rules here |
| `/run/log/journal/` | Volatile journal (default, wiped on reboot) |
| `/var/log/journal/` | Persistent journal (create this directory to enable) |
| `/usr/lib/systemd/journald.conf` | Vendor journal config - do not edit |
| `/etc/systemd/journald.conf` | Admin journal config - copy from `/usr/lib/` then edit |
| `/etc/chrony.conf` | NTP client configuration |
| `/var/lib/chrony/drift` | Local clock drift record |

---

## Things to Remember

1. **Two logging systems work together.** `systemd-journald` captures everything in a structured binary format. `rsyslog` writes human-readable text files to `/var/log/` for persistence and forwarding. Both must be healthy for complete logging.

2. **The journal is volatile by default.** `/run/log/journal/` is in RAM and is wiped on reboot. Creating `/var/log/journal/` enables persistence automatically - no config file change needed.

3. **rsyslog rule priority is "this level and above."** `authpriv.warning` captures warning, err, crit, alert, and emerg - not just warnings. Use `authpriv.=warning` to match only warning exactly.

4. **Always restart rsyslog after changing config.** Unlike journald (which can `--flush`), rsyslog reads its config at startup. `systemctl restart rsyslog` is mandatory after any change to `/etc/rsyslog.d/` or `/etc/rsyslog.conf`.

5. **Use `logger` to test rsyslog rules.** After creating a new config file, use `logger -p facility.priority "test message"` to verify it ends up in the expected log file.

6. **`journalctl -b -p err` is your first stop for troubleshooting.** Errors since the last boot, filtered to err and above. Start every troubleshooting session here before diving deeper.

7. **`journalctl --list-boots` requires a persistent journal.** With the volatile default, `--list-boots` only shows the current boot. Historical boot data requires `/var/log/journal/` to exist.

8. **`journalctl -u UNIT` beats grepping text files.** Because the journal is indexed, filtering by unit is near-instant even on systems with millions of log entries. `grep sshd /var/log/messages` reads the entire file; `journalctl -u sshd.service` uses the index.

9. **Time zones affect log readability, not log accuracy.** The journal always stores timestamps in UTC internally. `timedatectl set-timezone` changes how timestamps are displayed, not the underlying stored value. Set the correct zone so logs match your business hours.

10. **Accurate time is a compliance requirement.** PCI DSS Requirement 10.6 mandates time synchronisation for all system components in scope. IRAP requires it for log integrity. `chronyc tracking` showing a large offset is a finding, not just a configuration note.
