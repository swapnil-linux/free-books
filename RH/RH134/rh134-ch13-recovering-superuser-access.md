# RH134 Chapter 13 - Recovering Superuser Access

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Gain administrative access to a system when the superuser password is unknown or locked.

---

## Windows vs Linux: Password Recovery Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Boot from WinPE / Windows Installation media | Boot from RHEL boot.iso (rescue media) |
| `net user Administrator *` from recovery console | `passwd root` inside chroot |
| Safe Mode with Command Prompt | `emergency.target` or `init=/bin/bash` |
| Offline NT Password Editor (3rd party tool) | `init=/bin/bash` method (no media required) |
| BitLocker prevents offline attacks | LUKS full-disk encryption + GRUB2 password |
| Group Policy: require smart card for admin | PAM / sudo configuration |

---

## The Core Security Principle

> **Physical access to a server = potential root access.**
>
> If someone can reach the physical console and GRUB2 is not password-protected, they can reset the root password in under 5 minutes using either method in this chapter. This is not unique to Linux - any OS that allows boot media selection or boot parameter modification faces the same exposure.

### Countermeasures

| Control | What it prevents |
|---|---|
| GRUB2 bootloader password (`grub-setpasswd`) | Prevents editing boot entries without a password |
| LUKS full-disk encryption | Prevents access to data even from rescue media |
| Physical server security (locked DC, BIOS passwords) | Prevents access to the console entirely |
| `SELINUX=enforcing` (default) | Prevents exploiting unlabelled files after recovery |

> These controls are directly relevant to **CIS Benchmarks** (CIS RHEL 1.5.2 - bootloader password), **IRAP** (physical security and access controls), and **PCI DSS** (Requirement 9 - physical access controls).

---

## Two Recovery Methods Overview

| | Method 1: With Rescue Media | Method 2: Without Rescue Media |
|---|---|---|
| Requires | RHEL boot.iso on CD/USB/ISO | Console access only |
| GRUB2 editing | Not needed | Required (append kernel arg) |
| Real system mounted at | `/mnt/sysroot` | `/` (after remount) |
| chroot needed? | Yes - `chroot /mnt/sysroot` | No |
| Risk level | Lower (recommended for production) | Higher (bypass normal boot) |
| How to finish | `exit` twice | `exec /sbin/init` |
| `/.autorelabel` required | Yes | Yes |

---

## Method 1: Reset Root Password Using Rescue Media (Recommended)

Use this method in production environments, or when the GRUB2 boot entry editor is password-protected.

```
Step 1: Reboot and boot from RHEL boot.iso
   - Physical machine: insert CD/USB
   - Virtual machine: mount ISO

Step 2: From the GRUB2 menu:
   Troubleshooting > Rescue a Red Hat Enterprise Linux system

Step 3: At the "Rescue Mode" menu:
   Select option 1: "Continue" (mounts real system at /mnt/sysroot)

Step 4: Change root directory to the real system
```

```bash
bash-5.2# chroot /mnt/sysroot
```

```
Step 5: Set the new root password
```

```bash
bash-5.2# passwd root
New password: yourpassword
Retype new password: yourpassword
passwd: password updated successfully
```

```
Step 6: Create the autorelabel trigger file (CRITICAL - do not skip)
```

```bash
bash-5.2# touch /.autorelabel
```

```
Step 7: Exit chroot, then exit rescue mode (two exit commands)
```

```bash
bash-5.2# exit     # Exit the chroot (back to rescue environment)
bash-5.2# exit     # Exit rescue mode (system reboots)
```

```
Step 8: Wait for the system to:
   - Boot normally
   - Perform a full SELinux relabelling (slow - files scrolling - DO NOT INTERRUPT)
   - Reboot automatically a second time
   - Present a normal login prompt

Step 9: Log in with the new root password
```

---

## Method 2: Reset Root Password Without Rescue Media

Use this method in lab environments, VMs, or when no rescue media is available.

> **Warning:** This method is riskier than using rescue media. The kernel boots with bash as PID 1, bypassing normal startup and shutdown sequences.

```
Step 1: Reboot the system
```

```bash
systemctl reboot
# OR send Ctrl+Alt+Del from VM console
```

```
Step 2: At the GRUB2 menu, press any key (not Enter) to interrupt countdown

Step 3: Press E to edit the current boot entry

Step 4: Navigate to the line starting with 'linux' (the kernel command line)

Step 5: IMPORTANT - remove any console= options from the line
   (If console= is present and not removed, the bash prompt may appear
   on the wrong console and be inaccessible)

Step 6: Press Ctrl+E to move to end of the line

Step 7: Append (with a space before):  init=/bin/bash

Step 8: Press Ctrl+X to boot with these changes

Step 9: Wait for the system to boot to the bash prompt:
   bash-5.2#

Step 10: Remount root filesystem read/write (MANDATORY)
```

```bash
bash-5.2# mount -o remount,rw /
```

```
Step 11: Set the new root password
```

```bash
bash-5.2# passwd
New password: yourpassword
Retype new password: yourpassword
passwd: password updated successfully
```

> **"BAD PASSWORD" warnings** (e.g. "The password is shorter than 8 characters") are advisory only. The password is still set successfully. `passwd` reports `password updated successfully` regardless of the warning. In production, choose a strong password that meets the policy.

```
Step 12: Create the autorelabel trigger file (CRITICAL - do not skip)
```

```bash
bash-5.2# touch /.autorelabel
```

```
Step 13: Hand off from recovery bash to systemd (DO NOT use 'reboot')
```

```bash
bash-5.2# exec /sbin/init
```

```
Step 14: Wait for the system to:
   - Boot into normal mode
   - Perform a full SELinux relabelling (slow - files scrolling - DO NOT INTERRUPT)
   - Reboot automatically a second time
   - Present a normal login prompt

Step 15: Log in with the new root password
```

---

## Why Each Step Is Critical

### `mount -o remount,rw /`

The kernel mounts the root filesystem read-only when `init=/bin/bash` is used. Without this, `passwd` cannot write the new password to `/etc/shadow`.

```bash
# Verify it worked
mount | grep " on / "
# Before: (ro,relatime,...)
# After:  (rw,relatime,...)
```

### `touch /.autorelabel`

When `passwd` runs outside of a normal SELinux-aware session (during recovery), it recreates `/etc/shadow` as a new file without an SELinux context label. If SELinux is in enforcing mode, the next normal boot will deny login because `/etc/shadow` has no `shadow_t` context.

The `/.autorelabel` file triggers a full filesystem relabelling on the next boot. Every file, including the newly created `/etc/shadow`, receives the correct SELinux context.

```bash
# Without /.autorelabel:
# - Password IS changed on disk
# - Boot appears to succeed
# - Login fails with "Authentication failure" (SELinux denies /etc/shadow access)

# With /.autorelabel:
# - Full relabel runs on next boot (visible as file names scrolling)
# - System reboots automatically a second time
# - Login succeeds
```

> **The relabelling boot looks like the system is hanging.** File names scroll past as each file is relabelled. This is normal. On a VM this takes 30 seconds to 2 minutes. Do not interrupt it.

### `exec /sbin/init` (not `reboot`)

In the `init=/bin/bash` method, bash is running as PID 1. Systemd is not running. The `reboot` command communicates with systemd (PID 1) - but bash is PID 1, so `reboot` fails. `exec /sbin/init` replaces the current bash process (PID 1) with systemd, which then runs the normal startup and shutdown sequence properly.

```bash
# WRONG - reboot binary tries to signal systemd which is not running
bash-5.2# reboot
# Result: may hang or fail

# CORRECT - replaces bash PID 1 with systemd
bash-5.2# exec /sbin/init
# Result: systemd takes over and boots normally
```

### `chroot /mnt/sysroot` (rescue media method only)

When booted from rescue media, you are running the rescue environment's shell. The real system's disk is mounted at `/mnt/sysroot`. Without `chroot`, any commands (including `passwd`) operate on the rescue environment in RAM, not on the real disk. `chroot` changes the root directory reference to `/mnt/sysroot` so all subsequent commands act on the real system.

```
Without chroot:       passwd modifies the rescue environment's /etc/shadow (in RAM, discarded on exit)
With chroot:          passwd modifies /mnt/sysroot/etc/shadow (on the real disk)
```

---

## Quick Reference: Step Comparison

| Step | With Rescue Media | Without Rescue Media |
|---|---|---|
| Boot source | RHEL boot.iso | Normal disk boot (modified) |
| GRUB edit | No | Yes - append `init=/bin/bash` |
| Remove `console=` options | No | Yes - required |
| Remount rw | No (already rw) | `mount -o remount,rw /` |
| Change root directory | `chroot /mnt/sysroot` | Not needed |
| Set password | `passwd root` | `passwd` |
| Trigger SELinux relabel | `touch /.autorelabel` | `touch /.autorelabel` |
| Finish | `exit; exit` | `exec /sbin/init` |

---

## Key Files

| File | Purpose |
|---|---|
| `/etc/shadow` | Stores hashed root password (modified by `passwd`) |
| `/.autorelabel` | Triggers full filesystem SELinux relabelling on next boot |
| `/mnt/sysroot` | Mount point for real system disk when in rescue media environment |

---

## Things to Remember

1. **Physical console access + no GRUB2 password = root access.** Both methods in this chapter require only physical console access and knowledge of the procedure. Protect servers with GRUB2 bootloader passwords and physical access controls.

2. **`touch /.autorelabel` is mandatory - skipping it causes login failure.** The `passwd` command recreates `/etc/shadow` without a SELinux context. Without a relabel, SELinux denies login even though the password is correct. The symptom looks identical to a wrong password.

3. **The relabelling boot is slow and looks like a hang - do not interrupt it.** File names scroll past as every file is relabelled. On a VM this takes up to 2 minutes. Interrupting a relabel can leave the system in an inconsistent state.

4. **In the `init=/bin/bash` method, root is initially read-only.** Run `mount -o remount,rw /` immediately. Without it, `passwd` fails because it cannot write to `/etc/shadow`.

5. **Remove `console=` options from the kernel line in the `init=/bin/bash` method.** Leaving `console=ttyS0` (serial console) redirects output away from the screen. The bash prompt appears on the wrong device and you cannot access it.

6. **Use `exec /sbin/init` to finish the `init=/bin/bash` method, not `reboot`.** Bash is PID 1 in this scenario. Systemd is not running. `exec /sbin/init` correctly hands off PID 1 to systemd for a normal boot. `reboot` will fail or produce unexpected behaviour.

7. **In the rescue media method, `chroot /mnt/sysroot` is required before running `passwd`.** Without it, `passwd` modifies the rescue environment in RAM, not the real disk. The real system's password remains unchanged.

8. **Exit twice in the rescue media method.** First `exit` leaves the chroot. Second `exit` leaves the rescue environment and triggers a reboot.

9. **"BAD PASSWORD" warnings do not prevent the password from being set.** `passwd` will still report `password updated successfully`. In production, use a strong password to avoid the warning and meet policy requirements.

10. **`/.autorelabel` causes the system to reboot twice.** First boot: relabelling runs. System reboots automatically. Second boot: normal startup. Expect two reboots before the login prompt appears.
