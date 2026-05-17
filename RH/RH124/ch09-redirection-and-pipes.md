# Chapter 9 – Redirecting Shell Input and Output
## RH124 Student Quick Reference

---

## The Core Concept — Everything Is a Stream

Every Linux command has three standard data streams connected to it by default:

```
                    ┌─────────────┐
Keyboard ──stdin──► │   command   │ ──stdout──► Terminal (normal output)
                    │    (your    │ ──stderr──► Terminal (error messages)
                    │   program)  │
                    └─────────────┘
```

| Stream | Number | Name | Default Source/Destination |
|---|---|---|---|
| **stdin** | 0 | Standard Input | Keyboard |
| **stdout** | 1 | Standard Output | Terminal screen |
| **stderr** | 2 | Standard Error | Terminal screen (separately) |

> **Windows equivalent:** Think of stdout as the normal output window and stderr as the error/warning window — except in Linux you can route them independently to different places.

---

## Why This Matters

Redirection and pipes are what make the Linux command line genuinely powerful. Instead of writing one big program to do everything, you chain small focused commands together — each doing one thing well.

> *"This is the Unix philosophy: write programs that do one thing and do it well, and that work together."*
> — Doug McIlroy, Bell Labs

---

## Output Redirection — Writing to Files

### The Key Operators

| Operator | What It Does | Watch Out |
|---|---|---|
| `>` | Redirect stdout to a file **(overwrites)** | Destroys existing file content |
| `>>` | Redirect stdout to a file **(appends)** | Safe — adds to end of file |
| `2>` | Redirect stderr to a file **(overwrites)** | Only errors go here |
| `2>>` | Redirect stderr to a file **(appends)** | |
| `&>` | Redirect **both** stdout and stderr to a file (overwrite) | Bash-specific |
| `&>>` | Redirect **both** stdout and stderr to a file (append) | Bash-specific |
| `> file 2>&1` | Redirect both to same file (portable form) | Order matters — see below |
| `2>/dev/null` | Discard error messages entirely | The "silence errors" trick |
| `>/dev/null` | Discard all normal output entirely | |
| `>/dev/null 2>&1` | Discard everything — silent command | |

---

### Redirection Examples

```bash
# Save command output to a file (overwrites)
date > /tmp/timestamp.txt

# Append output to an existing file
echo "Job completed" >> /var/log/myjob.log

# Save errors separately from normal output
find /etc -name "*.conf" > results.txt 2> errors.txt

# Discard errors — only see normal output
find /etc -name "*.conf" 2> /dev/null

# Save everything (stdout + stderr) to one file
find /etc -name "*.conf" &> everything.txt
# or the portable equivalent:
find /etc -name "*.conf" > everything.txt 2>&1

# Append everything to an existing file
find /etc -name "*.conf" >> everything.txt 2>&1

# Completely silent — discard all output
some-noisy-command > /dev/null 2>&1
```

---

### ⚠️ The Order of `2>&1` Matters

This is one of the most common mistakes beginners make:

```bash
# CORRECT — redirect stdout to file, then redirect stderr to wherever stdout now goes (the file)
command > output.log 2>&1

# WRONG — redirects stderr to terminal first, then stdout to file (errors still appear on screen)
command 2>&1 > output.log
```

Read it left to right — the shell processes redirections in order.

---

## Pipes — Chaining Commands Together

The pipe `|` connects the **stdout** of one command to the **stdin** of the next.

```
command1 | command2 | command3
    ↓           ↓           ↓
  output  →   input     output  →  input    → terminal
```

### Pipe Examples

```bash
# Page through a long listing
ls -la /usr/bin | less

# Count how many files are in a directory
ls /etc | wc -l

# Find the 10 most recently changed files
ls -t | head -n 10

# Search for a specific process
ps aux | grep nginx

# Show only unique lines from a sorted file
sort names.txt | uniq

# Count how many times each line appears
sort names.txt | uniq -c | sort -rn

# Find the largest files
du -sh /var/* | sort -rh | head -n 10

# Show only error lines from a log
cat /var/log/messages | grep -i error

# Chain multiple filters
cat /var/log/secure | grep "Failed" | grep "ssh" | wc -l
```

---

## `tee` — Output to Screen AND File Simultaneously

The `tee` command splits output — it passes data through to the next command AND saves a copy to a file at the same time.

```
command ──► tee ──► terminal (stdout continues down the pipe)
                └──► file (saved copy)
```

```bash
# Display output on screen AND save to file
ls -la | tee filelist.txt

# In the middle of a pipeline — display AND save AND continue processing
find /etc -name "*.conf" | tee found-configs.txt | wc -l

# Append to a file instead of overwriting
ls -la | tee -a filelist.txt

# Save to multiple files at once
ls | tee file1.txt file2.txt file3.txt
```

> **When to use tee:** Whenever you want to see output on screen while also saving it — very useful for long-running commands or troubleshooting where you need a log.

---

## Input Redirection — Reading from Files

Less commonly used interactively but important in scripts:

```bash
# Feed a file as input to a command (instead of keyboard)
mail -s "Subject" user@example.com < message.txt

# Count lines in a file (two equivalent ways)
wc -l < /etc/passwd
wc -l /etc/passwd
```

---

## `/dev/null` — The Black Hole

`/dev/null` is a special file that:
- **Discards anything written to it** (like a black hole)
- **Returns nothing when read** (like an empty file)

```bash
# Silence all error messages
find / -name "secret" 2> /dev/null

# Run a command completely silently
backup-script.sh > /dev/null 2>&1

# Create an empty file instantly
cat /dev/null > emptyfile.txt
# (though touch emptyfile.txt is simpler)
```

---

## Useful Commands in Pipelines

These commands are almost always used in combination with pipes:

| Command | What It Does | Example |
|---|---|---|
| `grep pattern` | Filter lines matching a pattern | `cat log.txt \| grep ERROR` |
| `grep -v pattern` | Filter OUT lines matching pattern | `cat log.txt \| grep -v DEBUG` |
| `grep -i pattern` | Case-insensitive search | `cat log.txt \| grep -i warning` |
| `grep -c pattern` | Count matching lines | `cat log.txt \| grep -c ERROR` |
| `wc -l` | Count lines | `ls /etc \| wc -l` |
| `wc -w` | Count words | `cat file.txt \| wc -w` |
| `sort` | Sort lines alphabetically | `cat names.txt \| sort` |
| `sort -r` | Sort in reverse | `cat names.txt \| sort -r` |
| `sort -n` | Sort numerically | `cat numbers.txt \| sort -n` |
| `uniq` | Remove consecutive duplicate lines | `sort file.txt \| uniq` |
| `uniq -c` | Count occurrences of each line | `sort file.txt \| uniq -c` |
| `head -n 10` | Show first 10 lines | `cat log.txt \| head -n 10` |
| `tail -n 10` | Show last 10 lines | `cat log.txt \| tail -n 10` |
| `cut -d: -f1` | Extract a field from delimited text | `cat /etc/passwd \| cut -d: -f1` |
| `tr 'a-z' 'A-Z'` | Translate/replace characters | `echo "hello" \| tr 'a-z' 'A-Z'` |
| `sed 's/old/new/g'` | Find and replace in stream | `cat file \| sed 's/error/ERROR/g'` |
| `awk '{print $1}'` | Extract column from text | `ls -la \| awk '{print $9}'` |
| `less` | Page through output | `ls /usr/bin \| less` |
| `more` | Simpler pager | `cat bigfile.txt \| more` |
| `tee file.txt` | Copy to screen and file | `ls \| tee listing.txt` |

---

## Combining Pipes and Redirection

```bash
# Process output AND save errors separately
find /etc -name "*.conf" 2> errors.txt | wc -l

# Save piped output to a file at the end
ls -t | head -n 10 > recent-files.txt

# To redirect stderr through a pipe, merge it with stdout first
find / -name "passwd" 2>&1 | grep -v "Permission denied" | less
```

---

## Real-World Examples

```bash
# How many users are on the system?
cat /etc/passwd | wc -l

# Find the top 5 largest directories under /var
du -sh /var/* 2>/dev/null | sort -rh | head -n 5

# Show all failed SSH login attempts today
grep "Failed password" /var/log/secure | grep "$(date +%b\ %e)" | wc -l

# Save a full package list (useful before a major change)
rpm -qa | sort > /tmp/packages-before.txt

# Watch a log file and highlight errors in real time
tail -f /var/log/messages | grep --line-buffered -i error

# Extract just the usernames from /etc/passwd
cut -d: -f1 /etc/passwd | sort

# Find the most common IP address in a web log
awk '{print $1}' /var/log/httpd/access_log | sort | uniq -c | sort -rn | head -n 10

# Backup a config file and make changes — one liner
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak && vim /etc/ssh/sshd_config
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| `command > file 2>&1` vs `command 2>&1 > file` | Wrong order — stderr goes to screen | Always put `2>&1` after `>` |
| `command > file \| next` | Output goes to file, not to next command | Use `tee` instead |
| `>` instead of `>>` | Overwrites file, losing existing content | Use `>>` to append |
| Forgetting quotes around filenames with spaces | Shell splits the filename | `> "my file.txt"` |

---

## Windows Comparison

| Windows (PowerShell) | Linux | Notes |
|---|---|---|
| `command > file.txt` | `command > file.txt` | Same syntax — both overwrite |
| `command >> file.txt` | `command >> file.txt` | Same — both append |
| `command 2> file.txt` | `command 2> file.txt` | Same |
| `command \| next` | `command \| next` | Pipe syntax identical |
| `$null` | `/dev/null` | Discard output |
| `Tee-Object -FilePath file` | `tee file` | Split output to screen + file |
| `Select-String "pattern"` | `grep "pattern"` | Filter lines by pattern |
| `Select-Object -First 10` | `head -n 10` | First N lines |
| `Measure-Object -Line` | `wc -l` | Count lines |

> **Key difference:** PowerShell pipes **objects** between commands. Linux pipes **plain text**. This means Linux pipe commands work on raw text streams — simpler but requires knowing the text format of each command's output.

---

## Things to Remember

- `>` **overwrites** — never use it on a file you care about without a backup
- `>>` **appends** — safe to use in scripts that run repeatedly
- stdout and stderr both appear on screen by default — they look the same but are separate streams
- `2>/dev/null` is one of the most useful patterns in scripting — silences noisy error messages
- The pipe `|` only passes **stdout** — stderr still goes to screen unless you redirect it first
- Order matters with `2>&1` — always put it after `>`
- `tee` is your friend when you want to see output AND save it — do not choose one or the other
