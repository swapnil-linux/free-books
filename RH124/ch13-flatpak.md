# Chapter 13 – Installing and Updating Applications by Using Flatpak
## RH124 Student Quick Reference

---

## What Is Flatpak and Why Does It Exist?

Traditional Linux software packaging (RPM, DEB) has a significant limitation: packages are built specifically for one distro and one version. An app packaged for RHEL 10 will not work on Ubuntu 24.04 without repackaging.

Flatpak solves this by bundling the application with all the libraries it needs, making it **distro-independent**. One Flatpak package works on RHEL, Ubuntu, Fedora, Debian, and any other Linux distro that supports Flatpak.

```
Traditional RPM/DEB:
  App → depends on → system libraries (version must match exactly)

Flatpak:
  App → bundled with → its own libraries (in a sandbox, isolated from the OS)
```

> **Windows equivalent:** Think of Flatpak like a portable application or a Microsoft Store app that ships with everything it needs — no installer wizard, no DLL conflicts, just install and run.

---

## Flatpak vs DNF/RPM — When to Use Which

| | DNF/RPM | Flatpak |
|---|---|---|
| **Best for** | System software, servers, CLI tools, services | Desktop GUI applications |
| **Distro support** | RHEL/Fedora family only | Any Linux distro |
| **Library sharing** | Shares system libraries (smaller) | Bundles own libraries (larger) |
| **Isolation** | Runs with full system access | Sandboxed — limited system access |
| **Update model** | Tied to RHEL release cycle | App updates independently |
| **RHEL server use** | Yes — the primary method | Rarely — mostly desktop use |

> **Rule of thumb:** Use DNF for anything on a server. Use Flatpak for desktop applications where you want the latest version regardless of what RHEL ships.

---

## Key Concepts

### Runtimes

A **runtime** is a shared set of libraries that Flatpak applications depend on. Rather than every app bundling every library, multiple apps share one runtime.

```
org.freedesktop.Platform   ← runtime (shared libraries)
    ↑              ↑
org.mozilla.firefox    md.obsidian.Obsidian   ← apps that use it
```

You must have the runtime installed before an app that needs it can run. Flatpak installs required runtimes automatically when you install an app.

### Sandboxing

Every Flatpak app runs in a **sandbox** — it is isolated from the rest of the system. By default it cannot:
- Access files outside its own data directory
- Read other users' files
- Access arbitrary hardware
- See other running processes

Permissions can be granted explicitly (e.g. access to the home directory, camera, microphone). This makes Flatpak apps significantly safer than traditionally installed software.

### Application Identifiers

Flatpak uses a **reverse-domain naming convention** identical to Android and Java package names:

```
com.vscodium.codium
org.mozilla.Thunderbird
md.obsidian.Obsidian
org.gimp.GIMP

domain.organisation.AppName
```

You can use either the **full ID** or just the **short name**:
```bash
flatpak install org.mozilla.Thunderbird   # full ID
flatpak install Thunderbird               # short name (if unambiguous)
```

### Identifier Triple

When specifying an exact version, use the **identifier triple**:

```
com.redhat.Platform / x86_64 / el10
        │                │       │
   Application ID    Architecture  Branch (version)
```

---

## Remotes — Where Flatpak Gets Software

A **remote** is a Flatpak repository — equivalent to a DNF repository. You must add a remote before you can install apps from it.

### Common Remotes

| Remote | URL | Contents | Support |
|---|---|---|---|
| **rhel** | `flatpaks.redhat.io` | Red Hat curated apps | Officially supported by Red Hat |
| **Flathub** | `flathub.org` | Largest collection — thousands of apps | Community supported |
| **Fedora** | `registry.fedoraproject.org` | Fedora-maintained apps | Community supported |

> Red Hat does **not** support applications from third-party remotes (Flathub, Fedora). Use them at your own discretion in production.

---

## Managing Remotes

```bash
# List configured remotes
flatpak remotes
flatpak remotes -d                          # verbose — shows URLs and options

# Add a remote (system-wide — available to all users)
flatpak remote-add --if-not-exists flathub \
  https://dl.flathub.org/repo/flathub.flatpakrepo

# Add a remote (current user only)
flatpak remote-add --if-not-exists --user flathub \
  https://dl.flathub.org/repo/flathub.flatpakrepo

# Add a remote from a .flatpakrepo file
flatpak remote-add --if-not-exists myrepo \
  http://internal-server/myrepo.flatpakrepo

# Disable a remote (keep it configured but stop using it)
flatpak remote-modify --disable flathub

# Re-enable a remote
flatpak remote-modify --enable flathub

# Delete a remote (removes all apps installed from it)
flatpak remote-delete flathub

# List what is available in a remote
flatpak remote-ls flathub
flatpak remote-ls --app flathub             # apps only (not runtimes)
```

---

## Finding and Installing Applications

### Searching

```bash
flatpak search firefox                      # search all enabled remotes
flatpak remote-ls flathub | grep -i code    # list and filter a specific remote
```

### Installing

```bash
# Install an application (Flatpak resolves and installs required runtime)
flatpak install org.mozilla.Thunderbird

# Install from a specific remote
flatpak install flathub org.mozilla.Thunderbird

# Install a specific branch/version
flatpak install org.mozilla.Thunderbird/x86_64/stable

# Install for current user only (default is system-wide)
flatpak install --user org.mozilla.Thunderbird

# Install without prompting (useful in scripts)
flatpak install -y org.mozilla.Thunderbird

# Install or update if already installed
flatpak install --or-update org.mozilla.Thunderbird

# Install from a local .flatpak file
flatpak install ./application.flatpak
```

---

## Managing Installed Applications

### Listing

```bash
flatpak list                                # list all installed apps and runtimes
flatpak list --app                          # apps only
flatpak list --runtime                      # runtimes only
flatpak list --user                         # current user's installations only
flatpak list --system                       # system-wide installations only
```

### Getting Information

```bash
flatpak info org.mozilla.Thunderbird        # detailed info about installed app
flatpak remote-info rhel org.mozilla.Thunderbird  # info from remote (before install)

# Check available branches for an app
flatpak remote-info rhel com.redhat.Platform
```

### Updating

```bash
flatpak update                              # update all installed apps and runtimes
flatpak update org.mozilla.Thunderbird      # update a specific app only
flatpak update --user                       # update user-installed apps only
```

> Updates only happen **within the same branch** — if you installed `stable`, it updates to the latest `stable`, not `beta`.

### Preventing Unwanted Updates

```bash
flatpak mask org.mozilla.Thunderbird        # pin — prevent this app from updating
flatpak mask --remove org.mozilla.Thunderbird  # unpin — allow updates again
```

### Running Applications

```bash
flatpak run org.mozilla.Thunderbird         # launch a Flatpak app from terminal
flatpak run com.vscodium.codium             # launch VSCodium
```

Installed Flatpak apps also appear in the GNOME application launcher — you do not need to use the terminal to launch them.

---

## Uninstalling

```bash
flatpak uninstall org.mozilla.Thunderbird               # uninstall app
flatpak uninstall --delete-data org.mozilla.Thunderbird # uninstall + delete user data
flatpak uninstall --unused                              # remove runtimes not used by any app
flatpak uninstall org.freedesktop.Platform              # remove a runtime
```

> You **cannot remove a runtime** while an app that depends on it is still installed. Uninstall the app first.

---

## System vs User Installation

Flatpak supports two installation scopes. This is different from DNF where everything is system-wide.

| Scope | Flag | Who can use it | Where stored |
|---|---|---|---|
| **System** | `--system` (default) | All users | `/var/lib/flatpak/` |
| **User** | `--user` | Only the installing user | `~/.local/share/flatpak/` |

```bash
# See which scope each installed app uses
flatpak list
# The "Installation" column shows "system" or "user"
```

---

## Authentication for Red Hat's Flatpak Registry

The official Red Hat remote requires a Red Hat account. Log in first with `podman`:

```bash
# Log in to Red Hat's container registry
podman login registry.redhat.io

# Save credentials for the current user permanently
cp $XDG_RUNTIME_DIR/containers/auth.json \
   $HOME/.config/flatpak/oci-auth.json

# Or save system-wide (root required)
sudo cp $XDG_RUNTIME_DIR/containers/auth.json \
        /etc/flatpak/oci-auth.json
sudo chmod 644 /etc/flatpak/oci-auth.json
```

---

## Configuration Files

| Location | Purpose |
|---|---|
| `/etc/flatpak/remotes.d/` | System-wide remote repository definitions |
| `/etc/flatpak/oci-auth.json` | System-wide authentication tokens |
| `~/.config/flatpak/` | Per-user Flatpak configuration |
| `~/.local/share/flatpak/` | Per-user installed applications |
| `/var/lib/flatpak/` | System-wide installed applications |

---

## GNOME Software — The GUI Alternative

All Flatpak operations can also be done through **GNOME Software** (the graphical app store):

- Browse and search for applications
- Install and uninstall apps
- Manage repositories: Main Menu → Software Repositories
- Check for and apply updates

GNOME Software shows both DNF packages and Flatpak apps in a unified interface — it handles the underlying installation method automatically.

---

## Windows Comparison

| Windows | Flatpak | Notes |
|---|---|---|
| Microsoft Store | Flathub / RHEL remote | App store concept |
| App isolation (sandbox) | Flatpak sandbox | Apps isolated from system |
| Portable App (no install) | `flatpak run --user` | Per-user, no system changes |
| Side-by-side installs | Flatpak branches | Multiple versions coexist |
| App updates via Store | `flatpak update` | Independent of OS updates |
| Store URL / app ID | `com.vendor.AppName` | Unique reverse-domain ID |
| Uninstall + remove data | `flatpak uninstall --delete-data` | Clean removal |
| UWP app packages (`.appx`) | `.flatpak` files | Local install format |

---

## Things to Remember

- **Flatpak is for desktop apps** — it is not the right tool for server software, system services, or CLI tools
- **Runtimes install automatically** with apps — you rarely need to manage them manually
- **Updates stay within the same branch** — `stable` stays on `stable`, it does not jump to `beta`
- **`--user` vs system** — user installs do not require root and do not affect other users
- **Flathub is unsupported by Red Hat** — fine for development workstations, use caution in regulated environments
- **`flatpak uninstall --unused`** is good housekeeping — removes runtimes no longer needed by any installed app
- **`flatpak mask`** prevents an app from updating — useful if a specific version is certified or tested for your environment
- **Sandboxing improves security** — a compromised Flatpak app has much less access to the system than a traditionally installed app
- On **Ubuntu/Debian**, Flatpak is not installed by default but is available via `apt install flatpak` — same commands work across distros
