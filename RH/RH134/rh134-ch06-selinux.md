# RH134 Chapter 6 - Managing Security with SELinux

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Use SELinux to provide additional access control on a system, and to prevent unexpected access by compromised processes.

---

## Windows vs Linux: Access Control Equivalents

| Windows Concept | Linux / SELinux Equivalent |
|---|---|
| File ACLs (Discretionary) | DAC - standard Unix `rwx` permissions |
| Mandatory Integrity Control (MIC) | MAC - SELinux mandatory access control |
| AppLocker / Windows Defender Application Control | SELinux targeted policy per process |
| Integrity Levels (Low/Medium/High/System) | SELinux context types (`httpd_t`, `mysqld_t`) |
| Group Policy (cannot be bypassed by user) | SELinux policy (cannot be bypassed even by root) |
| Event Viewer / Security log | `/var/log/audit/audit.log` + `sealert` |
| Registry-based policy enforcement | `/etc/selinux/config` + policy database |

---

## 1. What Is SELinux?

### Discretionary vs Mandatory Access Control

| | DAC (Standard Unix Permissions) | MAC (SELinux) |
|---|---|---|
| Who sets it | File owner | System administrator via policy |
| Can root bypass it? | Yes | No |
| Based on | User/group/world bits | Labels on every process, file, and port |
| If bypassed | Attacker has owner's full access | Attacker is still confined to the process's allowed actions |
| Example | `chmod 777 /etc/shadow` | httpd process cannot read `/etc/shadow` even if apache is root |

> **The key value proposition:** If Apache is compromised by an attacker, SELinux limits what that attacker can do. The compromised process is still confined to `httpd_t` policy - it cannot read `/etc/shadow`, cannot write to `/home`, cannot bind to arbitrary ports. Without SELinux, a compromised Apache running as root is game over.

### How SELinux Makes Decisions

Every resource in the system has a label. Policy rules define which process labels can access which resource labels with which actions. If no rule explicitly allows an action, it is denied.

```
Process label (httpd_t)
    + Resource label (httpd_sys_content_t)
    + Action (read)
    = ALLOW (rule exists in policy)

Process label (httpd_t)
    + Resource label (shadow_t)
    + Action (read)
    = DENY (no rule exists)
```

---

## 2. SELinux Modes

| Mode | Behaviour | Use case |
|---|---|---|
| **Enforcing** | Policy is enforced; denials block access AND are logged | Production - always |
| **Permissive** | Policy is NOT enforced; denials are only logged | Troubleshooting only |
| **Disabled** | SELinux is completely off; nothing is logged or enforced | Not supported via config in RHEL 10+ |

> **Permissive is NOT safe mode.** All denied actions are still logged as if they would have been blocked. It is a diagnostic tool, not a solution. Never run production workloads in permissive mode.

### Checking the Current Mode

```bash
# Check current mode (runtime)
getenforce

# Check both current and configured mode
sestatus
```

### Changing Mode Temporarily (Survives Until Reboot)

```bash
# Switch to permissive
setenforce 0
setenforce Permissive   # same thing

# Switch back to enforcing
setenforce 1
setenforce Enforcing    # same thing
```

### Changing Mode Permanently (Persists Across Reboots)

Edit `/etc/selinux/config`:

```bash
vim /etc/selinux/config
```

```bash
# /etc/selinux/config

SELINUX=enforcing      # enforcing | permissive
SELINUXTYPE=targeted   # targeted | minimum | mls
```

```bash
# Verify the change
cat /etc/selinux/config | grep ^SELINUX

# Reboot to apply
# Red Hat recommends rebooting when switching from permissive TO enforcing
# to ensure services that started in permissive mode are re-confined
systemctl reboot
```

> **RHEL 10 change:** `SELINUX=disabled` in `/etc/selinux/config` is no longer supported. To boot without SELinux, pass `selinux=0` as a kernel boot parameter at the GRUB prompt.

---

## 3. SELinux Context Labels

Every process, file, directory, and port has an SELinux context label in the format:

```
user:role:type:level
  |    |    |    |
  |    |    |    Security level (MLS/MCS - ignore for targeted policy)
  |    |    Type - the ONLY field that matters in targeted policy
  |    Role - relevant in MLS environments
  SELinux user - maps to Linux user
```

### In Practice: Only the Type Field Matters

Type labels end in `_t`. These are the labels you will work with every day:

| Type Label | Assigned to |
|---|---|
| `httpd_t` | Apache web server process |
| `httpd_sys_content_t` | Web server content files (`/var/www/html/`) |
| `mysqld_t` | MariaDB/MySQL process |
| `mysqld_db_t` | Database files |
| `sshd_t` | SSH daemon process |
| `admin_home_t` | Files in `/root/` |
| `user_home_t` | Files in user home directories |
| `tmp_t` | Files in `/tmp/` |
| `default_t` | Files with no specific policy match |
| `shadow_t` | `/etc/shadow` |

### Viewing Context Labels

```bash
# View context of files and directories
ls -Z /var/www/html/
ls -lZ /var/www/html/
ls -Zd /custom            # just the directory itself, not contents

# View context of running processes
ps axZ
ps -ZC httpd              # just httpd processes

# View context of your current shell session
id -Z
```

### Example: What a Context Looks Like

```bash
$ ls -Z /var/www/html/
system_u:object_r:httpd_sys_content_t:s0 index.html

$ ps -ZC httpd
LABEL                              PID TTY   CMD
system_u:system_r:httpd_t:s0     6780 ?     httpd
```

---

## 4. Controlling SELinux File Contexts

### The Golden Rule

> **Never use `chcon` alone in production.** `chcon` changes the context directly on the file without writing a policy rule. The change will be overwritten the next time `restorecon` is run on that directory. Always use `semanage fcontext` to write the rule, then `restorecon` to apply it.

| Command | What it does | Persistent? |
|---|---|---|
| `chcon -t TYPE FILE` | Changes context directly on file | No - wiped by `restorecon` |
| `semanage fcontext -a -t TYPE 'PATTERN'` | Writes a policy rule | Yes - survives relabel |
| `restorecon -Rv PATH` | Applies policy rules to files on disk | Applies the policy |

### The Correct Workflow for Custom Directories

```bash
# Scenario: you want Apache to serve content from /custom/
# Step 1: Check what context /var/www/html has (the known-good reference)
ls -ldZ /var/www/html
# system_u:object_r:httpd_sys_content_t:s0

# Step 2: Write a policy rule that applies httpd_sys_content_t to /custom/
semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'
# The (/.*)?  means: the directory AND everything inside it

# Step 3: Apply the policy rule to files that already exist on disk
restorecon -Rv /custom
# -R = recursive, -v = verbose (shows what was relabeled)

# Step 4: Verify
ls -lZd /custom
ls -lZ /custom/
```

### Managing semanage fcontext Policies

```bash
# List ALL file context policies (very long list)
semanage fcontext -l

# Filter to see only policies for a specific path
semanage fcontext -l | grep /var/www

# Show ONLY local customisations (what YOU have changed)
semanage fcontext -l -C

# Delete a custom policy rule
semanage fcontext -d '/custom(/.*)?'
```

### Using `chcon` for Quick Tests Only

```bash
# Temporarily change context (testing and debugging only)
chcon -t httpd_sys_content_t /virtual/index.html
chcon -R -t httpd_sys_content_t /virtual/     # recursive

# Reset a file to its policy-defined context
restorecon -v /virtual/index.html

# Reset entire directory tree to policy-defined contexts
restorecon -Rv /virtual/
```

### Preserving Context When Copying Files

```bash
# cp changes the context to match the destination directory
cp /root/myfile /var/www/html/         # context changes to httpd_sys_content_t

# cp -p preserves all attributes including context
cp -p /var/www/html/file /tmp/         # preserves httpd_sys_content_t (may cause issues)

# cp --preserve=context preserves only context
cp --preserve=context /source /dest
```

> **Common problem:** Creating a file in `/root/` (context `admin_home_t`) then moving it to `/var/www/html/` retains `admin_home_t` - Apache cannot read it. Always run `restorecon` after moving files into service directories.

---

## 5. SELinux Booleans

Booleans are pre-built switches in the SELinux policy that enable optional application behaviours. They are the "approved knobs" that policy authors expose for common use cases.

### Viewing Booleans

```bash
# List all booleans with current and default values
semanage boolean -l

# Filter to a specific service (e.g. httpd)
semanage boolean -l | grep httpd

# Show ONLY booleans that have been changed from policy defaults
semanage boolean -l -C

# Check a specific boolean's current value
getsebool httpd_enable_homedirs
getsebool -a | grep httpd           # all httpd booleans
getsebool -a                        # all booleans
```

### semanage boolean -l Output Format

```
httpd_enable_homedirs       (off , off)    Allow httpd to enable homedirs
                              ^     ^
                              |     Default value (from policy)
                              Current value (runtime)
```

> When both values are the same, no change has been made (or the change was made persistently). When they differ, a temporary change is active.

### Changing Booleans

```bash
# Temporarily enable (until reboot)
setsebool httpd_enable_homedirs on
setsebool httpd_enable_homedirs 1   # same thing

# Temporarily disable
setsebool httpd_enable_homedirs off

# Permanently enable (survives reboot) - always use -P for production
setsebool -P httpd_enable_homedirs on

# Permanently disable
setsebool -P httpd_enable_homedirs off
```

### Commonly Used Booleans

| Boolean | Service | What it enables |
|---|---|---|
| `httpd_enable_homedirs` | Apache | Serve content from user home directories |
| `httpd_can_network_connect` | Apache | Allow httpd to make outbound network connections |
| `httpd_can_network_connect_db` | Apache | Allow httpd to connect to databases |
| `httpd_use_nfs` | Apache | Allow httpd to serve files on NFS shares |
| `ftp_home_dir` | vsftpd | Allow FTP access to home directories |
| `ssh_sysadm_login` | SSH | Allow sysadm role users to log in via SSH |
| `samba_enable_home_dirs` | Samba | Allow Samba to share home directories |

### Finding the Right Boolean

```bash
# Check the service-specific SELinux man page
man httpd_selinux
man sshd_selinux
man ftpd_selinux

# Search all booleans for a keyword
semanage boolean -l | grep nfs
semanage boolean -l | grep home
```

---

## 6. Investigating and Resolving SELinux Issues

### Where SELinux Logs Denials

| Log Location | Contents |
|---|---|
| `/var/log/audit/audit.log` | Raw AVC (Access Vector Cache) denial messages |
| `/var/log/messages` | Human-readable summary from `setroubleshootd` with `sealert` UUID |

### Troubleshooting Workflow

```
Service fails (403, permission denied, connection refused)
    |
    v
Check /var/log/messages for SELinux denial + UUID
    |
    v
Run sealert -l UUID for full analysis
    |
    v
Identify the root cause:
    - Wrong file context (most common) -> semanage fcontext + restorecon
    - Optional feature not enabled    -> setsebool -P
    - Genuinely unexpected access      -> investigate further, do NOT disable SELinux
    |
    v
Apply the fix
    |
    v
Test the service
    |
    v
Verify with getenforce (confirm still Enforcing)
```

### Reading AVC Messages in `audit.log`

```bash
# View recent SELinux denials
tail /var/log/audit/audit.log

# Example AVC message (annotated):
type=AVC msg=audit(1749508506.271:159): avc:  denied  { getattr }
#                                                         ^action denied
for pid=1984 comm="httpd"
#                  ^process name
path="/var/www/html/mypage"
#     ^resource being accessed
scontext=system_u:system_r:httpd_t:s0
#         ^source (process) context
tcontext=unconfined_u:object_r:admin_home_t:s0
#         ^target (file) context - this is the problem!
tclass=file permissive=0
```

> **Reading the AVC:** `httpd_t` tried to `getattr` a file with context `admin_home_t`. No policy rule allows this. The file was created in `/root/` and moved to `/var/www/html/` without fixing the context.

### The `sealert` Command

```bash
# View all SELinux alerts from the audit log (summary)
sealert -a /var/log/audit/audit.log

# View details for a specific alert by UUID (from /var/log/messages)
sealert -l UUID
sealert -l a0bca4b2-46a9-4252-b0cd-6de0dfcdd454
```

### Interpreting sealert Output

```bash
sealert -l UUID
# Shows:
# - What was denied and which process/file was involved
# - Suggested fixes with a CONFIDENCE PERCENTAGE
# - Source context (process)
# - Target context (file)
# - Raw AVC message
```

> **Important:** The confidence percentage is a hint, not a guarantee. `sealert` may suggest `semanage fcontext` when the real fix is to move the file back to its correct location and run `restorecon`. Always understand the root cause before applying any suggestion.

### The `ausearch` Command

```bash
# Search audit log for AVC (SELinux denial) events
ausearch -m AVC

# Search for denials involving a specific process
ausearch -m AVC -c httpd

# Show recent AVC events in readable format
ausearch -m AVC -ts recent

# Combine with aureport for a summary
aureport --avc
```

### Common Root Causes and Fixes

| Symptom | Most Likely Cause | Fix |
|---|---|---|
| File in standard location but wrong context | File was moved/copied from elsewhere | `restorecon -Rv /path/` |
| File in non-standard location | No policy exists for this location | `semanage fcontext -a -t TYPE 'PATTERN'` then `restorecon` |
| Service cannot do optional thing (NFS, home dirs) | Feature not enabled | `setsebool -P BOOLEAN on` |
| Correct context but still denied | Wrong location for the file | Move file to the correct standard location |

---

## 7. SELinux Context on Ports

SELinux also controls which ports services can bind to. If you change a service to a non-standard port, you must tell SELinux.

```bash
# List all port contexts
semanage port -l

# Find what ports Apache is allowed to use
semanage port -l | grep http

# Allow Apache to also use port 8888
semanage port -a -t http_port_t -p tcp 8888

# Verify
semanage port -l | grep 8888
```

---

## Quick Reference: All Commands

```bash
# --- mode ---
getenforce                          # Check current mode
sestatus                            # Check current + configured mode
setenforce 0                        # Permissive (temporary)
setenforce 1                        # Enforcing (temporary)
vim /etc/selinux/config             # Persistent mode (edit SELINUX= line)

# --- viewing contexts ---
ls -Z FILE                          # File context
ls -lZ DIR/                         # Directory contents with context
ls -Zd DIR                          # Directory itself only
ps axZ                              # All process contexts
ps -ZC httpd                        # Specific process context
id -Z                               # Current shell context

# --- file contexts (correct workflow) ---
semanage fcontext -l                           # List all policies
semanage fcontext -l | grep /var/www           # Filter policies
semanage fcontext -l -C                        # Show only local changes
semanage fcontext -a -t TYPE 'PATTERN(/.*)?'   # Add policy rule
semanage fcontext -d 'PATTERN'                 # Delete policy rule
restorecon -Rv /path/                          # Apply policy to existing files
restorecon -v /path/file                       # Apply to single file

# --- chcon (testing only - not persistent) ---
chcon -t httpd_sys_content_t /path/file
chcon -R -t httpd_sys_content_t /path/dir/

# --- booleans ---
getsebool -a                                   # List all booleans
getsebool BOOLEAN                              # Check specific boolean
semanage boolean -l                            # Full list with descriptions
semanage boolean -l | grep httpd               # Filter
semanage boolean -l -C                         # Show only changed booleans
setsebool BOOLEAN on                           # Temporary enable
setsebool -P BOOLEAN on                        # Permanent enable
setsebool -P BOOLEAN off                       # Permanent disable

# --- troubleshooting ---
tail /var/log/audit/audit.log                  # Raw AVC denials
grep sealert /var/log/messages                 # Denial summaries + UUIDs
sealert -a /var/log/audit/audit.log            # All alerts summary
sealert -l UUID                                # Specific alert detail
ausearch -m AVC                                # Search audit log for denials
ausearch -m AVC -c httpd                       # Denials for specific process
aureport --avc                                 # Summary report

# --- ports ---
semanage port -l | grep http                   # List http port contexts
semanage port -a -t http_port_t -p tcp 8888    # Add port to type
```

---

## Key Configuration Files and Paths

| Path | Purpose |
|---|---|
| `/etc/selinux/config` | Persistent SELinux mode and policy type |
| `/var/log/audit/audit.log` | Raw AVC denial events |
| `/var/log/messages` | Human-readable SELinux alerts (setroubleshootd) |
| `/etc/selinux/targeted/contexts/files/` | SELinux file context policy database |
| `/var/www/html/` | Default httpd content (httpd_sys_content_t) |

---

## Things to Remember

1. **SELinux is a second layer of access control, not a replacement for file permissions.** Both must allow an action for it to succeed. SELinux cannot grant access that DAC denies, and DAC cannot grant access that SELinux denies.

2. **Permissive mode is a diagnostic tool, not a workaround.** Denials are still logged in permissive mode. Never leave a production system in permissive mode. Use it to identify what SELinux would block, fix the root cause, then switch back to enforcing.

3. **`chcon` is temporary. `semanage fcontext + restorecon` is permanent.** A context set with `chcon` will be overwritten the next time `restorecon` is run on that path. Always write the rule into policy with `semanage fcontext -a` first.

4. **The most common SELinux problem is a file with the wrong context.** Usually caused by: creating a file in `/root/` or `/tmp/` and moving it to a service directory, or creating a new directory outside the standard paths. Fix: `semanage fcontext -a -t TYPE 'PATH(/.*)?'` then `restorecon -Rv PATH`.

5. **The `(/.*)?` pattern in `semanage fcontext` means "this directory AND everything inside it."** Without it, the rule only applies to the directory itself, not its contents. Always include `(/.*)?` when labelling a directory for a service.

6. **`setsebool -P` is the only boolean change that survives a reboot.** Without `-P`, the change is temporary and reverts on reboot. Always use `-P` when enabling a boolean in production.

7. **`semanage boolean -l -C` shows only what you have changed from defaults.** This is the compliance auditor's best friend - it gives a concise list of every non-default SELinux policy decision on the system.

8. **`sealert` suggestions have a confidence rating - read it critically.** A 99% confidence fix may still be wrong if the root cause is a misplaced file rather than a missing policy. Understand the cause before applying any suggestion.

9. **`SELINUX=disabled` in `/etc/selinux/config` is not supported in RHEL 10.** To boot without SELinux, pass `selinux=0` at the kernel boot prompt. Files will not be relabelled while SELinux is off - enabling it again requires a full filesystem relabel.

10. **If a service fails and SELinux is in enforcing mode, always check `sealert` before disabling SELinux.** Disabling SELinux is never the correct long-term answer. It is the equivalent of removing a smoke alarm because it keeps going off - the underlying problem still exists.
