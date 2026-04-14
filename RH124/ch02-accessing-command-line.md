# Chapter 2 – Accessing the Command Line
## RH124 Student Quick Reference

---

## Why the Command Line?

Coming from Windows, you are used to doing most things through a GUI.
In Linux, the **command line (shell) is the primary tool** for administration.

Why?
- Faster and more precise than clicking through menus
- Scriptable and automatable — one command can manage thousands of servers
- Works over SSH on remote servers with no GUI
- Every Linux distro has it; not every distro has a GUI

> Think of the Linux command line like PowerShell — but more powerful, and universally available.

---

## The Shell Prompt

```
username@hostname:~$        ← regular user
root@hostname:~#            ← root (administrator) — the # is a warning
```

| Part | Meaning |
|---|---|
| `username` | Who you are logged in as |
| `hostname` | The computer you are on |
| `~` | Current directory (~ means home directory) |
| `$` | Regular user prompt |
| `#` | Root (admin) prompt — elevated privileges, be careful |

---

## Command Structure

```
command  [options]  [arguments]
   ↑          ↑          ↑
what to do  how to do it  what to do it to
```

**Examples:**
```bash
ls                      # list files — no options, no arguments
ls -l                   # list files — long format option
ls -l /etc              # list files in /etc — option + argument
ls -la /etc             # multiple options can be combined
```

---

## Essential Commands

| Command | What It Does | Windows Equivalent |
|---|---|---|
| `pwd` | Show current directory | `cd` with no arguments |
| `ls` | List files | `dir` |
| `ls -la` | List all files, long format | `dir /a` |
| `cd /path` | Change directory | `cd C:\path` |
| `cd ..` | Go up one level | `cd ..` |
| `clear` | Clear the screen | `cls` |
| `date` | Show date and time | `date /t` + `time /t` |
| `who` | Show logged-in users | `query user` |
| `uptime` | System uptime and load | Task Manager → Performance |
| `passwd` | Change your password | `net user username *` |
| `cat file` | Print file to screen | `type file` |
| `less file` | Page through a file | `more file` |
| `head -n 10 file` | First 10 lines | (no direct equivalent) |
| `tail -n 10 file` | Last 10 lines | (no direct equivalent) |
| `tail -f file` | Follow a file live | (no direct equivalent) |
| `wc file` | Count lines/words/bytes | (no direct equivalent) |
| `file filename` | Identify file type | (no direct equivalent) |
| `history` | Show command history | `doskey /history` |
| `exit` | Log out / close session | `exit` |

---

## Keyboard Shortcuts

These work in virtually every Linux terminal — learn them, they save enormous time:

| Shortcut | What It Does |
|---|---|
| `Tab` | Auto-complete command, filename, or option |
| `Tab Tab` | Show all possible completions |
| `↑ / ↓` | Navigate through command history |
| `Ctrl+A` | Jump to beginning of line |
| `Ctrl+E` | Jump to end of line |
| `Ctrl+U` | Delete from cursor to beginning of line |
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+W` | Delete the previous word |
| `Ctrl+LeftArrow` | Jump left one word |
| `Ctrl+RightArrow` | Jump right one word |
| `Ctrl+R` | Search command history interactively |
| `Ctrl+C` | Cancel / kill current command |
| `Ctrl+D` | Log out (or end input) |
| `Ctrl+L` | Clear screen |
| `Esc+.` or `Alt+.` | Paste the last argument of the previous command |

> **Pro tip:** `Tab` completion is your best friend. If you are typing a long filename or path, press Tab — the shell will complete it for you. If nothing happens, press Tab again to see all options.

---

## Command History

```bash
history                 # show numbered list of previous commands
!42                     # re-run command number 42 from history
!ls                     # re-run the most recent command starting with "ls"
!!                      # re-run the very last command
sudo !!                 # re-run last command with sudo (very useful)
Ctrl+R                  # type to search history interactively
```

> History is saved in `~/.bash_history`. Use `↑` arrow to cycle through recent commands.

---

## Writing Long Commands Across Multiple Lines

Use `\` (backslash) at the end of a line to continue on the next line:

```bash
cp /very/long/source/path/file.txt \
   /very/long/destination/path/
```

---

## Running Multiple Commands

```bash
command1 ; command2             # run both, regardless of success or failure
command1 && command2            # run command2 only if command1 succeeded
command1 || command2            # run command2 only if command1 failed
```

---

## Viewing File Contents

```bash
cat /etc/hosts                  # print entire file — good for short files
less /var/log/messages          # page through — good for long files (q to quit)
head -n 20 file.txt             # show first 20 lines
tail -n 20 file.txt             # show last 20 lines
tail -f /var/log/syslog         # watch a log file update in real time (Ctrl+C to stop)
```

---

## Getting Help for Any Command

```bash
man ls                          # full manual page for ls
ls --help                       # quick help summary
```

---

## Things to Remember

- The `#` prompt means root — double-check before pressing Enter
- `Tab` completion is not just convenient — it also confirms the file or command exists
- Linux commands are case-sensitive: `LS` is not the same as `ls`
- There is no "Are you sure?" prompt for most commands — especially `rm`
- Use `Ctrl+C` to escape from any command that is stuck or taking too long
- `sudo !!` is one of the most useful tricks — re-runs your last command with admin rights
