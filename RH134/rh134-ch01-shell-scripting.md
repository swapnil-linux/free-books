# RH134 Chapter 1 - Shell Scripting and the Command Line

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Write and run simple shell scripts, and use shell scripting features to efficiently run commands at the shell prompt.

---

## Windows vs Linux: Scripting Equivalents

| Windows Concept | Linux / Bash Equivalent |
|---|---|
| `.bat` / `.ps1` script file | `.sh` Bash script (or no extension) |
| `%VARIABLE%` | `$VARIABLE` or `${VARIABLE}` |
| `set VAR=value` | `VAR=value` (no spaces!) |
| `setx VAR value` (persistent) | Add `export VAR=value` to `~/.bashrc` |
| System Environment Variables | Exported environment variables (`export`) |
| `%PATH%` | `$PATH` (colon-separated, not semicolon) |
| `echo %ERRORLEVEL%` | `echo $?` |
| `if ERRORLEVEL 1 ...` | `if [[ $? -ne 0 ]]; then ...` |
| `for /L %i IN (1,1,5) DO ...` | `for i in {1..5}; do ... done` |
| Task Scheduler script | Cron job / systemd timer |

---

## 1. Shell Variables

### Setting and Using Variables

```bash
# Assign a variable - NO spaces around the = sign
MYVAR=hello
FILEPATH=/home/student/data

# Reference a variable
echo $MYVAR
echo ${MYVAR}          # Safer form - use inside strings or adjacent to text
echo "Path is: ${FILEPATH}/file.txt"
```

> **Key rule:** No spaces around `=`. `VAR = value` fails because the shell tries to run `VAR` as a command.

### Export as Environment Variable (inherited by child processes)

```bash
# Set locally (only this shell sees it)
EDITOR=vim

# Export so child processes inherit it
export EDITOR=vim

# Set and export in one step
export EDITOR=vim

# View all current environment variables
env

# Unset a variable
unset MYVAR
```

### Shell vs Environment Variables

| Type | Visible To | How to Set |
|---|---|---|
| Shell variable | Current shell only | `VAR=value` |
| Environment variable | Shell + all child processes | `export VAR=value` |

> Think of it like variable scope: a shell variable is local to this terminal session. An environment variable is inherited by anything launched from it - but changes never flow back up to the parent.

---

## 2. Important Built-in Variables

| Variable | Purpose | Example |
|---|---|---|
| `$PATH` | Colon-separated list of directories searched for commands | `/usr/bin:/usr/local/bin` |
| `$HOME` | Current user's home directory | `/home/student` |
| `$USER` | Current username | `student` |
| `$LANG` | Locale and character encoding | `en_US.UTF-8` |
| `$EDITOR` | Default text editor for CLI programs | `vim` |
| `$PS1` | Shell prompt format string | `[\u@\h \W]\$` |
| `$HISTFILE` | Path to command history file | `~/.bash_history` |
| `$?` | Exit code of the last command | `0` = success |
| `$0` | Name of the current script | `./myscript.sh` |
| `$1`, `$2` ... | Positional arguments passed to a script | `$1` is first arg |
| `$#` | Number of arguments passed to a script | `3` |
| `$@` | All arguments as separate words | useful in loops |

### Useful Examples

```bash
# Check what directories are in your PATH
echo $PATH

# Add ~/bin to PATH for current session
export PATH=${PATH}:~/bin

# Change locale temporarily
export LANG=fr_FR.UTF-8
date    # output will be in French!
export LANG=en_US.UTF-8

# Customise the prompt
export PS1="[\u@\h \W]\$ "
```

---

## 3. Bash Startup Scripts

These files run automatically when a shell starts. Edit them to make variable settings persistent.

| File | When it runs | Typical use |
|---|---|---|
| `/etc/profile` | Login shell, all users | System-wide settings |
| `~/.bash_profile` | Login shell, this user | User-specific login settings |
| `~/.bashrc` | Interactive non-login shell | Aliases, functions, prompt |
| `/etc/bashrc` | Non-login shell, all users | System-wide non-login settings |

> **Practical pattern:** Put permanent `export` statements in `~/.bashrc`. Run `source ~/.bashrc` (or `. ~/.bashrc`) to apply changes without logging out.

```bash
# Make EDITOR setting permanent
echo 'export EDITOR=vim' >> ~/.bashrc
source ~/.bashrc
```

---

## 4. Writing Bash Scripts

### Script Structure

```bash
#!/usr/bin/bash
# This is a comment
# Script name: myscript.sh
# Purpose: demonstrate basic script structure

echo "Hello from my script"
```

### The Shebang Line (`#!`)

- `#!/usr/bin/bash` tells the kernel which interpreter to use
- The kernel reads it when you execute the file directly
- It must be the very first line, no blank line before it
- Without it, the current shell interprets the file (unpredictable behaviour)

### Making a Script Executable

```bash
# Create the script
vim ~/bin/myscript.sh

# Give it execute permission
chmod +x ~/bin/myscript.sh

# Run it (if ~/bin is in your PATH)
myscript.sh

# Run it with an explicit path
./myscript.sh
```

> **Why `./`?** The current directory is intentionally NOT in `$PATH` for security reasons. You must explicitly specify the path.

### Output Redirection in Scripts

```bash
# Write stdout to a file (overwrite)
echo "Report start" > /tmp/report.txt

# Append stdout to a file
echo "More data" >> /tmp/report.txt

# Redirect stderr to a file
echo "ERROR: something failed" >&2

# Discard all output (stdout and stderr)
command > /dev/null 2>&1

# Redirect stderr to a log file
./myscript.sh 2> /tmp/errors.log
```

> **Best practice:** Send error messages to STDERR (`>&2`), not STDOUT. This allows log aggregators, cron, and systemd to capture errors separately from normal output.

### A Complete Script Example

```bash
#!/usr/bin/bash
# collect-info.sh - gather system info

OUTFILE=~/output.txt

echo "=== System Report ===" > ${OUTFILE}
echo "" >> ${OUTFILE}
echo "-- Block Devices --" >> ${OUTFILE}
lsblk >> ${OUTFILE}
echo "" >> ${OUTFILE}
echo "-- Disk Space --" >> ${OUTFILE}
df -h >> ${OUTFILE}

echo "Report written to ${OUTFILE}"
```

---

## 5. Exit Codes

Every command produces an exit code when it finishes.

| Exit Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General error |
| `2` | Misuse of shell built-in |
| Any non-zero | Failure of some kind |

```bash
# Check the exit code of the last command
ls /tmp
echo $?       # 0 - success

ls /nonexistent
echo $?       # 2 - failure

# Useful test commands
/bin/true     # always exits 0
/bin/false    # always exits 1

# Explicitly set exit code in a script
exit 0        # success
exit 1        # failure
```

> **Important:** `$?` is overwritten by every command. Save it to a variable immediately if you need to use it later.

```bash
systemctl is-active httpd > /dev/null 2>&1
HTTPD_STATUS=$?
# now you can use $HTTPD_STATUS multiple times safely
```

---

## 6. Loops

### for Loop

```bash
# Basic syntax
for VARIABLE in LIST; do
    COMMAND
done

# Loop over a literal list
for HOST in servera serverb serverc; do
    ssh student@${HOST} hostname
done

# Loop using brace expansion (sequence)
for i in {1..5}; do
    echo "Count: $i"
done

# Loop using brace expansion (list)
for FILE in file{a,b,c}; do
    echo $FILE
done

# Loop using seq (step value)
for EVEN in $(seq 2 2 10); do
    echo $EVEN          # 2 4 6 8 10
done

# Loop using command substitution
for PKG in $(rpm -qa | grep kernel); do
    echo $PKG
done
```

### while Loop

Runs as long as the condition is **true** (exit code 0).

```bash
# Basic syntax
while <CONDITION>; do
    <STATEMENT>
done

# Example: count from 1 to 5
x=1
while [[ $x -le 5 ]]; do
    echo "x is $x"
    x=$((x + 1))
done
```

### until Loop

Runs as long as the condition is **false** (non-zero exit code). The opposite of `while`.

```bash
# Basic syntax
until <CONDITION>; do
    <STATEMENT>
done

# Example: count down from 5
x=5
until [[ $x -lt 1 ]]; do
    echo "x is $x"
    x=$((x - 1))
done
```

---

## 7. Test Conditions `[[ ]]`

Always use double brackets `[[ ]]` in Bash scripts. Spaces inside brackets are **mandatory**.

### Numeric Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `-eq` | Equal to | `[[ $x -eq 5 ]]` |
| `-ne` | Not equal to | `[[ $x -ne 0 ]]` |
| `-gt` | Greater than | `[[ $x -gt 10 ]]` |
| `-ge` | Greater than or equal | `[[ $x -ge 10 ]]` |
| `-lt` | Less than | `[[ $x -lt 10 ]]` |
| `-le` | Less than or equal | `[[ $x -le 10 ]]` |

### String Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` or `==` | Equal (case sensitive) | `[[ "$NAME" == "root" ]]` |
| `!=` | Not equal | `[[ "$NAME" != "root" ]]` |
| `-z` | String is empty | `[[ -z "$VAR" ]]` |
| `-n` | String is not empty | `[[ -n "$VAR" ]]` |

### File Test Operators

| Operator | Meaning | Example |
|---|---|---|
| `-f` | File exists and is a regular file | `[[ -f /etc/passwd ]]` |
| `-d` | Directory exists | `[[ -d /tmp/mydir ]]` |
| `-e` | File or directory exists | `[[ -e /etc/hosts ]]` |
| `-r` | File is readable | `[[ -r /etc/shadow ]]` |
| `-w` | File is writable | `[[ -w /tmp/test ]]` |
| `-x` | File is executable | `[[ -x ~/bin/myscript.sh ]]` |

```bash
# Quick test examples
[[ 5 -gt 3 ]]; echo $?      # 0 (true)
[[ 5 -lt 3 ]]; echo $?      # 1 (false)
[[ -f /etc/passwd ]]; echo $?  # 0 (file exists)
[[ -d /nonexistent ]]; echo $? # 1 (does not exist)
```

---

## 8. Conditional Statements

### if / then

```bash
if <CONDITION>; then
    <STATEMENT>
fi

# Example
if [[ $? -ne 0 ]]; then
    echo "ERROR: Command failed" >&2
fi
```

### if / then / else

```bash
if <CONDITION>; then
    <STATEMENT>
else
    <STATEMENT>
fi

# Example
systemctl is-active httpd > /dev/null 2>&1
if [[ $? -eq 0 ]]; then
    echo "httpd is running"
else
    echo "httpd is stopped"
fi
```

### if / elif / else

```bash
if <CONDITION>; then
    <STATEMENT>
elif <CONDITION>; then
    <STATEMENT>
else
    <STATEMENT>
fi

# Example: detect active database and connect to it
systemctl is-active mariadb > /dev/null 2>&1
MARIADB_ACTIVE=$?

systemctl is-active postgresql > /dev/null 2>&1
POSTGRESQL_ACTIVE=$?

if [[ "$MARIADB_ACTIVE" -eq 0 ]]; then
    mysql
elif [[ "$POSTGRESQL_ACTIVE" -eq 0 ]]; then
    psql
else
    sqlite3
fi
```

---

## 9. Putting It All Together

A complete script combining variables, a loop, conditionals, exit codes, and STDERR:

```bash
#!/usr/bin/bash
# check-hosts.sh - verify SSH connectivity to a list of servers

SERVERS="servera serverb serverc"
LOGFILE=/tmp/host-check.log
FAILURES=0

echo "Host check started: $(date)" > ${LOGFILE}

for HOST in ${SERVERS}; do
    ssh -q -o ConnectTimeout=5 student@${HOST} hostname >> ${LOGFILE} 2>&1
    if [[ $? -eq 0 ]]; then
        echo "${HOST}: OK" | tee -a ${LOGFILE}
    else
        echo "ERROR: ${HOST} unreachable" >&2
        echo "ERROR: ${HOST} unreachable" >> ${LOGFILE}
        FAILURES=$((FAILURES + 1))
    fi
done

echo "Check complete. Failures: ${FAILURES}"
exit ${FAILURES}
```

---

## Quick Reference: Common Patterns

```bash
# Run a command on multiple servers
for HOST in servera serverb; do ssh student@${HOST} uptime; done

# Check if a file exists before acting on it
[[ -f /etc/myconfig ]] && echo "Config found"

# Capture command output into a variable
HOSTNAME=$(hostname --fqdn)

# Arithmetic
x=$((x + 1))
y=$((10 * 3))

# Make ~/bin available as a command directory
mkdir -p ~/bin
export PATH=${PATH}:~/bin

# Run a script in debug mode (prints every command before executing)
bash -x ./myscript.sh

# Check syntax without running the script
bash -n ./myscript.sh
```

---

## Things to Remember

1. **No spaces around `=` in variable assignment.** `VAR=value` works. `VAR = value` fails with a confusing error.

2. **`$?` is overwritten by every command.** If you need the exit code of a command, save it to a named variable immediately: `STATUS=$?`.

3. **Always use `[[ ]]` not `[ ]`.** Double brackets are a Bash keyword - safer quoting, more operators, fewer surprises.

4. **Spaces inside `[[ ]]` are mandatory.** `[[$x -eq 5]]` will fail. Always write `[[ $x -eq 5 ]]`.

5. **Shell variables are not inherited by child processes** unless you `export` them. If a script seems not to see a variable you set, this is why.

6. **`./` is required to run a script in the current directory.** The current directory is not in `$PATH` by design - this is a security feature, not a bug.

7. **`#!/usr/bin/bash` is processed by the kernel**, not the shell. It must be line 1, character 1 of the file.

8. **Send error messages to STDERR**, not STDOUT: `echo "ERROR: ..." >&2`. This keeps error output separate from normal output and allows proper logging.

9. **`for`, `while`, and `until` all use `do ... done`** to wrap the command block. Forgetting `done` is the most common syntax error.

10. **`bash -x ./script.sh`** is your best friend when debugging. It prints every line before executing it, showing exactly what the shell sees after variable expansion.
