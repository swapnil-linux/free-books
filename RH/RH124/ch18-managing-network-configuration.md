# Chapter 18 – Managing Network Configuration
## RH124 Student Quick Reference

---

## What Is NetworkManager?

**NetworkManager** is the service that handles all network configuration on modern RHEL. It decides what IP address an interface gets, manages DNS settings, handles VPN connections, and switches between Wi-Fi networks automatically on laptops.

```
        ┌────────────────────────┐
        │    NetworkManager       │  ← manages everything
        │    (runs as systemd     │
        │     service)            │
        └───────────┬────────────┘
                    │
        ┌───────────┴────────────┐
        │    Connection profiles  │  ← settings for each connection
        │    (.nmconnection files)│
        └───────────┬────────────┘
                    │
              applied to
                    │
        ┌───────────┴────────────┐
        │   Network devices       │  ← physical/virtual interfaces
        │   (ens3, eno1, etc.)    │
        └────────────────────────┘
```

> **Windows equivalent:** NetworkManager is similar to the **Network Connections** control panel combined with **netsh** — it stores named profiles per interface and applies them automatically.

---

## Device vs Connection — A Critical Distinction

This is one of the most confusing concepts in RHEL networking. Get this right and everything else makes sense:

| | **Device** | **Connection** |
|---|---|---|
| What it is | A physical or virtual network interface | A configuration profile |
| Example | `ens3`, `eno1`, `wlp3s0` | "Home Wi-Fi", "Office LAN", "static-ens3" |
| Count | One per NIC | A device can have many profiles, only one active at a time |

> **Real-world example:** A laptop's Wi-Fi card (one **device** = `wlp3s0`) might have three **connections**: "Office", "Home", "Coffee Shop" — each with different IP/DNS settings. NetworkManager applies whichever one matches.

---

## The `nmcli` Command — Your Daily Tool

`nmcli` is the command-line interface for NetworkManager. It has a specific object-verb syntax:

```bash
nmcli <OBJECT> <ACTION>

Objects: device (dev), connection (con), general (gen), networking, radio
Actions: show, status, up, down, add, modify, delete, reload
```

### Abbreviations Save Typing

All `nmcli` commands accept short forms:

```bash
nmcli connection show           # full form
nmcli con show                  # short form
nmcli c s                       # shortest form that is unambiguous

nmcli device status             # full form
nmcli dev status                # short form
nmcli d s                       # shortest form
```

---

## Viewing Network Information

### Device Status

```bash
nmcli device status             # which devices are connected, and to which profile
nmcli dev show                  # full details for all devices
nmcli dev show ens3             # full details for one device
```

Sample output:
```
DEVICE  TYPE      STATE      CONNECTION
ens3    ethernet  connected  ethernet-ens3
lo      loopback  connected  (externally)
```

### Connection Profiles

```bash
nmcli connection show           # list all connection profiles
nmcli con show --active         # only active connections
nmcli con show ethernet-ens3    # full settings for one profile
nmcli -f ipv4.addresses con show ethernet-ens3   # one specific field
```

Sample output:
```
NAME              UUID                                  TYPE      DEVICE
ethernet-ens3     27afa607-ee36-43f0-b8c3-9d245cdc4bb3  ethernet  ens3
static-addr       c2131041-1fec-4563-9856-3bc582801758  ethernet  --
```

> The `--` in the DEVICE column means the profile exists but is **not active**.

---

## Creating a New Connection Profile

### DHCP (Automatic IP)

```bash
sudo nmcli con add \
  type ethernet \
  con-name "office-dhcp" \
  ifname ens3 \
  ipv4.method auto
```

### Static IP

```bash
sudo nmcli con add \
  type ethernet \
  con-name "static-ens3" \
  ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "192.168.1.1,8.8.8.8" \
  ipv4.dns-search "example.com"
```

---

## Modifying an Existing Connection

```bash
# Change IP address
sudo nmcli con mod static-ens3 ipv4.addresses 192.168.1.150/24

# Change gateway
sudo nmcli con mod static-ens3 ipv4.gateway 192.168.1.254

# Switch from static to DHCP
sudo nmcli con mod static-ens3 ipv4.method auto

# Switch from DHCP to static
sudo nmcli con mod ethernet-ens3 ipv4.method manual
sudo nmcli con mod ethernet-ens3 ipv4.addresses 192.168.1.100/24
sudo nmcli con mod ethernet-ens3 ipv4.gateway 192.168.1.1
```

### Adding vs Replacing Values

Some settings accept multiple values (like DNS servers). The `+` and `-` prefixes matter:

```bash
# REPLACE all DNS servers with just one
sudo nmcli con mod static-ens3 ipv4.dns 8.8.8.8

# ADD a DNS server to the existing list
sudo nmcli con mod static-ens3 +ipv4.dns 1.1.1.1

# REMOVE a specific DNS server from the list
sudo nmcli con mod static-ens3 -ipv4.dns 8.8.8.8
```

> ⚠️ Without `+` or `-`, the new value **replaces** the entire existing list. Forgetting this has wiped many admins' DNS configurations.

---

## Activating and Deactivating Connections

Changes made with `nmcli con mod` are **saved** but not applied until the connection is reactivated:

```bash
sudo nmcli con up static-ens3          # activate a connection
sudo nmcli con down static-ens3        # deactivate (disconnects the device)
sudo nmcli dev disconnect ens3         # disconnect the device itself
sudo nmcli dev connect ens3            # reconnect using its saved profile
```

### The Important Workflow

```
1. Create or modify the profile     (nmcli con add / nmcli con mod)
2. Activate it                       (nmcli con up <name>)
3. Verify                            (ip addr / nmcli con show --active)
```

---

## Deleting a Connection

```bash
sudo nmcli con delete static-ens3     # removes the profile and the file
```

> This does not affect the device itself — just removes the configuration profile.

---

## Configuration File Locations

NetworkManager stores profiles in three locations:

| Location | Purpose |
|---|---|
| `/etc/NetworkManager/system-connections/` | **Your configs** — administrator edits, persistent |
| `/usr/lib/NetworkManager/system-connections/` | Preconfigured, immutable profiles shipped with packages |
| `/run/NetworkManager/system-connections/` | Temporary profiles, lost on reboot |

Files use the `.nmconnection` extension and INI-style **keyfile format**.

### Example Connection File

```ini
[connection]
id=static-ens3
uuid=27afa607-ee36-43f0-b8c3-9d245cdc4bb3
type=ethernet
autoconnect=true
interface-name=ens3

[ethernet]

[ipv4]
method=manual
address1=192.168.1.100/24
gateway=192.168.1.1
dns=192.168.1.1;8.8.8.8;
dns-search=example.com

[ipv6]
method=auto
```

---

## Editing Config Files Directly

Sometimes it is faster to edit files directly than to run multiple `nmcli` commands. The workflow:

```bash
# 1 — Edit the connection file
sudo vim /etc/NetworkManager/system-connections/static-ens3.nmconnection

# 2 — Tell NetworkManager to reload all profiles
sudo nmcli con reload

# 3 — Activate the changes (they are NOT applied just from reload)
sudo nmcli con up static-ens3
```

### `nmcli` vs Direct File Editing

| | `nmcli` commands | Direct file edit |
|---|---|---|
| Speed to apply | Immediate | Needs `con reload` then `con up` |
| Good for | Small changes, quick edits | Complex configs, templating, multiple servers |
| Risk | Low (syntax checked automatically) | Typos break things silently |
| Auditable | Commands can be logged | Leaves file timestamps |

---

## IPv4 vs IPv6 Settings

Keyfile settings mirror `nmcli` settings:

| Purpose | `nmcli` setting | Keyfile section |
|---|---|---|
| Auto IP (DHCP) | `ipv4.method auto` | `[ipv4]` `method=auto` |
| Static IP | `ipv4.method manual` | `[ipv4]` `method=manual` |
| IP address | `ipv4.addresses 1.2.3.4/24` | `address1=1.2.3.4/24` |
| Gateway | `ipv4.gateway 1.2.3.1` | `gateway=1.2.3.1` |
| DNS server | `ipv4.dns 8.8.8.8` | `dns=8.8.8.8;` |
| Search domain | `ipv4.dns-search example.com` | `dns-search=example.com` |
| Ignore DHCP DNS | `ipv4.ignore-auto-dns yes` | `ignore-auto-dns=true` |

The IPv6 equivalents use `ipv6.` instead of `ipv4.`:

```bash
# IPv6 with SLAAC (router advertisement)
sudo nmcli con mod static-ens3 ipv6.method auto

# IPv6 with DHCPv6 (no SLAAC)
sudo nmcli con mod static-ens3 ipv6.method dhcp

# IPv6 static
sudo nmcli con mod static-ens3 ipv6.method manual
sudo nmcli con mod static-ens3 ipv6.addresses 2001:db8::10/64
sudo nmcli con mod static-ens3 ipv6.gateway 2001:db8::1
```

---

## Hostname Management

A system has **three** kinds of hostnames:

| Type | Where Set | When Used |
|---|---|---|
| **Static** | `/etc/hostname` | Persistent across reboots |
| **Transient** | Kernel (from DHCP or reverse DNS) | Runtime only, overridden by static |
| **Pretty** | systemd metadata | For display in UIs |

### `hostname` — Temporary (Runtime Only)

```bash
hostname                            # show current hostname
sudo hostname web1.example.com      # change temporarily — lost on reboot
```

### `hostnamectl` — Permanent

```bash
hostnamectl                                  # show all hostname info
hostnamectl status                           # same
hostnamectl hostname                         # show static hostname
sudo hostnamectl hostname web1.example.com   # set persistently
```

### Direct File Editing

```bash
sudo vim /etc/hostname
# put your FQDN on one line, save
# takes effect on next boot or after 'hostnamectl' is run
```

> **Note:** Your shell prompt may still show the old hostname after a change because shells cache environment variables. Open a new session or log out/in to see the update.

---

## Name Resolution

### The Resolution Order — `/etc/nsswitch.conf`

When something asks "what IP is `web1.example.com`?", Linux checks sources in the order defined in `/etc/nsswitch.conf`:

```
hosts: files dns myhostname
```

This means:
1. **files** — check `/etc/hosts` first
2. **dns** — query DNS servers from `/etc/resolv.conf`
3. **myhostname** — fall back to local hostname

### `/etc/hosts` — Local Overrides

```bash
cat /etc/hosts
# 127.0.0.1   localhost localhost.localdomain
# ::1         localhost localhost.localdomain
# 192.168.1.50  server1.example.com server1 primary-server
```

Add an entry to override DNS for a specific host:

```bash
sudo vim /etc/hosts
# append: 192.168.1.99 database.internal.example.com database
```

> **Windows equivalent:** `C:\Windows\System32\drivers\etc\hosts` — identical format and purpose.

### `/etc/resolv.conf` — DNS Servers

```bash
cat /etc/resolv.conf
# Generated by NetworkManager
# search example.com
# nameserver 8.8.8.8
# nameserver 1.1.1.1
```

> ⚠️ **Do not edit `/etc/resolv.conf` directly on RHEL.** NetworkManager automatically overwrites it based on connection profile DNS settings. Change DNS via `nmcli con mod ... ipv4.dns ...` instead.

### Testing Name Resolution

```bash
host web1.example.com              # basic DNS lookup
host 192.168.1.50                  # reverse lookup (IP → hostname)
dig web1.example.com               # detailed DNS query
dig +short web1.example.com        # just the answer
dig @8.8.8.8 web1.example.com      # query a specific DNS server
dig MX example.com                 # query mail exchange records
dig -x 192.168.1.50                # reverse lookup

# Test what the system ACTUALLY resolves (includes /etc/hosts)
getent hosts web1.example.com
```

> **Critical distinction:** `host` and `dig` go **directly to DNS** — they ignore `/etc/hosts`. `getent hosts` follows the full nsswitch order, so it shows what applications will actually see.

---

## Real-World Workflow Examples

### Change Static IP on a Running System

```bash
# Inspect current state
nmcli con show --active
ip -br addr

# Change the IP address
sudo nmcli con mod static-ens3 ipv4.addresses 192.168.1.150/24

# Apply the change
sudo nmcli con up static-ens3

# Verify
ip -br addr
```

### Switch a Server from DHCP to Static

```bash
# Find the active connection name
nmcli con show --active
# (assume it shows: ethernet-ens3)

# Change to static and configure
sudo nmcli con mod ethernet-ens3 ipv4.method manual
sudo nmcli con mod ethernet-ens3 ipv4.addresses 192.168.1.100/24
sudo nmcli con mod ethernet-ens3 ipv4.gateway 192.168.1.1
sudo nmcli con mod ethernet-ens3 ipv4.dns "8.8.8.8,1.1.1.1"

# Reactivate
sudo nmcli con up ethernet-ens3

# Verify
ip -br addr
cat /etc/resolv.conf
```

### Adding a Secondary IP Address

```bash
sudo nmcli con mod static-ens3 +ipv4.addresses 192.168.1.101/24
sudo nmcli con up static-ens3
ip addr show ens3
```

### Change Hostname and Update Local Hosts File

```bash
sudo hostnamectl hostname db1.example.com
echo "192.168.1.99 db-backup.example.com db-backup" | sudo tee -a /etc/hosts
```

---

## Windows Comparison

| Windows | Linux (NetworkManager) | Notes |
|---|---|---|
| Network Connections control panel | `nmcli con show` | List connections |
| Network adapter Properties | `nmcli con show <name>` | View connection details |
| IPv4 Properties → "Obtain automatically" | `ipv4.method auto` | DHCP |
| IPv4 Properties → "Use the following" | `ipv4.method manual` | Static IP |
| Preferred DNS server | `ipv4.dns` | DNS configuration |
| `netsh interface ip set address` | `nmcli con mod ... ipv4.addresses` | Set IP from CLI |
| `netsh interface ip set dns` | `nmcli con mod ... ipv4.dns` | Set DNS from CLI |
| `ipconfig /release` + `/renew` | `nmcli con down && nmcli con up` | DHCP release/renew |
| Rename computer (System Properties) | `hostnamectl hostname <name>` | Change hostname |
| `C:\Windows\System32\drivers\etc\hosts` | `/etc/hosts` | Local hostname overrides |
| Multiple network profiles (Wi-Fi) | Multiple `.nmconnection` files | Named connection profiles |
| Ethernet "enabled/disabled" | `nmcli dev connect/disconnect` | Enable/disable interface |

---

## Things to Remember

- **Device ≠ connection** — a device is hardware; a connection is a profile. One device can have many profiles.
- **`nmcli con mod` changes persist** — they go straight to the connection file on disk.
- **`nmcli con mod` changes do NOT apply until `nmcli con up`** — editing without activating leaves the live config unchanged.
- **`+` and `-` prefixes** matter for multi-value settings like `ipv4.dns` — without them, you replace the whole list.
- **Never edit `/etc/resolv.conf` directly** — NetworkManager will overwrite it. Use `nmcli con mod ... ipv4.dns ...`.
- **`hostname` is temporary, `hostnamectl hostname` is permanent** — use `hostnamectl` for any real change.
- **Shell prompt caches hostname** — start a new shell to see the updated name in your prompt.
- **`host` and `dig` skip `/etc/hosts`** — they go directly to DNS. Use `getent hosts` to see what applications actually resolve to.
- **After editing config files manually**, always run `nmcli con reload` followed by `nmcli con up <name>`.
- **RHEL 10 dropped `ifcfg-*` files** — only the `.nmconnection` keyfile format works now. If you find old docs referencing `/etc/sysconfig/network-scripts/`, it does not apply to RHEL 10.
