# RH134 Chapter 3 - Scheduling User Tasks

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Schedule programs to run in the future, either at a specific time and date or on a recurring basis, as a regular user.

---

## Windows vs Linux: Task Scheduling Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Task Scheduler (one-time) | `at` command + `atd` daemon |
| Task Scheduler (recurring) | `cron` + `crontab` |
| `schtasks /create` | `at TIMESPEC` or `crontab -e` |
| `schtasks /query` | `atq` (deferred) or `crontab -l` (recurring) |
| `schtasks /delete` | `atrm JOBNUMBER` or `crontab -r` |
| Task Scheduler GUI | No GUI equivalent - all text-based |
| Task runs as SYSTEM | Root crontab or `/etc/cron.d/` |
| Task runs as current user | User crontab (`crontab -e`) |

---

## Overview: Two Scheduling Systems

| Tool | Type of Job | Managed with | Daemon |
|---|---|---|---|
| `at` | One-time, future | `at`, `atq`, `atrm` | `atd` |
| `crontab` | Recurring, scheduled | `crontab -e/-l/-r` | `crond` |

---

## Part 1: Deferred (One-Time) Jobs with `at`

### How `at` Works

The `at` command schedules a command to run **once** at a future time. The `atd` daemon monitors the queue and executes jobs when their time arrives.

### Scheduling a Job

```bash
# Interactive input - type commands, then press Ctrl+D to finish
at now +5min
at> date >> /home/student/output.txt
at> <Ctrl+D>

# Better: pipe or redirect commands in
echo "date >> /home/student/output.txt" | at now +5min

# From a script file
at now +1hour < myscript.sh
```

### Time Specification Formats

| Format | Example | Meaning |
|---|---|---|
| `now +Nmin` | `now +30min` | 30 minutes from now |
| `now +Nhour` | `now +2hour` | 2 hours from now |
| `now +Nday` | `now +3day` | 3 days from now |
| `now +Nweek` | `now +1week` | 1 week from now |
| `HH:MM` | `at 14:30` | At 2:30 PM today (or tomorrow if past) |
| `HH:MMpm` | `at 2:30pm` | At 2:30 PM |
| `midnight` | `at midnight` | At 00:00 |
| `noon` | `at noon tomorrow` | At 12:00 tomorrow |
| `teatime` | `at teatime` | At 16:00 (4 PM) |
| `Date + time` | `at 5pm august 3 2025` | At 5 PM on 3 August 2025 |
| `noon +4 days` | | Noon, four days from now |

> Time comes before date in the `at` specification. If only a time is given, it assumes today (or tomorrow if that time has already passed).

### Managing Deferred Jobs

```bash
# List all pending jobs for the current user
atq
at -l          # same as atq

# Inspect what a specific job will run (shows full environment + commands)
at -c JOBNUMBER

# Remove a job
atrm JOBNUMBER

# Example workflow
$ echo "systemctl restart httpd" | at now +10min
job 7 at Thu May 29 14:35:00 2025

$ atq
7  Thu May 29 14:35:00 2025 a student

$ at -c 7        # verify what it will run before it fires

$ atrm 7         # cancel it if no longer needed
```

> **Root vs regular users:** Regular users can only see and manage their own jobs. Root can manage all jobs for all users.

### `atq` Output Format

```
7   Thu May 29 14:35:00 2025 a student
^   ^                         ^ ^
|   |                         | Username
|   |                         Queue letter (a=default)
|   Scheduled execution time
Job number
```

> The queue letter ranges from `a` to `z` and `A` to `Z`. Higher letters run at a lower system priority. The `batch` command uses queue `b` and only runs when system load is low.

---

## Part 2: Recurring Jobs with `cron`

### How `cron` Works

The `crond` daemon reads user crontab files and system crontab files. It checks every minute whether any jobs are due to run. User crontab files are stored in `/var/spool/cron/` and are edited only via the `crontab` command - never directly.

### Managing User Crontabs

| Command | Action |
|---|---|
| `crontab -e` | Open your crontab in the default editor (vim) |
| `crontab -l` | List your current cron jobs |
| `crontab -r` | **Delete ALL your cron jobs (no confirmation!)** |
| `crontab -u USERNAME -l` | List another user's jobs (root only) |
| `crontab -u USERNAME -e` | Edit another user's jobs (root only) |
| `crontab filename` | Replace your crontab with the contents of a file |

> **Warning:** `crontab -r` deletes everything instantly with no confirmation and no undo. Always run `crontab -l` first so you can see what you are about to lose. It is one key away from `crontab -e`.

### Crontab File Format

```
# ┌─ Minute     (0-59)
# │ ┌─ Hour      (0-23)
# │ │ ┌─ Day of month (1-31)
# │ │ │ ┌─ Month   (1-12 or Jan-Dec)
# │ │ │ │ ┌─ Day of week  (0-7 or Sun-Sat, 0 and 7 both = Sunday)
# │ │ │ │ │
# * * * * *  COMMAND
```

### Field Syntax Rules

| Syntax | Meaning | Example |
|---|---|---|
| `*` | Every possible value | `* * * * *` = every minute |
| `N` | A specific value | `30` in minute field = at minute 30 |
| `N-M` | A range (inclusive) | `9-17` in hour field = 9 AM to 5 PM |
| `N,M` | A list | `1,15` in day field = 1st and 15th |
| `N,M-P` | List with ranges | `5,10-13,17` = at 5, 10, 11, 12, 13, 17 |
| `*/N` | Every N units (step) | `*/5` in minute = every 5 minutes |
| `Mon`, `Tue` etc. | 3-letter abbreviation for days | `Mon-Fri` |
| `Jan`, `Feb` etc. | 3-letter abbreviation for months | `Jan,Jul` |

> Days of week: `0` = Sunday, `1` = Monday ... `6` = Saturday, `7` = Sunday (both 0 and 7 are Sunday).

### Crontab Examples

```bash
# Run at 09:00 every day
0 9 * * * /usr/local/bin/daily_report

# Run every 5 minutes
*/5 * * * * /usr/local/bin/check_service

# Run at 2:30 AM every Monday
30 2 * * Mon /usr/local/bin/weekly_cleanup

# Run at midnight on the 1st of every month
0 0 1 * * /usr/local/bin/monthly_report

# Run at 09:00 on 3 February every year
0 9 3 2 * /usr/local/bin/yearly_backup

# Run every 2 minutes (redirect output to a file)
*/2 * * * * /usr/bin/date >> /home/student/timestamps.txt

# Run Monday to Friday, 9 AM to 6 PM, every 5 minutes
*/5 9-18 * * Mon-Fri /usr/local/bin/check_status

# Run at 11:58 PM every weekday
58 23 * * 1-5 /usr/local/bin/daily_report

# Run on the 11th day of every month AND every Friday at 12:15
15 12 11 * Fri /usr/local/bin/command
```

### Important: Day-of-Month AND Day-of-Week Behaviour

When both `Day of month` and `Day of week` are set (neither is `*`), the job runs when **either** condition is true - not both at once.

```bash
# This runs at 12:15 on EVERY 11th of the month
# AND ALSO at 12:15 on EVERY Friday
# It does NOT mean "only on Fridays that fall on the 11th"
15 12 11 * Fri /usr/local/bin/command
```

> This is counterintuitive. If both fields are `*`, neither restriction applies (runs every day). If only one is set, only that restriction applies.

### The `%` Character Trap

An unescaped `%` in a crontab command line is treated as a **newline**, and everything after it is sent to the command as standard input.

```bash
# This FAILS in a crontab - % is interpreted as newline
*/5 * * * * date +%Y-%m-%d >> /tmp/dates.txt

# Correct - escape all % characters
*/5 * * * * date +\%Y-\%m-\%d >> /tmp/dates.txt
```

### Controlling Cron Output (Emails)

If a cron job produces any output (stdout or stderr) and that output is not redirected, `crond` attempts to email it to the job owner.

```bash
# Suppress all output (common for noisy but harmless jobs)
*/5 * * * * /usr/local/bin/check >/dev/null 2>&1

# Capture stdout and stderr to a log file
*/5 * * * * /usr/local/bin/check >> /var/log/check.log 2>&1

# Email output to a specific address (set in crontab header)
MAILTO=admin@example.com
*/5 * * * * /usr/local/bin/check

# Suppress email entirely (set in crontab header)
MAILTO=""
*/5 * * * * /usr/local/bin/check
```

### Crontab Environment Variables

These can be set at the top of the crontab file and apply to all jobs below them:

```bash
# Set inside crontab -e
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com

# Then jobs follow
0 9 * * * /usr/local/bin/daily_report
```

> **Critical:** The environment that cron runs in is minimal - much shorter `PATH` than your interactive shell, and none of your `~/.bashrc` settings. Always use full absolute paths in cron commands (e.g. `/usr/bin/date` not just `date`). This is the most common reason jobs "work in the terminal but fail in cron."

---

## Crontab Reference Examples

```
# Every minute
* * * * * COMMAND

# Every 15 minutes
*/15 * * * * COMMAND

# At 6:00 AM daily
0 6 * * * COMMAND

# At midnight, Monday to Friday
0 0 * * Mon-Fri COMMAND

# At 3:30 AM on the 1st and 15th of every month
30 3 1,15 * * COMMAND

# Hourly from 9 AM to 6 PM on weekdays
0 9-18 * * Mon-Fri COMMAND

# Every 5 minutes during business hours on weekdays
*/5 9-17 * * Mon-Fri COMMAND

# At 11:55 PM on the last day of specified months
55 23 * Jan,Apr,Jul,Oct * COMMAND
```

---

## Access Control: Who Can Use `at` and `cron`

These files control which users are permitted to schedule jobs. This is a security hardening consideration.

| File | Behaviour when it exists |
|---|---|
| `/etc/at.allow` | Only users listed here can use `at` |
| `/etc/at.deny` | All users except those listed can use `at` |
| `/etc/cron.allow` | Only users listed here can use `crontab` |
| `/etc/cron.deny` | All users except those listed can use `crontab` |

> If `at.allow` exists, `at.deny` is ignored. If neither exists, only root can use `at`. The same logic applies to `cron.allow` and `cron.deny`.

---

## Quick Reference: Command Summary

```bash
# --- at (one-time jobs) ---
echo "COMMAND" | at now +10min   # Schedule a job
echo "COMMAND" | at 3pm tomorrow # Schedule for tomorrow afternoon
at teatime < script.sh           # Run script at 4 PM
atq                              # List pending jobs
at -c JOBNUMBER                  # Inspect a job's commands
atrm JOBNUMBER                   # Remove a job

# --- crontab (recurring jobs) ---
crontab -e                       # Edit your crontab (use this to add/change jobs)
crontab -l                       # List your current jobs (do this BEFORE -r!)
crontab -r                       # Delete ALL your jobs (no undo!)
crontab -u USERNAME -l           # List another user's jobs (root only)

# --- Useful one-liners ---
# Check the crond service is running
systemctl status crond

# Watch for cron job output in the system log
tail -f /var/log/cron

# Verify a job ran
tail -f /var/log/messages | grep --line-buffered "CRON"
```

---

## Things to Remember

1. **`at` = run once. `cron` = run repeatedly.** Choose the right tool for the job. One-off tasks use `at`; scheduled recurring tasks use `crontab`.

2. **`crontab -r` has no confirmation and no undo.** Always run `crontab -l` first so your jobs are visible in the terminal scroll buffer before you delete them.

3. **Cron runs in a minimal environment.** Your `~/.bashrc` is not sourced. `PATH` is very short. Always use absolute paths in cron commands (`/usr/bin/date` not `date`). This is the number one reason cron jobs fail.

4. **Unescaped `%` in crontab is treated as a newline.** If you use `date +%Y-%m-%d` in a crontab, escape the percent signs: `date +\%Y-\%m-\%d`.

5. **Day-of-month and Day-of-week use OR logic, not AND.** If both are set (not `*`), the job runs when either condition matches - not when both conditions match simultaneously.

6. **Cron emails output by default.** If a cron job produces output and you don't redirect it, crond tries to email it to the job owner. Always redirect output explicitly: `>> /var/log/job.log 2>&1`.

7. **The `at` queue letter indicates priority.** Default queue is `a`. Higher-letter queues run at lower CPU priority. The `batch` command uses queue `b` and waits for low system load.

8. **`at -c JOBNUMBER` shows the full job contents before it runs.** Use this to verify a scheduled job is going to do what you think before it fires.

9. **Ranges in crontab must go low-to-high.** `Mon-Fri` is valid. `Fri-Mon` is NOT valid and will cause an error. To express a range that wraps around, use a list: `Fri-Sat,Sun-Mon`.

10. **`at` jobs capture the current environment at the time of scheduling.** This means the `PATH` and variables you have when you run `at` are preserved in the job - which is more forgiving than cron, but still worth being aware of.
