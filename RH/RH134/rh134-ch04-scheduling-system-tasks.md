# RH134 Chapter 4 - Scheduling System Tasks

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Schedule system programs that must run on a recurring basis to support daemons or operating system functions.

---

## Chapter Overview: Three Mechanisms

| Mechanism | Best for | Config location |
|---|---|---|
| systemd timer units | Modern system tasks, RHEL 10+ preferred | `/etc/systemd/system/` |
| systemd-tmpfiles | Managing temporary files and directories | `/etc/tmpfiles.d/` |
| System cron (`/etc/cron.d/`) | Legacy system tasks, time-sensitive recurring jobs | `/etc/cron.d/` |

> **RHEL 10 shift:** From RHEL 10 onwards, systemd timer units are the preferred replacement for cron in most system scheduling scenarios. Cron remains available and functional, but systemd timers offer better logging, dependency management, and integration with the rest of systemd.

---

## Part 1: Systemd Timer Units

### What Is a Timer Unit?

A systemd timer is a pair of unit files that work together:

| File | Purpose |
|---|---|
| `unitname.timer` | The trigger - defines WHEN to run |
| `unitname.service` | The action - defines WHAT to run |

By convention, the two files share the same base name. The `.timer` file activates the matching `.service` file automatically.

> **Windows comparison:** This is equivalent to Windows Task Scheduler, which also separates the trigger (schedule) from the action (program to run). The key difference is that systemd splits these into two separate files.

### Listing Timer Units

```bash
# List all active/waiting/failed timer units
systemctl list-units -t timer

# List all installed timer unit files (including disabled ones)
systemctl list-unit-files -t timer

# Show detailed status of a specific timer (including next trigger time)
systemctl status logrotate.timer
systemctl status systemd-tmpfiles-clean.timer
```

#### Output Fields

| Field | Meaning |
|---|---|
| `UNIT` | Full name of the unit |
| `LOAD` | Whether the unit definition was loaded |
| `ACTIVE` | High-level state (active, inactive, failed) |
| `SUB` | Detailed state (e.g. `waiting` = scheduled, not yet fired) |
| `DESCRIPTION` | Brief description |

### Starting, Stopping, Enabling, Disabling Timers

```bash
# Start a timer now (does not persist across reboots)
systemctl start logrotate.timer

# Stop a timer
systemctl stop logrotate.timer

# Enable a timer at boot (does not start it now)
systemctl enable logrotate.timer

# Disable a timer at boot (does not stop it now)
systemctl disable logrotate.timer

# Enable AND start in one step (most common)
systemctl enable --now logrotate.timer

# Disable AND stop in one step
systemctl disable --now logrotate.timer
```

### Viewing Timer Unit File Contents

```bash
# View the unit file as systemd loaded it (respects overrides)
systemctl cat sysstat-collect.timer

# View the contents of the service the timer activates
systemctl cat sysstat-collect.service
```

### Timer Unit File Structure

```ini
# /etc/systemd/system/my-task.timer

[Unit]
Description=Run my task on a schedule

[Timer]
OnCalendar=*:00/10          # Every 10 minutes
# OR
OnBootSec=15min             # 15 minutes after boot
OnUnitActiveSec=1d          # Then every 24 hours after last run

[Install]
WantedBy=timers.target
```

### Timer Scheduling Options

| Option | Type | Example | Meaning |
|---|---|---|---|
| `OnCalendar` | Calendar | `*:00/10` | Every 10 minutes |
| `OnCalendar` | Calendar | `daily` | Once per day at midnight |
| `OnCalendar` | Calendar | `Mon *-*-* 03:00:00` | Every Monday at 3 AM |
| `OnBootSec` | Relative | `15min` | 15 minutes after boot |
| `OnUnitActiveSec` | Relative | `1d` | 24 hours after last run |
| `OnUnitActiveSec` | Relative | `30min` | 30 minutes after last run |

#### `OnCalendar` vs `OnUnitActiveSec`

| | `OnCalendar` | `OnUnitActiveSec` |
|---|---|---|
| Fires at | A specific clock time | N time after last run |
| Misses handled | Catches up after downtime | Waits the full interval from next start |
| Good for | Backups, reports, audits | Periodic maintenance, polling |

### `OnCalendar` Time Specification Examples

```
*:00/10           Every 10 minutes (any hour, any day)
*:00/2            Every 2 minutes
hourly            Once per hour (at :00)
daily             Once per day (at 00:00)
weekly            Once per week (Monday 00:00)
monthly           Once per month (1st day, 00:00)
Mon *-*-* 03:00   Every Monday at 3 AM
*-*-1 00:00       1st of every month at midnight
2025-07-* 12:35   Every day of July 2025 at 12:35
```

> Use `systemd-analyze calendar 'EXPRESSION'` to validate a calendar expression and see when it will next fire.

### Modifying a Timer Unit (Override Pattern)

**Never edit files in `/usr/lib/systemd/system/` - package updates will overwrite them.**

```bash
# Step 1: Copy vendor file to admin-controlled location
cp /usr/lib/systemd/system/sysstat-collect.timer \
   /etc/systemd/system/sysstat-collect.timer

# Step 2: Edit the copy in /etc/systemd/system/
vim /etc/systemd/system/sysstat-collect.timer

# Step 3: Tell systemd to re-read unit files (MANDATORY after changes)
systemctl daemon-reload

# Step 4: Enable and start the modified timer
systemctl enable --now sysstat-collect.timer
```

> **Golden rule:** The same file name in `/etc/systemd/system/` always wins over the same file name in `/usr/lib/systemd/system/`. Copy, edit, reload - in that order, every time.

---

## Part 2: Managing Temporary Files with systemd-tmpfiles

### Why This Exists

Daemons and applications create temporary files in `/tmp`, `/run`, and other locations. Without cleanup:
- Disks fill up silently
- Stale lock files prevent services from starting
- Applications fail because expected directories do not exist

`systemd-tmpfiles` provides a structured, configurable way to create, manage, and clean temporary directories and files on a schedule.

### How It Works at Boot

```
system boot
    |
    v
systemd-tmpfiles-setup.service
    |-- reads /usr/lib/tmpfiles.d/*.conf
    |-- reads /run/tmpfiles.d/*.conf
    |-- reads /etc/tmpfiles.d/*.conf
    |-- creates directories, sets permissions, creates symlinks
    v
system running
    |
    v
systemd-tmpfiles-clean.timer  (fires 15 min after boot, then every 24 hours)
    |
    v
systemd-tmpfiles-clean.service
    |-- runs systemd-tmpfiles --clean
    |-- removes files older than the configured age
```

### Configuration File Locations and Precedence

| Location | Managed by | Priority |
|---|---|---|
| `/etc/tmpfiles.d/*.conf` | System administrator | Highest - wins over all others |
| `/run/tmpfiles.d/*.conf` | Runtime daemons | Middle |
| `/usr/lib/tmpfiles.d/*.conf` | RPM packages (vendor) | Lowest - never edit these |

> If two files share the same name, the one in the highest-priority directory wins. Copy vendor files to `/etc/tmpfiles.d/` to customise them safely.

### Configuration File Syntax

Each line uses the format:

```
TYPE  PATH  MODE  UID  GID  AGE  ARGUMENT
```

| Column | Description |
|---|---|
| `TYPE` | Action to take (see type table below) |
| `PATH` | File or directory path to act on |
| `MODE` | Octal permissions (e.g. `0755`, `1777`) |
| `UID` | Owning user (or `-` to leave unchanged) |
| `GID` | Owning group (or `-` to leave unchanged) |
| `AGE` | How old files must be before cleanup (e.g. `10d`, `30s`, `-` = never) |
| `ARGUMENT` | Optional extra (e.g. symlink target) |

### Common Type Values

| Type | Action |
|---|---|
| `d` | Create directory if it does not exist; leave contents alone |
| `D` | Create directory if it does not exist; remove stale contents when cleaning |
| `q` | Same as `d` - quota-aware variant (used for `/tmp`) |
| `f` | Create file if it does not exist |
| `L` | Create a symbolic link |
| `Z` | Recursively restore SELinux contexts, permissions, and ownership |
| `z` | Restore SELinux context and permissions (non-recursive) |

### Configuration Examples

```bash
# Create /run/myapp directory, owned by myapp:myapp, mode 0750
# Never purge automatically (age = -)
d /run/myapp 0750 myapp myapp -

# Create /tmp directory, sticky bit set (1777), clean files older than 10 days
q /tmp 1777 root root 10d

# Clean files older than 5 days from /tmp
q /tmp 1777 root root 5d

# Create /run/momentary, owned by root, mode 0700
# Remove files unused for 30 seconds
d /run/momentary 0700 root root 30s

# Create directory, remove stale contents older than 1 day
D /home/student 0700 student student 1d

# Create a symlink
L /run/fstablink - root root - /etc/fstab
```

### Age Format Reference

| Suffix | Meaning |
|---|---|
| `s` | Seconds |
| `min` | Minutes |
| `h` | Hours |
| `d` | Days |
| `w` | Weeks |
| `-` | Never automatically purge |

### Manual Commands

```bash
# Create configured directories/files NOW (based on a config file)
systemd-tmpfiles --create /etc/tmpfiles.d/myapp.conf

# Clean stale files NOW (based on a config file)
systemd-tmpfiles --clean /etc/tmpfiles.d/myapp.conf

# Test a config file without making changes (dry run)
systemd-tmpfiles --create --dry-run /etc/tmpfiles.d/myapp.conf

# Check exit code: 0 = success, non-zero = error in config
echo $?
```

> **Tip:** When testing new configs, always specify a single file on the command line. Running against all configs at once makes it hard to isolate problems.

### Customising the Cleanup Timer

The default cleanup fires 15 minutes after boot, then every 24 hours:

```bash
# View the default timer configuration
systemctl cat systemd-tmpfiles-clean.timer
```

To change the frequency, copy and override:

```bash
cp /usr/lib/systemd/system/systemd-tmpfiles-clean.timer \
   /etc/systemd/system/systemd-tmpfiles-clean.timer

# Edit and change OnUnitActiveSec to your desired interval
# e.g. OnUnitActiveSec=30min for every 30 minutes

systemctl daemon-reload
```

---

## Part 3: System Cron Jobs with `/etc/cron.d/`

### System vs User Crontab Format

The key difference is one extra field - the **username** the command runs as:

```
# User crontab format (6 fields):
MIN  HR  DOM  MON  DOW  COMMAND

# System crontab format (7 fields - note the USERNAME):
MIN  HR  DOM  MON  DOW  USERNAME  COMMAND
```

> This is the most common mistake students make when writing their first `/etc/cron.d/` file. Forgetting the username field causes the job to fail silently.

### Where to Put System Cron Jobs

| Location | Purpose | Format |
|---|---|---|
| `/etc/cron.d/` | Custom system cron files | 7-field (with username) |
| `/etc/cron.hourly/` | Scripts run every hour | Executable shell scripts (not crontab files) |
| `/etc/cron.daily/` | Scripts run daily via anacron | Executable shell scripts |
| `/etc/cron.weekly/` | Scripts run weekly via anacron | Executable shell scripts |
| `/etc/cron.monthly/` | Scripts run monthly via anacron | Executable shell scripts |
| `/etc/crontab` | Reference only - do not edit | 7-field |

> **Important:** Never edit `/etc/crontab` directly. Place all custom system cron jobs in `/etc/cron.d/` using individual files. This prevents package updates from overwriting your changes and keeps jobs logically organised.

### Creating a System Cron Job

```bash
# Create a file in /etc/cron.d/ (not /etc/crontab!)
vim /etc/cron.d/myapp-job
```

```bash
# /etc/cron.d/myapp-job

# Environment settings (optional but recommended)
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# MIN  HR  DOM  MON  DOW  USERNAME  COMMAND
*/3   *   *   *   *   root      logger "There are $(w -h | wc -l) active users"
0     2   *   *   *   root      /usr/local/bin/nightly-backup >> /var/log/backup.log 2>&1
*/2   *   *   *   *   root      date >> /tmp/timestamps.txt
```

### Anacron: Ensuring Missed Jobs Run

Anacron is responsible for running the `/etc/cron.daily`, `/etc/cron.weekly`, and `/etc/cron.monthly` script directories. Unlike cron, anacron handles systems that are not always on.

| Tool | Behaviour when system was offline |
|---|---|
| cron | Missed jobs are gone - they do not run when system returns |
| anacron | Detects missed jobs and runs them when system comes back up |

#### `/etc/anacrontab` Structure

```bash
# /etc/anacrontab

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
RANDOM_DELAY=45          # Random delay up to 45 min (prevents thundering herd)
START_HOURS_RANGE=3-22   # Only run jobs during these hours

# PERIOD(days)  DELAY(min)  JOB-ID        COMMAND
1               5           cron.daily    nice run-parts /etc/cron.daily
7               25          cron.weekly   nice run-parts /etc/cron.weekly
@monthly        45          cron.monthly  nice run-parts /etc/cron.monthly
```

| Field | Meaning |
|---|---|
| Period | How often to run (in days; `@monthly` is a macro) |
| Delay | Minutes to wait after crond triggers before starting the job |
| Job-ID | Unique name used in log messages |
| Command | Command to run |

#### How Cron and Anacron Work Together

```
crond
  |-- /etc/cron.d/0hourly (runs every hour at :01)
        |-- run-parts /etc/cron.hourly
              |-- 0anacron script
                    |-- /usr/sbin/anacron -s
                          |-- reads /etc/anacrontab
                                |-- runs /etc/cron.daily (if missed or due)
                                |-- runs /etc/cron.weekly (if missed or due)
                                |-- runs /etc/cron.monthly (if missed or due)
```

> The `RANDOM_DELAY=45` in `/etc/anacrontab` prevents all virtual machines on a shared host from hammering storage at the same moment when they all start their daily jobs.

---

## Quick Reference: All Commands

```bash
# --- systemd timers ---
systemctl list-units -t timer              # List active timers
systemctl list-unit-files -t timer         # List all installed timer files
systemctl status logrotate.timer           # Status + next trigger time
systemctl cat sysstat-collect.timer        # View timer unit contents
systemctl enable --now my-task.timer       # Enable and start
systemctl disable --now my-task.timer      # Stop and disable
systemctl daemon-reload                    # Reload after editing unit files (MANDATORY)

# Validate a calendar expression
systemd-analyze calendar '*:00/10'

# --- systemd-tmpfiles ---
systemd-tmpfiles --create /etc/tmpfiles.d/myconf.conf   # Create configured items
systemd-tmpfiles --clean /etc/tmpfiles.d/myconf.conf    # Remove stale files
systemd-tmpfiles --create --dry-run /etc/tmpfiles.d/myconf.conf  # Test only

# --- system cron ---
# Verify a /etc/cron.d/ file is being picked up
ls -la /etc/cron.d/
cat /etc/cron.d/myfile

# Watch for system cron activity in the log
tail -f /var/log/cron
tail -f /var/log/messages | grep --line-buffered "CRON\|crond"
```

---

## File and Directory Summary

| Path | Purpose |
|---|---|
| `/usr/lib/systemd/system/*.timer` | Vendor-supplied timer units - do not edit |
| `/etc/systemd/system/*.timer` | Admin-controlled timer overrides - edit here |
| `/usr/lib/tmpfiles.d/*.conf` | Vendor tmpfiles config - do not edit |
| `/etc/tmpfiles.d/*.conf` | Admin tmpfiles config - create/edit here |
| `/run/tmpfiles.d/*.conf` | Runtime tmpfiles config (daemons) |
| `/etc/cron.d/` | System cron job files (7-field format) |
| `/etc/cron.hourly/` | Executable scripts, run every hour |
| `/etc/cron.daily/` | Executable scripts, run daily via anacron |
| `/etc/cron.weekly/` | Executable scripts, run weekly via anacron |
| `/etc/cron.monthly/` | Executable scripts, run monthly via anacron |
| `/etc/anacrontab` | Anacron configuration |
| `/var/spool/anacron/` | Timestamps of last anacron job runs |
| `/etc/crontab` | Reference only - never edit this directly |

---

## Things to Remember

1. **`systemctl daemon-reload` is mandatory after editing any unit file.** systemd caches unit files - without a reload, your changes are completely ignored. This is the most common reason "I edited the file but nothing changed."

2. **Never edit files in `/usr/lib/systemd/system/` or `/usr/lib/tmpfiles.d/`.** Package updates will overwrite them. Always copy to `/etc/systemd/system/` or `/etc/tmpfiles.d/` first, then edit the copy.

3. **System crontab files need a USERNAME field - user crontabs do not.** `/etc/cron.d/` files use seven fields: `MIN HR DOM MON DOW USERNAME COMMAND`. Forgetting the username is the most common `/etc/cron.d/` mistake.

4. **Never edit `/etc/crontab` directly.** Place custom system jobs in `/etc/cron.d/` as individual files. This keeps jobs organised and safe from package overwrites.

5. **`OnCalendar` fires at a clock time; `OnUnitActiveSec` fires relative to the last run.** Use `OnCalendar` for jobs that must happen at a specific time (backups, reports). Use `OnUnitActiveSec` for jobs that just need to run periodically.

6. **Validate `OnCalendar` expressions before deploying.** Run `systemd-analyze calendar 'EXPRESSION'` to see when the timer will next fire before committing the change.

7. **`/etc/cron.daily/` contains executable scripts, not crontab files.** Drop executable scripts in there; they run via anacron. Do not use crontab syntax in those files.

8. **Anacron runs missed jobs when the system comes back up; cron does not.** If a daily job was missed because the server was rebooted, anacron will run it once when crond's hourly check kicks 0anacron. Cron just skips it forever.

9. **The `RANDOM_DELAY` in `/etc/anacrontab` is intentional.** It staggers job starts across multiple systems to prevent them all hitting shared storage simultaneously. Do not remove it.

10. **The `-` in an age field means "never purge automatically."** Use it for directories that must always exist but whose contents you do not want aged out, such as daemon socket directories.
