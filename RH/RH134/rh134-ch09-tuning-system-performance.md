# RH134 Chapter 9 - Tuning System Performance

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Improve system performance by setting a tuning profile and adjusting the scheduling priority of specific processes.

---

## Windows vs Linux: Performance Tuning Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Power Plans (Balanced / High Performance / Power Saver) | TuneD profiles (`balanced`, `throughput-performance`, `powersave`) |
| `powercfg /setactive` | `tuned-adm profile PROFILENAME` |
| Task Manager priority (Low / Normal / High) | `nice` / `renice` values (-20 to +19) |
| Task Manager "Set Priority" (right-click) | `renice -n VALUE PID` |
| Start a program at Low priority | `nice -n 15 COMMAND` |
| `sysctl` equivalent | `sysctl` (same name, same concept) |
| Windows Performance Monitor counters | `top`, `ps`, `iostat`, `vmstat` |
| Group Policy for performance settings | TuneD profile `tuned.conf` directives |

---

## Part 1: TuneD Tuning Profiles

### What TuneD Does

TuneD (Tuning Daemon) applies sets of kernel parameters, disk scheduler settings, CPU governor settings, and other system parameters as a named profile. When a profile is activated, all its settings are applied immediately. When you switch profiles, TuneD restores the previous settings and applies the new ones.

- **Static tuning** (default): settings are applied once when the profile loads and remain constant
- **Dynamic tuning** (optional): TuneD monitors system activity and continuously adjusts settings in response to changing load

### Checking and Managing the TuneD Service

```bash
# Verify TuneD is installed
dnf list tuned

# Check service status
systemctl status tuned
systemctl is-active tuned
systemctl is-enabled tuned

# Enable and start (if not already running)
systemctl enable --now tuned
```

### `tuned-adm` Command Reference

| Command | Purpose |
|---|---|
| `tuned-adm active` | Show the currently active profile |
| `tuned-adm list` | List all available profiles + current active |
| `tuned-adm profile_info` | Show details of the currently active profile |
| `tuned-adm profile_info PROFILE` | Show details of a specific profile |
| `tuned-adm profile PROFILENAME` | Switch to a different profile (persistent) |
| `tuned-adm recommend` | Recommend a profile based on system characteristics |
| `tuned-adm verify` | Check if current settings match the active profile |
| `tuned-adm off` | Deactivate all TuneD settings (no active profile) |

```bash
# Show currently active profile
tuned-adm active

# List all available profiles
tuned-adm list

# Get a recommendation for this system
tuned-adm recommend

# Switch to a profile
tuned-adm profile throughput-performance
tuned-adm profile virtual-guest
tuned-adm profile balanced

# Verify settings match the profile (useful after manual sysctl changes)
tuned-adm verify

# Show what the current profile sets
tuned-adm profile_info

# Show details of a specific profile
tuned-adm profile_info latency-performance
```

### Available Profiles on RHEL 10

| Profile | Best for |
|---|---|
| `balanced` | General use - balances performance and power |
| `balanced-battery` | Laptops - `balanced` with extra power saving |
| `powersave` | Minimum power consumption |
| `throughput-performance` | General server workloads - maximises throughput |
| `latency-performance` | Time-sensitive workloads needing consistent response |
| `network-latency` | Low-latency network applications |
| `network-throughput` | High-throughput network workloads |
| `accelerator-performance` | GPU / accelerator workloads |
| `hpc-compute` | High-Performance Computing |
| `desktop` | Interactive desktop use |
| `virtual-guest` | Running as a VM guest (default on VMs) |
| `virtual-host` | Running a KVM hypervisor host |
| `aws` | AWS EC2 instances |
| `intel-sst` | Intel Speed Select Technology |
| `optimize-serial-console` | Serial console environments |

> **Choosing a profile:** Run `tuned-adm recommend` first. On a VM, it typically suggests `virtual-guest`. On a database server, `throughput-performance` or `latency-performance` depending on the workload type.

### Profile Configuration Files

| Location | Purpose |
|---|---|
| `/usr/lib/tuned/profiles/` | Vendor-supplied profiles - do not edit |
| `/etc/tuned/profiles/` | Custom admin-created profiles |
| `/usr/lib/tuned/profiles/PROFILE/tuned.conf` | Individual profile config |
| `/etc/tuned/tuned-main.conf` | Global TuneD daemon config (dynamic tuning, update interval) |
| `/var/log/tuned/tuned.log` | TuneD activity log |

```bash
# Read the active profile's config to understand what it changes
cat /usr/lib/tuned/profiles/virtual-guest/tuned.conf
cat /usr/lib/tuned/profiles/throughput-performance/tuned.conf

# See what kernel parameters the profile sets
# Example: vm.swappiness and vm.dirty_ratio
sysctl vm.swappiness
sysctl vm.dirty_ratio
```

### What Profiles Actually Change - A Concrete Example

| Parameter | `virtual-guest` | `latency-performance` | What it controls |
|---|---|---|---|
| `vm.swappiness` | 30 | 10 | Kernel tendency to use swap (lower = prefers RAM) |
| `vm.dirty_ratio` | 30 | 10 | % of RAM that can hold dirty (unwritten) data |

```bash
# See the effect of switching profiles on kernel parameters
sysctl vm.swappiness       # check current value

sudo tuned-adm profile latency-performance

sysctl vm.swappiness       # value has changed to 10 immediately
```

### Enabling Dynamic Tuning (Optional)

Dynamic tuning is disabled by default. To enable it:

```bash
vim /etc/tuned/tuned-main.conf
# Change: dynamic_tuning = 0
# To:     dynamic_tuning = 1
# Optionally: update_interval = 10  (seconds between adjustments)

systemctl restart tuned
```

---

## Part 2: Process Scheduling and Nice Values

### How the Linux Scheduler Works

The Linux kernel scheduler decides which process runs on the CPU and for how long. From RHEL 10, the default scheduler algorithm is **EEVDF** (Earliest Eligible Virtual Deadline First), replacing CFS (Completely Fair Scheduler) from RHEL 9 and earlier.

Key scheduling policies:

| Policy | Name | Used by | Nice values apply? |
|---|---|---|---|
| `SCHED_NORMAL` / `SCHED_OTHER` | Time-sharing (TS) | Regular processes | Yes |
| `SCHED_FIFO` | First In, First Out (FF) | Real-time processes | No |
| `SCHED_RR` | Round Robin (RR) | Real-time processes | No |

> **Real-time processes always run before normal processes**, regardless of nice values. Nice values only affect competition between `SCHED_NORMAL` processes.

### The Nice Value Scale

```
Most aggressive                                      Most yielding
(grabs more CPU)                                   (yields more CPU)

-20  -10   -5    0    5    10   15   19
 |    |     |    |    |    |    |    |
 +----+-----+----+----+----+----+----+
 root only        default       any user can set
                              (up to +19)
```

| Nice value | Behaviour | Who can set it |
|---|---|---|
| `-20` | Highest priority - gets maximum CPU time | Root only |
| `-1` to `-20` | Above default priority | Root only |
| `0` | Default priority (most processes) | All users (default) |
| `+1` to `+19` | Below default priority | Any user (for own processes) |
| `+19` | Lowest priority - gets minimum CPU time | Any user |

> **Memory aid:** "Nice" = polite = yields to others. A very "nice" process (+19) steps back and lets others run first. A "not nice" process (-20) is demanding and grabs CPU aggressively.

### The PR Column in `top` and `ps`

The PR (Priority) column is a calculated representation of scheduling priority:

```
Normal processes:   PR = 20 + nice_value
  nice  0 -> PR 20  (default)
  nice 10 -> PR 30  (lower priority)
  nice -5 -> PR 15  (higher priority)
  nice -20 -> PR  0 (highest)

Real-time processes: PR = -1 - rt_priority  (always negative)
  rt priority 1 -> PR -2
  rt priority 99 -> PR -100 (shown as 'rt' in top)
```

### Viewing Process Priorities

```bash
# Dynamic view with priorities (top interactive)
top
# In top, PR = priority column, NI = nice value column

# Static snapshot of specific processes
ps -o pid,priority,nice,cls,pcpu,comm -C PROCESSNAME

# Examples
ps -o pid,nice,comm -C sha1sum
ps -o pid,priority,nice,cls,pcpu,comm -C httpd

# Sort all processes by nice value (descending - least nice first)
ps -eo pid,priority,nice,cls,pcpu,comm --sort=-nice | head

# Sort by CPU usage (highest first)
ps aux --sort=-%cpu | head

# Show scheduling class (TS = normal, FF = FIFO, RR = round-robin)
ps -o pid,priority,nice,cls,pcpu,comm -C sha1sum
```

### `ps` Output: Scheduling Class (CLS) Values

| CLS | Scheduling policy | Nice applies? |
|---|---|---|
| `TS` | SCHED_NORMAL (time-sharing) | Yes |
| `FF` | SCHED_FIFO (real-time first-in first-out) | No |
| `RR` | SCHED_RR (real-time round-robin) | No |

### Starting a Process with a Specific Nice Value

```bash
# Start a process with the default elevated nice value (+10)
nice COMMAND

# Start a process with a specific nice value
nice -n VALUE COMMAND

# Examples
nice -n 10 sha1sum /dev/zero &    # lower priority than default
nice -n 19 tar -czf backup.tar.gz /data   # lowest priority backup
nice -n -5 COMMAND                 # higher priority (root only)

# Check the nice value was applied
ps -o pid,nice,comm -C PROCESSNAME
```

> **Why run things at lower nice values?** Backups, log compression, and batch jobs compete with user-facing services for CPU. Running them at `nice -n 15` or higher ensures they do not degrade interactive or service response during business hours.

### Changing the Nice Value of a Running Process

```bash
# Change nice value by PID
renice -n VALUE PID
renice -n VALUE PID1 PID2 PID3    # multiple PIDs at once

# Examples
renice -n 10 3456                  # lower priority
renice -n 19 $(pgrep sha1sum)      # by process name lookup
renice -n -5 1234                  # raise priority (root only)

# Change nice value in top (interactive)
# Press R in top, enter PID, enter new nice value
```

> **Unprivileged users can only increase their own processes' nice values.** You cannot lower a nice value you have already raised, and you cannot adjust another user's processes. Only root can lower nice values (increase priority).

### Combining `pgrep` with `renice`

```bash
# Find PIDs by name and renice them in one command
renice -n 10 $(pgrep sha1sum)
renice -n 10 $(pgrep -x httpd)

# Or use ps to find and renice
ps -o pid,nice,comm -C sha1sum
renice -n 5 PID_FROM_ABOVE
```

### A Practical Scheduling Demo

```bash
# Start three CPU-intensive jobs
for i in {1..3}; do sha1sum /dev/zero & done

# Start one at lower priority
nice -n 15 sha1sum /dev/zero &

# View CPU distribution - the nice 15 process gets less time
ps -o pid,pcpu,nice,comm -C sha1sum

# Lower its priority further
renice -n 19 $(pgrep sha1sum | tail -1)

# Clean up all instances
pkill sha1sum
```

---

## Quick Reference: All Commands

```bash
# --- TuneD ---
systemctl status tuned                 # Check service is running
tuned-adm active                       # Show current profile
tuned-adm list                         # List all profiles + current
tuned-adm recommend                    # Recommend profile for this system
tuned-adm profile PROFILENAME          # Switch profile (persistent)
tuned-adm profile_info                 # Current profile details
tuned-adm profile_info PROFILENAME     # Specific profile details
tuned-adm verify                       # Confirm settings match profile
tuned-adm off                          # Deactivate all tuning

# Verify a profile parameter has been applied
sysctl vm.swappiness
sysctl vm.dirty_ratio

# View profile config
cat /usr/lib/tuned/profiles/virtual-guest/tuned.conf

# --- Process priorities ---
top                                    # Dynamic view (PR and NI columns)
# In top: R key = renice interactively

ps -o pid,nice,comm -C PROCESSNAME     # Nice value of specific process
ps -o pid,priority,nice,cls,pcpu,comm -C PROCESSNAME  # Full detail
ps aux --sort=-%cpu | head             # Top CPU consumers
ps -eo pid,priority,nice,cls,pcpu,comm --sort=-nice | head  # Sort by nice

# Start with nice value
nice COMMAND                           # Default +10
nice -n VALUE COMMAND                  # Specific value

# Change running process priority
renice -n VALUE PID                    # By PID
renice -n VALUE $(pgrep PROCESSNAME)   # By name

# Useful combinations
ps -o pid,pcpu,nice,comm $(pgrep sha1sum)  # Stats for named process
pkill PROCESSNAME                          # Terminate all by name
```

---

## Key Configuration Files and Paths

| Path | Purpose |
|---|---|
| `/usr/lib/tuned/profiles/` | Vendor profile definitions (read-only) |
| `/etc/tuned/profiles/` | Custom admin-created profiles |
| `/etc/tuned/tuned-main.conf` | Global TuneD settings (dynamic tuning toggle) |
| `/var/log/tuned/tuned.log` | TuneD activity and change log |

---

## Things to Remember

1. **`tuned-adm profile PROFILENAME` is persistent across reboots.** TuneD is enabled by default and reapplies the profile on every boot. There is no need to add anything to `rc.local` or `cron`.

2. **`tuned-adm recommend` is the right starting point.** It analyses your system (VM vs bare metal, CPU, memory) and suggests the most appropriate profile. Never leave the default without at least checking what it recommended.

3. **`tuned-adm verify` catches drift.** If kernel parameters were changed manually after a profile was applied, `verify` reports the mismatch. Use it to confirm a profile is fully in effect.

4. **Never edit files in `/usr/lib/tuned/profiles/`.** Package updates overwrite them. Create custom profiles in `/etc/tuned/profiles/` instead.

5. **Higher nice value = lower CPU priority = more polite.** Nice -20 grabs maximum CPU. Nice +19 yields to everything else. Default is 0. The scale feels backwards to most people at first - more positive means less priority.

6. **Only root can lower a nice value (increase priority).** Unprivileged users can only raise their own process nice values. They cannot lower them again once raised, and cannot touch other users' processes.

7. **Nice values only apply to `SCHED_NORMAL` processes.** Real-time processes (`SCHED_FIFO`, `SCHED_RR`) run at a higher scheduling class and always preempt normal processes regardless of nice values.

8. **`PR = 20 + nice_value` for normal processes in `top`.** A nice 0 process shows PR 20. A nice 10 process shows PR 30. Real-time processes show negative PR values. This formula makes the `top` priority column readable.

9. **`nice` sets priority at launch; `renice` changes it on a running process.** If you forget to start a backup job at low priority, `renice -n 15 $(pgrep tar)` fixes it without restarting the job.

10. **Always `pkill` test processes after scheduling demos.** `sha1sum /dev/zero &` runs forever at 100% CPU. Multiple instances will saturate all CPU cores. `pkill sha1sum` terminates all of them in one command.
