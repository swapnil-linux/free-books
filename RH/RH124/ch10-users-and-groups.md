# Chapter 10 – Managing Local Users and Groups
## RH124 Student Quick Reference

---

## The Big Picture — Why Users and Groups Exist

In Linux, **every process runs as a user, and every file is owned by a user and a group**. This is the foundation of all access control on the system. Nothing happens outside the context of a user identity.

> **Windows equivalent:** Like Windows user accounts and security groups — but in Linux this model applies to everything, including background services, web servers, and database engines. Each service runs as its own dedicated user account to limit what damage it can do if compromised.

---

## User Account Types

| Type | Who | UID Range | Can Log In? |
|---|---|---|---|
| **Superuser** | `root` — full system control | 0 | Yes (avoid direct login) |
| **System users** | Services — `apache`, `mysql`, `sshd` | 1–999 | No (no interactive shell) |
| **Regular users** | Human users | 1000+ | Yes |

> Linux assigns UIDs starting at **1000** for regular users. UIDs below 1000 are reserved for the OS and services. Windows starts regular users at a higher range but the concept is identical.

---

## Key User Information Files

All user and group data is stored as plain text — no registry, no binary database:

| File | Contains | Readable By |
|---|---|---|
| `/etc/passwd` | Username, UID, GID, home dir, shell | Everyone |
| `/etc/shadow` | Password hashes, aging policy | Root only |
| `/etc/group` | Group names, GIDs, members | Everyone |
| `/etc/gshadow` | Group password hashes | Root only |

### `/etc/passwd` Format
```
username:x:UID:GID:comment:home_directory:shell
student:x:1000:1000:Student User:/home/student:/bin/bash
```

### `/etc/shadow` Format
```
username:hashed_password:last_change:min:max:warn:inactive:expire:reserved
```

### `/etc/group` Format
```
groupname:x:GID:member1,member2,member3
wheel:x:10:student,admin1
```

---

## Checking User Information

```bash
id                          # show your own UID, GID, and group memberships
id username                 # show another user's info
whoami                      # just your username
who                         # who is currently logged in
w                           # who is logged in and what they are doing
last                        # recent login history
last username               # login history for a specific user
grep username /etc/passwd   # show user's entry in passwd file
```

---

## Gaining Superuser Access

### `sudo` — Run One Command as Root (Preferred)

```bash
sudo command                # run a single command as root
sudo -i                     # open a full root shell (login shell)
sudo -s                     # open a root shell (non-login)
sudo -l                     # list what sudo commands you are allowed to run
sudo -k                     # clear your cached sudo password immediately
sudo -u username command    # run a command as a different user (not just root)
sudo !!                     # re-run the last command with sudo
```

### `su` — Switch User

```bash
su -                        # switch to root (requires root password)
su - username               # switch to another user (requires their password)
su username                 # switch user without resetting environment
```

> **`sudo` vs `su`:** `sudo` uses your own password and logs every command. `su` requires the target user's password. On most modern Linux systems, `sudo` is the preferred approach — many systems have root login disabled entirely.

---

## sudo Configuration

### The `wheel` Group (RHEL / CentOS / Fedora)

Adding a user to the `wheel` group gives them full sudo access:

```bash
sudo usermod -aG wheel username
```

> **Equivalent on Ubuntu/Debian:** the group is called `sudo` instead of `wheel`:
> ```bash
> sudo usermod -aG sudo username
> ```

### Configuring sudo — `/etc/sudoers`

**Always edit with `visudo`** — it validates syntax before saving (a syntax error in sudoers can lock you out):

```bash
sudo visudo
```

### Drop-in Config Files (Preferred Method)

Create files in `/etc/sudoers.d/` instead of editing the main file directly:

```bash
# Give a specific user full sudo access
echo "username ALL=(ALL) ALL" > /etc/sudoers.d/username

# Give a group full sudo access (% prefix = group)
echo "%groupname ALL=(ALL) ALL" > /etc/sudoers.d/groupname

# Allow passwordless sudo (use with care — automation accounts)
echo "ansible ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/ansible
```

**sudo rule format explained:**
```
username  ALL=(ALL:ALL)  ALL
   ↑        ↑    ↑  ↑    ↑
   who    any  as  as  any
         host  user grp  cmd
```

---

## Managing User Accounts

### Create a User

```bash
sudo useradd username                       # create user with defaults
sudo useradd -m username                    # create user and home directory
sudo useradd -c "Full Name" username        # with comment/description
sudo useradd -u 1500 username               # specify UID
sudo useradd -g groupname username          # set primary group
sudo useradd -G group1,group2 username      # set supplementary groups
sudo useradd -s /bin/bash username          # set login shell
sudo useradd -d /custom/home username       # set custom home directory
sudo useradd -e 2026-12-31 username         # set account expiry date

# Typical real-world creation with common options
sudo useradd -m -c "Jane Smith" -G wheel jsmith
sudo passwd jsmith                          # set the password immediately after
```

### Modify a User

```bash
sudo usermod -c "New Name" username         # update comment
sudo usermod -s /bin/bash username          # change shell
sudo usermod -d /new/home -m username       # move home directory
sudo usermod -aG groupname username         # add to a group (use -a or you REPLACE groups)
sudo usermod -g groupname username          # change primary group
sudo usermod -L username                    # lock account
sudo usermod -U username                    # unlock account
sudo usermod -e 2026-06-30 username         # set expiry date
sudo usermod -e "" username                 # remove expiry date
sudo usermod -L -e 2026-06-01 username      # lock AND expire (for offboarding)
```

> ⚠️ **`usermod -G` without `-a` REPLACES all supplementary groups.** Always use `-aG` together to add to groups without removing existing ones.

### Delete a User

```bash
sudo userdel username                       # remove user, keep home directory
sudo userdel -r username                    # remove user AND home directory AND mail spool
```

> ⚠️ If you delete a user without `-r`, their files become owned by a phantom UID. If a new user is later assigned the same UID, they inherit those files. Always use `-r` or lock the account instead of deleting.

### Set / Change Passwords

```bash
sudo passwd username                        # set password for another user (as root)
passwd                                      # change your own password
sudo passwd -l username                     # lock (same as usermod -L)
sudo passwd -u username                     # unlock (same as usermod -U)
sudo passwd -e username                     # force password change on next login
```

---

## Managing Group Accounts

### Create a Group

```bash
sudo groupadd groupname                     # create group with next available GID
sudo groupadd -g 5000 groupname             # create group with specific GID
sudo groupadd -r groupname                  # create system group (GID < 1000)
```

### Modify a Group

```bash
sudo groupmod -n newname oldname            # rename a group
sudo groupmod -g 6000 groupname             # change GID
```

### Delete a Group

```bash
sudo groupdel groupname                     # delete group (cannot be anyone's primary group)
```

### Managing Group Membership

```bash
sudo usermod -aG groupname username         # add user to a supplementary group
sudo gpasswd -d username groupname          # remove user from a group
sudo gpasswd -M user1,user2,user3 groupname # set the full member list for a group
```

### Check Group Membership

```bash
groups                      # show your own group memberships
groups username             # show another user's groups
id username                 # more detail — shows UIDs and GIDs
cat /etc/group | grep groupname   # see group entry and all members
```

---

## Primary Group vs Supplementary Groups

| | Primary Group | Supplementary Groups |
|---|---|---|
| How many? | Exactly one | Zero or many |
| Defined in | `/etc/passwd` (GID field) | `/etc/group` |
| Used for | Ownership of new files created by the user | Access control |
| Change with | `usermod -g` | `usermod -aG` |

> When a user creates a file, the file's group owner is set to their **primary group** by default.

---

## Password Aging Policy

### `chage` — Change Age (per-user policy)

```bash
sudo chage -l username                  # list current password aging settings
sudo chage -M 90 username               # maximum 90 days before password must change
sudo chage -m 7 username                # minimum 7 days before password can be changed
sudo chage -W 14 username               # warn user 14 days before expiry
sudo chage -I 30 username               # lock account 30 days after password expires
sudo chage -E 2026-12-31 username       # set account expiration date
sudo chage -E -1 username               # remove account expiration
sudo chage -d 0 username                # force password change on next login
```

### All options in one command

```bash
sudo chage -m 0 -M 90 -W 7 -I 14 username
```

### System-wide Defaults — `/etc/login.defs`

Applies to **new users only** — does not change existing accounts:

```bash
sudo vim /etc/login.defs
```

Key settings:
```
PASS_MAX_DAYS   90      # max days before password must change
PASS_MIN_DAYS   0       # min days before password can be changed
PASS_WARN_AGE   7       # days warning before expiry
UID_MIN         1000    # first UID for regular users
GID_MIN         1000    # first GID for regular groups
```

---

## Disabling Logins Without Deleting an Account

### Lock the account (password disabled)

```bash
sudo usermod -L username                # prevents password-based login
                                        # SSH keys still work — see below
```

### Expire the account (blocks ALL login including SSH keys)

```bash
sudo usermod -L -e 2026-06-01 username  # lock + expire — recommended for offboarding
```

### Set shell to nologin

```bash
sudo usermod -s /sbin/nologin username  # interactive login blocked, services can still run
```

---

## The `/sbin/nologin` Shell

Setting a user's shell to `/sbin/nologin` prevents interactive logins but allows the account to still be used by services and processes. Common for system accounts like `apache`, `mysql`, `nobody`.

```bash
# See which accounts use nologin
grep nologin /etc/passwd
```

---

## Windows Comparison

| Windows | Linux | Notes |
|---|---|---|
| User Accounts (Control Panel) | `/etc/passwd` | Plain text file |
| Stored passwords | `/etc/shadow` | Hashed, root-only |
| Active Directory Groups | `/etc/group` | Local groups only |
| Administrator account | `root` (UID 0) | Should not be used directly |
| UAC elevation | `sudo` | Per-command elevation |
| `net user username /add` | `useradd username` | Create user |
| `net user username /delete` | `userdel -r username` | Delete user |
| `net localgroup groupname /add` | `groupadd groupname` | Create group |
| `net localgroup group username /add` | `usermod -aG group username` | Add user to group |
| `net user username *` | `passwd username` | Set password |
| Account expiry (AD) | `chage -E date username` | Per-user account expiry |
| Group Policy password rules | `/etc/login.defs` | System-wide password policy |
| `runas /user:admin command` | `sudo command` | Run as different user |
| Administrators group | `wheel` group (RHEL) / `sudo` group (Ubuntu) | Grants sudo access |

---

## Things to Remember

- **Use `sudo` not `su -`** in day-to-day work — it logs everything and limits exposure
- **Always use `-aG` not just `-G`** when adding a user to a group — `-G` alone replaces all groups
- **Lock, do not delete** accounts for departing users — deleting orphans file ownership to a UID number
- **`userdel -r`** removes home directory — make sure you have backed up anything needed first
- **`visudo`** is the only safe way to edit `/etc/sudoers` — a syntax error can lock you out of root access
- **Password hashes in `/etc/shadow`** use yescrypt on RHEL 10 — much stronger than the old MD5 used in legacy systems
- **`-L` only locks password login** — SSH key authentication still works. Use `usermod -e` to block everything
- **`chage -d 0`** forces a password change on the next login — useful after creating accounts or after a suspected compromise
