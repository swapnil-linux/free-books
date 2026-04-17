# Chapter 17 – Introduction to Networking
## RH124 Student Quick Reference

---

## The Big Picture — The TCP/IP Model

All network communication on a Linux system is organised into four layers. Each layer has its own responsibility:

```
┌──────────────────────────────────────────────────────────┐
│ Application Layer   HTTPS, SSH, DNS, SMTP, FTP           │  ← What you use
├──────────────────────────────────────────────────────────┤
│ Transport Layer     TCP, UDP — ports and sockets         │  ← How data flows
├──────────────────────────────────────────────────────────┤
│ Internet Layer      IP (v4 and v6) — addressing, routing │  ← Where data goes
├──────────────────────────────────────────────────────────┤
│ Link Layer          Ethernet, Wi-Fi — MAC addresses      │  ← Physical/local
└──────────────────────────────────────────────────────────┘
```

You may also see references to the 7-layer **OSI model** — admins commonly use "Layer 2" (Ethernet), "Layer 3" (IP), and "Layer 4" (TCP/UDP). These map directly to the TCP/IP layers above.

---

## TCP vs UDP — When to Use Which

| | TCP | UDP |
|---|---|---|
| **Reliable?** | Yes — confirms delivery | No — "fire and forget" |
| **Ordered?** | Yes — data arrives in order | No — packets may arrive out of order |
| **Overhead** | Higher (handshakes, retries) | Lower (minimal) |
| **Connection** | Connection-oriented | Connectionless |
| **Good for** | Web, SSH, email, file transfer | DNS, DHCP, streaming, VoIP, games |

> **Analogy:** TCP is like a phone call — establishes a connection, confirms both ends are listening, retries if something is unclear. UDP is like a postcard — send it, hope it arrives.

---

## IPv4 — The Most Common Addressing

An **IPv4 address** is a 32-bit number, displayed as four decimal numbers (octets) separated by dots:

```
192.168.1.107
 │   │   │ │
 └───┴───┴─┴── Each octet is 0-255 (8 bits = 256 possible values)
```

### Common IPv4 Address Ranges to Remember

| Range | Purpose | Notes |
|---|---|---|
| `10.0.0.0/8` | Private (RFC 1918) | Large corporate networks |
| `172.16.0.0/12` | Private (RFC 1918) | Medium networks |
| `192.168.0.0/16` | Private (RFC 1918) | Home and small office |
| `127.0.0.0/8` | Loopback | `127.0.0.1` = localhost (this machine) |
| `169.254.0.0/16` | Link-local (APIPA) | Auto-assigned when DHCP fails |
| `224.0.0.0/4` | Multicast | One-to-many communication |
| `0.0.0.0/0` | Default route | "All other destinations" |
| `0.0.0.0` | "Any" (listening) | Service listens on all interfaces |

---

## Subnet Masks and CIDR Notation

A **netmask** divides an IP address into a **network prefix** and a **host identifier**.

Two ways to write the same netmask:

```
CIDR notation:   192.168.1.107/24
Decimal:         192.168.1.107 with mask 255.255.255.0
```

The `/24` means the first 24 bits are the network, the remaining 8 bits are the host.

### Common Subnet Mask Reference

| CIDR | Decimal Mask | Hosts | Typical Use |
|---|---|---|---|
| `/8` | `255.0.0.0` | 16,777,214 | Huge corporate network |
| `/16` | `255.255.0.0` | 65,534 | Large network |
| `/24` | `255.255.255.0` | 254 | Typical LAN / home network |
| `/25` | `255.255.255.128` | 126 | Split a /24 in half |
| `/26` | `255.255.255.192` | 62 | Small office |
| `/27` | `255.255.255.224` | 30 | Small subnet |
| `/28` | `255.255.255.240` | 14 | Very small subnet |
| `/29` | `255.255.255.248` | 6 | Tiny subnet (e.g. point-to-point) |
| `/30` | `255.255.255.252` | 2 | Two hosts (router-to-router links) |
| `/32` | `255.255.255.255` | 1 | Single host (loopback, API endpoints) |

### Three Special Addresses in Any Subnet

For `192.168.1.0/24`:
- **Network address** — first address, all host bits zero: `192.168.1.0`
- **Broadcast address** — last address, all host bits one: `192.168.1.255`
- **Usable hosts** — everything in between: `192.168.1.1` to `192.168.1.254`

### `ipcalc` — Do the Math For You

```bash
ipcalc 192.168.1.107/24
# Address:     192.168.1.107
# Network:     192.168.1.0/24
# Netmask:     255.255.255.0
# Broadcast:   192.168.1.255
# HostMin:     192.168.1.1
# HostMax:     192.168.1.254
# Hosts/Net:   254
```

> Every time you catch yourself about to calculate subnets in your head, use `ipcalc` instead. Fewer mistakes, faster answers.

---

## IPv6 — The Future (and Present)

An **IPv6 address** is a 128-bit number, displayed as eight groups of four hexadecimal digits:

```
2001:0db8:0000:0010:0000:0000:0000:0001
  │    │    │    │    │    │    │    │
  └────┴────┴────┴────┴────┴────┴────┴── Each quartet = 16 bits (4 hex digits)
```

### Shortening Rules

1. **Drop leading zeros** in each quartet: `0db8` → `db8`, `0010` → `10`, `0000` → `0`
2. **Replace one run of consecutive zero-quartets with `::`** (only once per address)

```
2001:0db8:0000:0010:0000:0000:0000:0001
            ↓
2001:db8:0:10:0:0:0:1        (after dropping leading zeros)
            ↓
2001:db8:0:10::1             (after collapsing zeros)
```

> ⚠️ The `::` can only appear **once** in an address. `2001:db8::7::2` is invalid because it is ambiguous.

### Common IPv6 Addresses

| IPv6 | Meaning | IPv4 Equivalent |
|---|---|---|
| `::1/128` | Loopback (this machine) | `127.0.0.1` |
| `::` | Unspecified / "any" | `0.0.0.0` |
| `::/0` | Default route | `0.0.0.0/0` |
| `fe80::/10` | Link-local (auto-assigned) | `169.254.0.0/16` |
| `fd00::/8` | Unique local (private-ish) | `10.0.0.0/8`, `192.168.0.0/16` |
| `2000::/3` | Global unicast (public internet) | Public IPv4 space |
| `ff02::1` | All nodes on local link | Broadcast (sort of) |

### IPv6 Subnet Convention

Most IPv6 subnets use a **`/64` prefix** — half the address is network, half is host. With 64 bits for hosts, a single subnet can hold an enormous number of devices (2^64, which is a number with 20 digits).

### Important IPv6 Rules

- When writing an IPv6 address with a port, **enclose it in brackets**: `[2001:db8::1]:443`
- IPv6 has **no broadcast address** — multicast replaces it
- Every IPv6 interface automatically gets a link-local address (`fe80::`)

---

## Network Interface Names

Modern RHEL uses **predictable interface names** that persist across reboots. They no longer change based on detection order (the old `eth0`, `eth1` world).

### Naming Prefixes

| Prefix | Type |
|---|---|
| `en` | Ethernet (wired) |
| `wl` | Wireless (Wi-Fi) |
| `ww` | Wireless WAN (cellular) |
| `lo` | Loopback (always present) |

### Suffix Tells You Where It Is

| Example | Meaning |
|---|---|
| `ens3` | Ethernet, hot-pluggable slot 3 |
| `eno1` | Ethernet, onboard NIC number 1 |
| `enp0s3` | Ethernet, PCI bus 0 slot 3 |
| `wlp3s0` | Wireless, PCI bus 3 slot 0 |

> You will often see **multiple alternative names** for the same device (`altname`) — they all point to the same interface.

---

## Inspecting Network Configuration — `ip` Command

The `ip` command is the modern, recommended tool. It has replaced the older `ifconfig`, `route`, and `arp` commands.

### Showing Interfaces

```bash
ip link show                       # list all interfaces (link layer)
ip link show ens3                  # one specific interface
ip -br link                        # brief output
ip -s link show ens3               # include statistics (packets, errors, drops)
```

### Showing IP Addresses

```bash
ip addr                            # all addresses on all interfaces
ip addr show ens3                  # one specific interface
ip -br addr                        # brief output — most useful
ip -4 addr                         # IPv4 only
ip -6 addr                         # IPv6 only
```

Sample `ip -br addr` output:
```
lo      UNKNOWN   127.0.0.1/8    ::1/128
ens3    UP        172.25.250.10/24    fe80::5054:ff:fe00:fa0a/64
ens4    UP
```

### Showing Routing Tables

```bash
ip route                           # IPv4 routing table
ip -6 route                        # IPv6 routing table
ip route get 8.8.8.8               # which interface would be used to reach 8.8.8.8?
```

Sample `ip route` output:
```
default via 192.0.2.254 dev ens3 proto static metric 100
192.0.2.0/24 dev ens3 proto kernel scope link src 192.0.2.2
10.0.0.0/8 dev ens4 proto kernel scope link src 10.0.0.11
```

Read as: *"Anything not in `192.0.2.0/24` or `10.0.0.0/8` goes to the gateway at `192.0.2.254` via `ens3`."*

---

## Connectivity Testing

### `ping` — Is the Host Reachable?

```bash
ping 192.168.1.1                   # pings forever (Ctrl+C to stop)
ping -c 4 192.168.1.1              # send exactly 4 packets
ping -i 0.5 192.168.1.1            # interval of 0.5 seconds between packets
ping -s 1500 192.168.1.1           # larger packet size (test MTU issues)
ping google.com                    # also tests DNS resolution
ping6 2001:db8::1                  # IPv6 equivalent
ping -c 3 fe80::1%ens3             # link-local — must specify interface with %
```

> **Windows difference:** Windows `ping` sends 4 packets by default then stops. Linux `ping` runs forever until you stop it with `Ctrl+C` or specify `-c`.

### `traceroute`, `tracepath`, `mtr` — Where Is It Breaking?

```bash
tracepath google.com               # traces the path to google.com (no root needed)
traceroute google.com              # similar, usually needs installing first
traceroute -I google.com           # use ICMP (like Windows tracert)
traceroute -T -p 443 google.com    # use TCP to specific port (gets through firewalls)
mtr google.com                     # interactive — combines ping + traceroute
mtr -r -c 10 google.com            # report mode — 10 packets, then exit
```

> **`mtr` is the power user's tool** — it runs continuously showing packet loss and latency at every hop. Excellent for diagnosing intermittent issues.

---

## Port and Socket Inspection — `ss`

The `ss` command (socket statistics) has replaced the older `netstat`. They do similar things.

### Common `ss` Commands

```bash
ss                                 # all established connections
ss -t                              # TCP connections only
ss -u                              # UDP connections only
ss -l                              # listening sockets only
ss -a                              # all (listening + established)
ss -n                              # show port numbers, not service names
ss -p                              # include process/PID info (run as root)

# Most useful combinations
ss -tulpn                          # TCP+UDP listening, with PIDs, numeric ports
ss -tan                            # all TCP sockets, numeric
ss -lt                             # listening TCP only
```

### Sample Output (What It Means)

```
State    Local Address:Port    Peer Address:Port
LISTEN   127.0.0.1:25          0.0.0.0:*              ← SMTP, loopback only
LISTEN   0.0.0.0:22            0.0.0.0:*              ← SSH on all IPv4
LISTEN   [::]:22               [::]:*                  ← SSH on all IPv6
ESTAB    172.25.250.10:22      172.25.250.9:57560     ← active SSH connection
LISTEN   *:9090                *:*                     ← web console on all
```

- `0.0.0.0` or `*` = listening on all IPv4 interfaces
- `127.0.0.1` = loopback only (not accessible from the network)
- `[::]` = listening on all IPv6 interfaces
- `ESTAB` = an active connection
- `LISTEN` = a service waiting for connections

---

## Well-Known Ports

The `/etc/services` file maps service names to port numbers. Common ports every admin should know:

| Port | Protocol | Service |
|---|---|---|
| 20, 21 | TCP | FTP |
| 22 | TCP | SSH, SFTP |
| 23 | TCP | Telnet (legacy, insecure) |
| 25 | TCP | SMTP (email) |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP (server, client) |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 (email retrieval) |
| 123 | UDP | NTP (time sync) |
| 143 | TCP | IMAP (email retrieval) |
| 389 | TCP | LDAP |
| 443 | TCP | HTTPS |
| 465 | TCP | SMTPS |
| 514 | UDP | Syslog |
| 636 | TCP | LDAPS |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 3389 | TCP | RDP |
| 5432 | TCP | PostgreSQL |
| 3306 | TCP | MySQL / MariaDB |
| 9090 | TCP | Cockpit (RHEL Web Console) |

```bash
grep -w 443 /etc/services          # look up what service uses port 443
```

---

## DNS Basics

DNS translates hostnames to IP addresses. Settings are controlled by `/etc/resolv.conf`:

```bash
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 1.1.1.1
# search example.com
```

Name resolution order (controlled by `/etc/nsswitch.conf`):

1. `/etc/hosts` — local static entries (checked first)
2. DNS servers from `/etc/resolv.conf`

```bash
# Query DNS directly
dig google.com                     # detailed DNS query
dig +short google.com              # just the answer
dig google.com MX                  # mail exchange records
dig @8.8.8.8 google.com            # query a specific DNS server

host google.com                    # simpler DNS lookup
nslookup google.com                # another option
```

> The full name resolution and hostnames topic is covered in Chapter 18. This section is the bare minimum to understand what is happening.

---

## Windows Comparison

| Windows | Linux | Notes |
|---|---|---|
| `ipconfig /all` | `ip addr` | Show all IP addresses |
| `ipconfig /release` / `/renew` | `nmcli connection down/up` (Ch 18) | DHCP release/renew |
| `route print` | `ip route` | Show routing table |
| `ping hostname` | `ping hostname` | Same (Linux pings forever) |
| `ping -n 4 host` | `ping -c 4 host` | Count packets |
| `tracert host` | `traceroute host` / `tracepath host` | Trace route |
| `pathping host` | `mtr host` | Combined ping + trace |
| `netstat -an` | `ss -an` | Show all connections |
| `netstat -ano` | `ss -anp` | Include PID info |
| `nslookup host` | `dig host` / `nslookup host` | DNS query |
| `getmac` | `ip link` | Show MAC addresses |
| Network adapter name (e.g. "Ethernet 1") | `ens3`, `eno1`, `enp0s3` | Interface name |
| Default gateway | Default route (`ip route`) | Gateway concept |
| `hosts` file in `C:\Windows\System32\drivers\etc\` | `/etc/hosts` | Local name overrides |

---

## Things to Remember

- **RFC 1918 private ranges** — `10/8`, `172.16/12`, `192.168/16` — not routable on the public internet
- **`127.0.0.1` and `::1`** — loopback, the machine talking to itself, always reachable
- **`/24`, `/16`, `/8`** — the three "classful" subnet sizes you will see most often
- **Network and broadcast addresses are not usable** — a `/24` subnet has 256 addresses but only 254 usable hosts
- **IPv4 and IPv6 run side by side** on RHEL by default (dual-stack) — every interface has both
- **IPv6 `::` can only appear once** in an address — otherwise it is ambiguous
- **Use `ipcalc`** for subnet calculations — do not do binary math in your head
- **`ss` replaced `netstat`** — learn `ss`, but `netstat` still works if installed
- **`ip` replaced `ifconfig`, `route`, `arp`** — all three old commands are deprecated but often still present
- **`0.0.0.0` in listening output** means "all IPv4 interfaces" — the service is reachable from the network
- **`127.0.0.1` in listening output** means loopback only — the service is NOT reachable from the network, a common security configuration
- **`mtr` is better than `traceroute`** for diagnosing intermittent issues — shows packet loss at every hop continuously
