# Chapter 15 – Monitoring and Managing Linux Processes
## RH124 Student Quick Reference

---

## What Is a Process?

A **process** is a running instance of a program. Every command you run, every service in the background, every daemon — each is a process with its own:

- **PID** — Process ID (unique number, assigned at creation)
- **PPID** — Parent Process ID (who created it)
- **Owner** — which user the process runs as
- **State** — what it is currently doing
- **Resources** — CPU time, memory in use

> **Windows equivalent:** Exactly like a process in Windows Task Manager. The concepts are identical — PID, CPU%, memory usage, owner. The tools differ.

---

## Process States

Every process is always in one of these states:

| Flag | State | Meaning |
|---|---|---|
| `R` | Running | Actively using the CPU, or ready and waiting for CPU time |
| `S` | Sleeping | Waiting for an event (keyboard input, network data, timer) |
| `D` | Uninterruptible sleep | Waiting for I/O (disk/network) — **cannot be killed** |
| `T` | Stopped | Suspended — paused, not running, waiting to be resumed |
| `Z` | Zombie | Process has finished but parent has not cleaned it up yet |

> **D state** is important — a process waiting for a hung disk or NFS mount will be in D state and cannot be killed with any signal, including SIGKILL. Only fixing or removing the underlying I/O issue will clear it.

---

## The Process Lifecycle

```
                    fork()
Parent process ──────────────► Child process (copy of parent)
                                      │
                               exec() │ (load new program)
                                      ▼
                               Running program
                                      │
                               exit() │ (finished)
                                      ▼
                               Zombie (Z) ← waiting for parent to read exit status
                                      │
                               wait() │ (parent reads exit status)
                                      ▼
                               Process gone completely
```

---

## Listing Processes

### `ps` — Snapshot of Processes

```bash
ps                          # your processes in this terminal only
ps aux                      # ALL processes, all users (most common)
ps aux | grep nginx         # find a specific process by name
ps -ef                      # all processes, full format (UNIX style)
ps aux --sort=-%cpu         # sort by CPU usage (highest first)
ps aux --sort=-%mem         # sort by memory usage (highest first)
ps -u alice                 # processes owned by user alice
ps --forest                 # show parent/child tree relationships
ps aux | grep -v grep | grep nginx   # filter without showing grep itself
```

### Understanding `ps aux` Output

```
USER    PID  %CPU  %MEM    VSZ    RSS  TTY   STAT  START   TIME  COMMAND
root      1   0.1   0.2  48740  40532  ?     Ss   16:47   0:01  systemd
student 2496  0.0   0.0 231132   3872  pts/0  R+  16:45   0:00  ps aux
```

| Column | Meaning |
|---|---|
| `USER` | Who owns the process |
| `PID` | Process ID |
| `%CPU` | CPU usage percentage |
| `%MEM` | Memory usage percentage |
| `VSZ` | Virtual memory size (total claimed) |
| `RSS` | Resident Set Size (actual RAM in use) |
| `TTY` | Terminal (`?` = no terminal, a daemon) |
| `STAT` | Process state (R, S, D, T, Z) |
| `TIME` | Total CPU time consumed |
| `COMMAND` | The command that started the process |

### `pgrep` and `pstree`

```bash
pgrep nginx                 # list PIDs of processes named nginx
pgrep -l nginx              # list PID and name
pgrep -u alice              # list PIDs of all processes owned by alice
pstree                      # display process tree for all users
pstree alice                # process tree for one user
pstree -p                   # include PIDs in tree
```

---

## `top` — Live Process Monitor

```bash
top                         # launch top (q to quit)
top -u alice                # show only alice's processes
top -p 1234,5678            # monitor specific PIDs only
```

### `top` Header Lines Explained

```
top - 14:35:22 up 2:41,  3 users,  load average: 0.52, 0.38, 0.25
Tasks: 145 total,   1 running, 144 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.1 us,  0.5 sy,  0.0 ni, 97.2 id,  0.1 wa,  0.0 hi,  0.1 si
MiB Mem :   1705.2 total,   1243.0 free,    420.1 used,    184.4 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   1285.1 avail Mem
```

| Field | Meaning |
|---|---|
| `up 2:41` | System uptime |
| `load average` | 1, 5, 15 minute averages (see below) |
| `us` | User space CPU % |
| `sy` | System (kernel) CPU % |
| `id` | Idle CPU % |
| `wa` | Waiting for I/O % — high value = disk/network bottleneck |
| `buff/cache` | Memory used for caching (can be reclaimed) |

### Useful `top` Keyboard Shortcuts

| Key | Action |
|---|---|
| `q` | Quit |
| `h` | Help |
| `k` | Kill a process (prompts for PID then signal) |
| `r` | Renice a process (change priority) |
| `P` | Sort by CPU usage |
| `M` | Sort by memory usage |
| `u` | Filter by user |
| `1` | Toggle individual CPU core display |
| `d` | Change refresh interval |
| `W` | Save current settings |

---

## Load Average — Understanding System Load

Load average appears in `top`, `uptime`, and `w`:

```bash
uptime
# 14:35:22 up 2:41, 3 users, load average: 0.52, 0.38, 0.25
#                                           1min  5min  15min
```

### How to Interpret It

Load average = number of processes **wanting to run** (waiting for CPU or waiting for I/O) at any moment.

```
Number of CPU cores = the baseline

Load 1.0 on 1 CPU  = 100% busy, nothing waiting
Load 2.0 on 1 CPU  = 100% busy, 1 process waiting  ← overloaded
Load 1.0 on 4 CPUs = 25% busy, plenty of headroom
Load 4.0 on 4 CPUs = 100% busy, nothing waiting     ← fully loaded
Load 8.0 on 4 CPUs = fully loaded, 4 processes waiting ← overloaded
```

**Practical rule:** if load average is consistently **higher than the number of CPU cores**, the system is overloaded.

```bash
# How many CPU cores does this system have?
nproc
grep -c ^processor /proc/cpuinfo
```

> High load with low CPU% (`id` near 100%) usually means **I/O bottleneck** — processes are waiting for disk or network, not CPU. Check `wa` percentage in `top`.

---

## Job Control — Running Commands in Background

Job control lets you manage multiple commands in one terminal session.

```bash
command &                   # start a command in the background
jobs                        # list background jobs in this shell
jobs -l                     # include PIDs

fg                          # bring most recent background job to foreground
fg %2                       # bring job number 2 to foreground

bg                          # resume most recent stopped job in background
bg %2                       # resume job number 2 in background

Ctrl+Z                      # suspend (pause) the foreground process → sends SIGTSTP
Ctrl+C                      # terminate the foreground process → sends SIGINT
Ctrl+\                      # core dump and terminate → sends SIGQUIT
```

### Typical Job Control Workflow

```bash
# Start a long task, then suspend it to do something else
sleep 1000
^Z                          # press Ctrl+Z to suspend
[1]+  Stopped   sleep 1000

# Do other work...
ls -la

# Resume in background
bg %1
[1]+ sleep 1000 &

# Check jobs
jobs
[1]+  Running   sleep 1000 &

# Bring back to foreground
fg %1
```

### `nohup` — Keep Running After Logout

```bash
nohup long-running-script.sh &      # continues even after you log out
nohup long-running-script.sh > output.log 2>&1 &   # with log capture
```

By default output goes to `~/nohup.out`. Used when you need a process to keep running after closing your SSH session.

---

## Signals

Signals are software interrupts sent to processes. Think of them as messages that tell a process to do something.

### The Most Important Signals

| Number | Name | Default Action | Can Be Ignored? | Keyboard |
|---|---|---|---|---|
| 1 | `SIGHUP` | Reload config (or terminate) | Yes | — |
| 2 | `SIGINT` | Terminate gracefully | Yes | `Ctrl+C` |
| 9 | `SIGKILL` | Kill immediately — no cleanup | **No** | — |
| 15 | `SIGTERM` | Terminate gracefully | Yes | — |
| 18 | `SIGCONT` | Resume a stopped process | No | — |
| 19 | `SIGSTOP` | Suspend (pause) process | **No** | — |
| 20 | `SIGTSTP` | Suspend (pause) process | Yes | `Ctrl+Z` |

### The SIGTERM vs SIGKILL Rule

> **Always try SIGTERM first. Only use SIGKILL if SIGTERM fails.**

- **SIGTERM (15)** — polite request to stop. Process can catch it, save state, close files, clean up, then exit. Like asking someone nicely to stop.
- **SIGKILL (9)** — immediate forceful termination by the kernel. Process has no chance to clean up. Can leave temp files, corrupt databases, break open file handles. Like pulling the power cord.

### SIGHUP — The Reload Signal

Many daemons (nginx, sshd, apache) reload their configuration file when they receive SIGHUP, **without restarting**:

```bash
sudo kill -SIGHUP $(pgrep nginx)    # reload nginx config without downtime
sudo kill -HUP $(pgrep sshd)        # reload sshd config
```

---

## Sending Signals

### `kill` — Send Signal by PID

```bash
kill PID                    # send SIGTERM (default) to a process
kill -9 PID                 # send SIGKILL
kill -SIGKILL PID           # same as above — use names not numbers
kill -SIGTERM PID           # explicit SIGTERM
kill -SIGHUP PID            # reload config
kill -SIGSTOP PID           # suspend a process
kill -SIGCONT PID           # resume a suspended process

kill %1                     # send SIGTERM to job number 1
kill -9 %2                  # send SIGKILL to job number 2

kill -l                     # list all available signals
```

### `pkill` — Send Signal by Process Name

```bash
pkill nginx                 # SIGTERM all processes named nginx
pkill -9 nginx              # SIGKILL all processes named nginx
pkill -SIGKILL -u alice     # kill all of alice's processes
pkill -t pts/1              # kill all processes on terminal pts/1
pkill -P 1234               # kill all children of PID 1234
```

### `killall` — Kill All by Name

```bash
killall firefox             # SIGTERM all firefox processes
killall -9 firefox          # SIGKILL all firefox processes
```

### Finding the PID First

```bash
pgrep nginx                             # just PIDs
pgrep -l nginx                          # PIDs and names
ps aux | grep nginx                     # full info
pidof nginx                             # PIDs of processes named exactly "nginx"
```

---

## Process Priority — `nice` and `renice`

Linux schedules CPU time based on **priority**. The `nice` value ranges from **-20** (highest priority) to **+19** (lowest priority). Default is 0.

```
-20 ← highest priority (gets more CPU)
  0 ← default
+19 ← lowest priority (yields CPU to others)
```

```bash
# Start a command with a lower priority (be "nice" to others)
nice -n 10 my-heavy-script.sh
nice -n 19 backup-job.sh           # very low priority

# Start with higher priority (requires root)
sudo nice -n -10 critical-task.sh

# Change priority of a running process
renice -n 10 -p 1234               # lower priority of PID 1234
renice -n 10 -u alice              # lower priority of all of alice's processes
sudo renice -n -5 -p 1234          # raise priority (root required)
```

> Lowering a process's priority with `nice` is useful for backup jobs, batch processing, or anything that should not compete with interactive users.

---

## Logging Out a User's Session

```bash
# Who is logged in and what are they doing?
w
w -u alice

# Find all of a user's processes
pgrep -l -u alice

# Kill all processes for a user (leaves bash session leader alive)
pkill -u alice

# Kill everything including the session (boots them off)
pkill -SIGKILL -u alice

# Kill all processes on a specific terminal
pkill -t tty3
pkill -SIGKILL -t tty3         # also kill the session leader
```

---

## Windows Comparison

| Windows | Linux | Notes |
|---|---|---|
| Task Manager | `top`, `ps aux` | View running processes |
| Task Manager → End Task | `kill PID` / `pkill name` | Terminate a process |
| End Task (force) | `kill -9 PID` | Force kill — no cleanup |
| Task Manager → Details tab | `ps aux` | Full process list with details |
| Ctrl+Alt+Delete | `top` or `ps aux` | View all processes |
| Process priority slider | `nice` / `renice` | CPU scheduling priority |
| Services (services.msc) | `systemctl` (Chapter 16) | Service management |
| Run in background | `command &` | Background execution |
| Minimize to tray (keep running) | `nohup command &` | Keep running after logout |
| %CPU in Task Manager | `%CPU` in `ps aux` / `top` | CPU usage per process |
| Memory (Working Set) | `RSS` in `ps` / `RES` in `top` | Actual RAM used |
| `taskkill /PID 1234` | `kill 1234` | Kill by PID |
| `taskkill /IM notepad.exe` | `pkill notepad` | Kill by name |

---

## Things to Remember

- **`ps aux`** is the go-to command — all processes, all users, full detail
- **SIGTERM first, SIGKILL only if it does not respond** — SIGKILL risks data corruption and leaves cleanup undone
- **`D` state processes cannot be killed** — fix the underlying I/O issue instead
- **Load average > number of CPUs** means the system is overloaded
- **High load + high `wa`** in top means I/O bottleneck, not CPU — look at disk or network
- **`Ctrl+Z` suspends, `Ctrl+C` terminates** — know the difference
- **`nohup`** is essential for SSH sessions — without it your process dies when you disconnect
- **`nice` and `renice`** are often overlooked but invaluable for running heavy batch jobs without impacting other users
- **`pgrep` before `pkill`** — verify which processes you are about to kill before killing them
- **`SIGHUP` reloads config** — most daemons use this to reload without restarting, avoiding downtime
