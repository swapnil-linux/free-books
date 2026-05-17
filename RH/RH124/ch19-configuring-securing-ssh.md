# Chapter 19 – Configuring and Securing SSH
## RH124 Student Quick Reference

---

## What Is SSH and Why Does It Matter?

**SSH (Secure Shell)** is the standard protocol for secure remote administration on Linux. Every remote connection to a Linux server — whether you are logging in interactively, running a command remotely, or copying files — typically uses SSH.

```
┌─────────────┐    encrypted channel    ┌─────────────┐
│ SSH Client  │ ◄─────────────────────► │ SSH Server  │
│   (ssh)     │      port 22 TCP        │   (sshd)    │
└─────────────┘                          └─────────────┘
   your laptop                              remote host
```

> **Windows equivalent:** SSH is the Linux equivalent of **Remote Desktop (RDP) + WinRM + PowerShell Remoting** combined — but far lighter, usually command-line only, and ubiquitous across every Unix-like system since the late 1990s. Modern Windows now includes OpenSSH too, so the `ssh` command works identically from Windows, macOS, and Linux.

---

## The Two Types of SSH Authentication

| | **Password Authentication** | **Key-based Authentication** |
|---|---|---|
| What you provide | Your account password | Your private key file |
| What the server checks | Password hash on the server | Your public key in `authorized_keys` |
| Vulnerable to brute force? | Yes | Essentially no |
| Requires typing each time? | Yes | No (unless passphrase-protected) |
| Recommended? | For initial setup only | **Yes — use wherever possible** |

---

## SSH Host Keys — The Server's Identity

The server has its **own** keypair that identifies it to clients. This is different from the user keys you create for authentication.

### The First-Connection Prompt

The first time you connect to a new server, you see this:

```
$ ssh alice@newserver.example.com
The authenticity of host 'newserver.example.com' can't be established.
ED25519 key fingerprint is SHA256:i5IURCZ+/4moD3P1TIYzZdFzqBmgm3Kl80YNbh8z7yU.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is your client saying *"I have not seen this server before — here is the key it just presented. Do you trust it?"*

If you type `yes`, the server's host key is saved to `~/.ssh/known_hosts`. On every subsequent connection, your client silently verifies that the server is still presenting the **same** key.

### Why This Matters — Man-in-the-Middle Protection

If someone intercepts the connection and impersonates the server, they will present a **different** key. Your client will refuse to connect and show a loud warning:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
```

### When This Warning Is Legitimate (and Not an Attack)

- Server was rebuilt or reinstalled (host keys regenerated)
- VM was restored from a different snapshot
- IP address was reassigned to a different server
- Container was replaced

### Host Key Files on the Server

Host keys live in `/etc/ssh/`:

```bash
ls -l /etc/ssh/ssh_host_*_key*
# /etc/ssh/ssh_host_ed25519_key       ← private (mode 600, root only)
# /etc/ssh/ssh_host_ed25519_key.pub   ← public
# /etc/ssh/ssh_host_ecdsa_key
# /etc/ssh/ssh_host_ecdsa_key.pub
# /etc/ssh/ssh_host_rsa_key
# /etc/ssh/ssh_host_rsa_key.pub
```

### Known Hosts Files on the Client

| Location | Scope |
|---|---|
| `~/.ssh/known_hosts` | Per-user (most common) |
| `/etc/ssh/ssh_known_hosts` | System-wide (all users on this machine) |

---

## Managing Host Keys

### Inspecting Fingerprints

```bash
# Show fingerprint of each host key on the server
ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub

# Fingerprints of all hosts in your known_hosts file
ssh-keygen -l -f ~/.ssh/known_hosts
```

### Removing a Stale Host Key (When the Server Has Legitimately Changed)

```bash
ssh-keygen -R hostname                              # remove all entries for hostname
ssh-keygen -R hostname -f /etc/ssh/ssh_known_hosts  # from the system-wide file
```

### Pre-Populating Host Keys — `ssh-keyscan`

Useful for automation — fetch a server's host keys without connecting:

```bash
# Fetch all host keys for a server
ssh-keyscan hostb

# Fetch only the ED25519 key
ssh-keyscan -t ed25519 hostb

# Append to known_hosts (quiet mode)
ssh-keyscan -q -t ed25519 hostb >> ~/.ssh/known_hosts

# Scan an entire subnet — bulk populate before automation runs
sudo ssh-keyscan -q -t ed25519 192.168.1.0/24 >> /etc/ssh/ssh_known_hosts
```

> ⚠️ **`ssh-keyscan` does not verify identity.** You are trusting whatever key happens to be presented at that moment. For production, verify fingerprints through an out-of-band channel (like the server console or a separate secure notification).

---

## SSH User Keys — Key-based Authentication

### How Key-based Auth Works

```
┌──────────────┐                  ┌──────────────┐
│   Client     │                  │    Server    │
│              │                  │              │
│ ~/.ssh/      │                  │ ~/.ssh/      │
│  id_ed25519  │                  │  authorized_ │
│  (private)   │                  │    keys      │
└──────┬───────┘                  └──────┬───────┘
       │                                  │
       │  1. Client offers public key     │
       │ ───────────────────────────────► │
       │                                  │
       │  2. Server checks authorized_keys│
       │                                  │
       │  3. Server sends encrypted       │
       │     challenge                    │
       │ ◄─────────────────────────────── │
       │                                  │
       │  4. Client decrypts with private │
       │     key and sends back           │
       │ ───────────────────────────────► │
       │                                  │
       │  5. Access granted               │
       │ ◄─────────────────────────────── │
```

The **private key never leaves your machine**. The server only ever sees your public key.

---

## Generating an SSH Key Pair

```bash
ssh-keygen                              # interactive, uses defaults (Ed25519 on RHEL 10)
ssh-keygen -t ed25519                   # explicit Ed25519 (modern, recommended)
ssh-keygen -t rsa -b 4096               # RSA 4096-bit (compatibility with older systems)
ssh-keygen -t ed25519 -C "alice@laptop" # add a comment (often used for identification)
ssh-keygen -f ~/.ssh/custom_key         # save to a specific location
```

### What Gets Created

```
~/.ssh/id_ed25519           ← PRIVATE key (mode 600, never share)
~/.ssh/id_ed25519.pub       ← PUBLIC key (safe to share)
```

### Default Algorithm on RHEL 10

RHEL 10 uses **Ed25519** by default — modern, fast, secure, and produces short keys. RSA is used instead when the system is in FIPS mode.

### Passphrase: To Use or Not?

When `ssh-keygen` prompts for a passphrase:

| | With Passphrase | Without Passphrase |
|---|---|---|
| Security if key is stolen | Attacker still needs passphrase | Attacker has full access |
| Convenience | Must type each time (or use `ssh-agent`) | Instant login |
| Good for | Interactive user keys | Automation accounts, scripts |

> **Recommendation:** Use a passphrase for your personal keys. Combine with `ssh-agent` for convenience without losing security.

### Overwriting Warning

```bash
ssh-keygen           # if ~/.ssh/id_ed25519 exists, it ASKS before overwriting
```

> ⚠️ If you overwrite your default key and have no backup, you lose access to every system where you deployed the corresponding public key. Be very careful with `ssh-keygen` on existing files.

---

## Deploying Your Public Key — `ssh-copy-id`

The simplest way to install your public key on a remote server:

```bash
ssh-copy-id alice@remote.example.com                    # uses default id_ed25519.pub
ssh-copy-id -i ~/.ssh/custom_key.pub alice@remote       # specify a particular key
```

`ssh-copy-id` does the following on the remote server:

1. Creates `~/.ssh/` if needed (with mode `700`)
2. Creates `~/.ssh/authorized_keys` if needed (with mode `600`)
3. Appends your public key to `authorized_keys`

### Manual Alternative

If `ssh-copy-id` is not available:

```bash
cat ~/.ssh/id_ed25519.pub | ssh alice@remote \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
     cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Critical File Permissions

SSH is paranoid about permissions — if these are too loose, SSH will silently refuse to use them:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519           # private key
chmod 644 ~/.ssh/id_ed25519.pub       # public key
chmod 600 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/known_hosts
```

---

## Using Your SSH Key

```bash
ssh alice@remote.example.com                    # uses default keys automatically
ssh -i ~/.ssh/custom_key alice@remote           # specify a particular key
ssh -p 2222 alice@remote                        # non-standard port
ssh -v alice@remote                             # verbose (troubleshoot)
ssh -vv alice@remote                            # more verbose
ssh -vvv alice@remote                           # most verbose
```

### Running Commands Without a Full Login

```bash
ssh alice@remote 'hostname'                     # run one command, then exit
ssh alice@remote 'uptime; df -h'                # multiple commands
ssh alice@remote 'cat /var/log/messages' | grep error   # pipe remotely
```

---

## `ssh-agent` — Passphrase Convenience Without Insecurity

`ssh-agent` caches your passphrase in memory so you only enter it once per session:

```bash
# Start the agent (usually started automatically in GNOME)
eval $(ssh-agent)

# Add your key (prompts for passphrase once)
ssh-add ~/.ssh/id_ed25519
ssh-add ~/.ssh/another-key

# List loaded keys
ssh-add -l

# Remove a specific key from the agent
ssh-add -d ~/.ssh/id_ed25519

# Remove ALL keys from the agent
ssh-add -D
```

After adding a key, `ssh` commands use it automatically without prompting.

> **When you log out**, the agent and all its cached passphrases are cleared. You must `ssh-add` again next session.

---

## Client-Side Configuration — `~/.ssh/config`

Shortcut settings for connections you use often:

```
# ~/.ssh/config
Host prod-web
    HostName web01.prod.example.com
    User admin
    Port 2222
    IdentityFile ~/.ssh/prod-key

Host dev-*
    User developer
    IdentityFile ~/.ssh/dev-key

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Now:
```bash
ssh prod-web        # equivalent to: ssh -p 2222 -i ~/.ssh/prod-key admin@web01.prod.example.com
```

Common options worth knowing:

| Option | Purpose |
|---|---|
| `HostName` | The actual hostname or IP to connect to |
| `User` | Default username |
| `Port` | SSH port if not 22 |
| `IdentityFile` | Which private key to use |
| `ServerAliveInterval` | Send keepalive every N seconds (prevents idle disconnect) |
| `ForwardAgent yes` | Forward the ssh-agent to the remote (use carefully) |
| `ProxyJump` | Connect through a bastion host |

---

## Server-Side Configuration — `/etc/ssh/sshd_config`

This is where security hardening happens. After any change, **reload** or **restart** `sshd`:

```bash
sudo systemctl reload sshd
```

### Key Hardening Settings

#### Prevent Direct Root Login

```
# /etc/ssh/sshd_config
PermitRootLogin no                  # no remote root login at all (most secure)
PermitRootLogin prohibit-password   # keys only — no password even for root (RHEL 9+ default)
PermitRootLogin yes                 # allow everything (BAD)
```

**Why this matters:**
- `root` is a known username on every Linux system — attackers only need the password
- Compromising root compromises the entire system
- Logging user → `sudo` gives better audit trail than direct root login

#### Disable Password Authentication

```
PasswordAuthentication no       # keys only — no passwords at all
```

> ⚠️ **Before you set this to `no`**: verify that you have already deployed SSH keys for every user who needs access. Otherwise you will lock yourself (and everyone else) out.

#### Other Important Settings

```
Port 22                         # non-standard port reduces log noise from bots
Protocol 2                      # only SSH protocol version 2 (default in modern versions)
PubkeyAuthentication yes        # allow key-based auth (default, required for keys)
PermitEmptyPasswords no         # never allow empty passwords (default)
MaxAuthTries 3                  # limit password guess attempts per connection
MaxSessions 10                  # concurrent sessions per connection
LoginGraceTime 60               # seconds before disconnecting unauthenticated connections
ClientAliveInterval 300         # send keepalive every 5 minutes
ClientAliveCountMax 2           # drop after 2 missed keepalives
AllowUsers alice bob            # only these users can SSH
AllowGroups sshusers wheel      # only members of these groups
DenyUsers baduser               # explicitly deny users
```

### Testing Config Before Reloading

```bash
sudo sshd -t                    # test config syntax — fails loudly if there is a problem
sudo sshd -T                    # show the full effective config
```

> Always `sshd -t` before `systemctl reload sshd`. A broken config can lock you out.

---

## Troubleshooting SSH

### Client-Side Verbose Mode

Three levels of verbosity:

```bash
ssh -v alice@remote         # basic debug info
ssh -vv alice@remote        # more detail
ssh -vvv alice@remote       # maximum detail (protocol-level)
```

Common things `-v` reveals:
- Which config files are being read
- Which key files are being offered
- Which authentication methods the server accepts
- Why authentication is failing

### Server-Side Logs

```bash
sudo journalctl -u sshd -f                      # follow SSH service logs
sudo journalctl -u sshd --since "10 min ago"    # recent entries
sudo grep sshd /var/log/secure                  # on older systems
```

### Common Failure Modes

| Symptom | Likely Cause |
|---|---|
| "Permission denied (publickey)" with no password prompt | `PasswordAuthentication no` + your key is not in `authorized_keys` |
| "Permission denied (password)" | Wrong password (or `PermitRootLogin no` if logging in as root) |
| "Host key verification failed" | Known hosts entry does not match — server changed or MITM |
| Connection hangs then times out | Firewall blocking port 22, or server not running `sshd` |
| SSH refuses to use your key | Wrong permissions on `~/.ssh/` or the key file (should be 700/600) |
| "Authentication refused: bad ownership or modes" | Something in `~/.ssh/` has group or other write permission |

---

## Real-World Workflow Examples

### First-Time Key Setup (Typical Developer Workflow)

```bash
# On your laptop — once
ssh-keygen -t ed25519 -C "alice@laptop"
# Accept defaults, use a strong passphrase

# Deploy to every server you use
ssh-copy-id alice@server1.example.com
ssh-copy-id alice@server2.example.com

# Load passphrase into agent
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519

# Now ssh works without typing the passphrase again this session
ssh alice@server1.example.com
```

### Hardening a New Server (Production Checklist)

```bash
# 1. Ensure your SSH key is deployed first
ssh-copy-id admin@newserver

# 2. Test key-based login works
ssh admin@newserver 'hostname'

# 3. Connect and edit sshd_config
ssh admin@newserver
sudo vim /etc/ssh/sshd_config
# Set:
#   PermitRootLogin no
#   PasswordAuthentication no
#   AllowUsers admin

# 4. Test config syntax BEFORE reloading
sudo sshd -t

# 5. Reload the service
sudo systemctl reload sshd

# 6. IMPORTANT: keep existing session open, test from a NEW terminal
# If the new connection works, the change is safe
# If it fails, you still have the original session to fix it
```

### Rotating SSH Keys

```bash
# Generate a new key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new

# Deploy the new key alongside the old one
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub alice@remote

# Verify the new key works
ssh -i ~/.ssh/id_ed25519_new alice@remote 'date'

# Remove the old key from the remote authorized_keys
ssh alice@remote "sed -i '/old-key-comment/d' ~/.ssh/authorized_keys"

# Replace local default key
mv ~/.ssh/id_ed25519_new ~/.ssh/id_ed25519
mv ~/.ssh/id_ed25519_new.pub ~/.ssh/id_ed25519.pub
```

---

## Windows Comparison

| Windows | Linux (SSH) | Notes |
|---|---|---|
| Remote Desktop (RDP) | `ssh` | Remote access (SSH is CLI, RDP is GUI) |
| Credential Manager | `~/.ssh/config` + `ssh-agent` | Store connection details and cache credentials |
| PowerShell Remoting / WinRM | `ssh alice@remote 'command'` | Run remote commands |
| `ssh.exe` (Windows 10+) | Same `ssh` | Windows now has OpenSSH built in |
| Server certificate trust prompts | Host key fingerprint prompt | First-connection verification |
| Certificate store | `~/.ssh/known_hosts` | Stored server identities |
| `net use \\server /user:alice` | `ssh alice@server` | Authenticate to remote system |
| SMB share | `sftp` / `scp` / `rsync` | File transfer over SSH |
| Firewall rule for port 3389 | Firewall rule for port 22 | Default ports |

---

## Things to Remember

- **Use Ed25519 keys** — modern, secure, fast. RSA 4096-bit if you need compatibility with older systems.
- **Always use a passphrase on your private key** — combine with `ssh-agent` for convenience.
- **Your private key NEVER leaves your machine** — only the public key is deployed to servers.
- **`ssh-copy-id` is the safest way to deploy public keys** — handles permissions and appends instead of overwriting.
- **File permissions matter** — `~/.ssh` must be `700`, private keys `600`. SSH refuses to use keys with wrong permissions.
- **Test `sshd -t` before reloading** — a syntax error can lock you out of your server.
- **Keep an existing session open when testing hardening changes** — if the new session fails, you still have a way in to fix it.
- **`PermitRootLogin prohibit-password`** (RHEL default) — allows root with keys only, blocks password-based root login. Good balance.
- **Never disable `PasswordAuthentication`** without first confirming key-based auth works for everyone who needs access.
- **The first-connection "trust on first use" prompt** is a known weakness — for high-security environments, pre-populate `/etc/ssh/ssh_known_hosts` via configuration management.
- **`ssh -v` is your friend** — when connections fail mysteriously, verbose mode almost always shows why.
- **Rotate keys periodically** — especially when people leave the team or if you suspect a key has been compromised.

---

Congratulations — you have completed RH124! From here, the next steps are:

- **RH134 (Red Hat System Administration II)** — filesystem management, LVM, storage, logging, crond/timers, firewalls
- **RH294 (Ansible)** — automate everything you have learned
- **EX200 (RHCSA certification)** — hands-on exam that tests the practical skills from RH124 and RH134
