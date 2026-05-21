# RH134 Chapter 14 - Managing Network Security

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Control network connections to services by using the system firewall, and control which ports services can bind to by using SELinux port labelling.

---

## Windows vs Linux: Network Security Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| Windows Firewall / Windows Defender Firewall | `firewalld` service |
| `netsh advfirewall` | `firewall-cmd` |
| Windows Firewall with Advanced Security (GUI) | `firewall-cmd` or cockpit web console |
| Inbound firewall rules | `firewall-cmd --add-service` / `--add-port` |
| Network profiles (Domain / Private / Public) | firewalld zones (internal / work / public) |
| `iptables` (older Linux) | `nftables` (kernel) managed via `firewalld` |
| Windows Filtering Platform (WFP) | `netfilter` kernel framework |
| AppLocker port restrictions | SELinux port labelling (`semanage port`) |

---

## The Two Security Layers for Network Services

To allow external access to a service on a non-standard port, BOTH of the following must be correct:

```
External Client
      |
      v
┌─────────────────────────────────┐
│  FIREWALLD (zone rules)         │  <- Layer 1: Does the zone allow this port/service?
│  firewall-cmd --add-port        │
└─────────────────┬───────────────┘
                  | (if allowed)
                  v
┌─────────────────────────────────┐
│  SELinux PORT LABELLING         │  <- Layer 2: Can this process bind to this port?
│  semanage port -a -t TYPE       │
└─────────────────┬───────────────┘
                  | (if allowed)
                  v
            Service Process
           (httpd, sshd, etc.)
```

> **Both gates must be open.** Opening only the firewall but not the SELinux port label leaves the service unable to bind. Opening only the SELinux port label but not the firewall leaves external clients unable to reach the service.

---

## Part 1: Managing Firewalls with firewalld

### Architecture

```
Incoming packet
    |
    v
Does source IP match a zone's source rule?
    YES -> apply that zone's rules
    NO  -> Does incoming interface have a zone assignment?
               YES -> apply that zone's rules
               NO  -> apply the DEFAULT zone's rules (usually 'public')
```

### Predefined Zones (ordered from most to least permissive)

| Zone | Default Behaviour |
|---|---|
| `trusted` | Allow ALL incoming traffic |
| `home` | Allow ssh, mdns, ipp-client, samba-client, dhcpv6-client |
| `internal` | Same as home (intended for internal network interfaces) |
| `work` | Allow ssh, ipp-client, dhcpv6-client |
| `public` | Allow ssh, dhcpv6-client only. **Default zone** |
| `external` | Allow ssh only. Outgoing IPv4 masqueraded (NAT) |
| `dmz` | Allow ssh only. For DMZ servers |
| `block` | Reject all incoming (sends ICMP rejection - sender knows it was blocked) |
| `drop` | Drop all incoming silently (no response - sender cannot tell host exists) |

> **Default zone:** `public` is the default zone for all new network interfaces. The `lo` loopback interface is permanently in `trusted`.

> **`drop` vs `block`:** Use `drop` on internet-facing interfaces to avoid revealing the host's existence to port scanners. Use `block` on internal interfaces where applications need prompt failure notification rather than timeouts.

### Runtime vs Permanent Configuration

| Configuration | Effect | Survives reload/reboot? |
|---|---|---|
| Runtime (no `--permanent`) | Active immediately | No |
| Permanent (`--permanent`) | Written to disk, not yet active | Yes |

```bash
# Runtime only (testing):
firewall-cmd --add-service=http

# Permanent only (not yet active - must reload):
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Both at once (active now AND persistent):
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Promote all current runtime rules to permanent:
firewall-cmd --runtime-to-permanent
```

> **Professional workflow:** Use `--permanent` + `--reload` for production changes. Test changes without `--permanent` first, then use `--runtime-to-permanent` to commit when confirmed working.

### Key `firewall-cmd` Commands

#### Inspecting the Firewall

```bash
# Show the default zone
firewall-cmd --get-default-zone

# List all available zones
firewall-cmd --get-zones

# List all active zones and their interface/source assignments
firewall-cmd --get-active-zones

# Show all configuration for the default zone
firewall-cmd --list-all

# Show all configuration for a specific zone
firewall-cmd --list-all --zone=internal

# Show all information for all zones
firewall-cmd --list-all-zones

# List all available predefined service names
firewall-cmd --get-services

# List all available predefined service names (one per line)
firewall-cmd --get-services | tr ' ' '\n'
```

#### Changing the Default Zone

```bash
# Change default zone (affects both runtime AND permanent)
firewall-cmd --set-default-zone=dmz
firewall-cmd --set-default-zone=public
```

#### Managing Services

```bash
# Add a service to the default zone (runtime only)
firewall-cmd --add-service=http
firewall-cmd --add-service=https
firewall-cmd --add-service=mysql

# Add a service permanently (reload required)
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Add a service to a specific zone permanently
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --reload

# Remove a service
firewall-cmd --permanent --remove-service=http
firewall-cmd --reload
```

#### Managing Ports

```bash
# Allow a specific port/protocol (runtime)
firewall-cmd --add-port=8080/tcp
firewall-cmd --add-port=514/udp

# Allow a port permanently
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload

# Allow a port range
firewall-cmd --permanent --add-port=8000-8100/tcp
firewall-cmd --reload

# Remove a port
firewall-cmd --permanent --remove-port=82/tcp
firewall-cmd --reload
```

#### Managing Zone Sources (Source IP-based Assignment)

```bash
# Assign all traffic from a network to a zone (permanent)
firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24
firewall-cmd --reload

# Assign traffic from a single host to a zone
firewall-cmd --permanent --zone=trusted --add-source=172.25.25.11/32
firewall-cmd --reload

# Remove a source assignment
firewall-cmd --permanent --zone=internal --remove-source=192.168.0.0/24
firewall-cmd --reload
```

#### Managing Zone Interface Assignments

```bash
# Assign an interface to a zone
firewall-cmd --permanent --zone=dmz --add-interface=eth1
firewall-cmd --reload

# Move an interface to a different zone
firewall-cmd --permanent --zone=internal --change-interface=eth0
firewall-cmd --reload
```

### Common Scenarios

```bash
# Allow Apache web server (HTTP and HTTPS)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Allow a non-standard web port
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --reload

# Allow MySQL from internal network only
firewall-cmd --permanent --zone=internal --add-source=10.0.0.0/8
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --reload

# Verify the configuration looks correct
firewall-cmd --list-all
firewall-cmd --list-all --zone=internal

# Check the firewalld service itself
systemctl status firewalld
systemctl enable --now firewalld
```

### `firewall-cmd --list-all` Output Explained

```
public (default, active)           <- zone name, whether default, whether in use
  target: default                  <- what happens to unmatched traffic (default = reject)
  ingress-priority: 0
  egress-priority: 0
  interfaces: ens3                 <- interfaces assigned to this zone
  sources:                         <- source IPs/networks assigned to this zone
  services: cockpit dhcpv6-client https ssh  <- allowed services
  ports: 82/tcp                    <- allowed ports not covered by a service
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

---

## Part 2: SELinux Port Labelling

### Why SELinux Port Labelling Exists

SELinux labels network ports just like it labels files. When a service process tries to bind to a port (start listening), SELinux checks if that process type is allowed to bind to ports with that label. This prevents:
- A compromised service from hijacking well-known ports
- Rogue processes from binding to reserved service ports
- Services being accidentally configured to listen on wrong ports

```
httpd_t tries to bind to port 82/tcp
    -> SELinux checks: is 82/tcp labelled with a type that httpd_t can bind to?
    -> 82/tcp has label reserved_port_t (default for unlabelled ports)
    -> httpd_t CANNOT bind to reserved_port_t
    -> SELinux DENIES the bind -> service fails to start
```

### Listing Port Labels

```bash
# List all SELinux port labels
semanage port -l

# Filter by service type label
semanage port -l | grep http
semanage port -l | grep ssh
semanage port -l | grep ftp
semanage port -l | grep mysql

# Filter by port number
semanage port -l | grep -w 80
semanage port -l | grep -w 82

# Show ONLY locally customised port labels (your changes only)
semanage port -l -C
```

### Common Port Type Labels

| Label | Default Ports | Used by |
|---|---|---|
| `http_port_t` | 80, 81, 443, 488, 8008, 8009, 8443, 9000 | Apache, nginx |
| `http_cache_port_t` | 8080, 8118, 8123, 10001-10010 | Squid, web caches |
| `ssh_port_t` | 22 | SSH daemon |
| `ftp_port_t` | 21, 989, 990 | FTP server |
| `mysqld_port_t` | 3306, 1186 | MariaDB/MySQL |
| `postgresql_port_t` | 5432 | PostgreSQL |
| `smtp_port_t` | 25, 465, 587 | Mail servers |
| `dns_port_t` | 53 | DNS servers |

### Managing Port Labels

```bash
# Add a new port label (allow service to bind to this port)
semanage port -a -t PORT_TYPE -p tcp|udp PORTNUMBER

# Examples:
semanage port -a -t http_port_t -p tcp 82
semanage port -a -t http_port_t -p tcp 8888
semanage port -a -t ssh_port_t -p tcp 2222

# Verify the addition
semanage port -l | grep http_port_t
semanage port -l | grep -w 82

# Modify an existing label (change a port's type)
semanage port -m -t http_port_t -p tcp 8080

# Delete a custom label
semanage port -d -t http_port_t -p tcp 82
```

> **You cannot change default port labels** (those built into the policy module). You can only add, modify, or delete custom labels. To allow a service on a non-standard port, add a new label - do not try to remove the existing one.

### The Complete Non-Standard Port Workflow

When a service needs to use a non-standard port (e.g. Apache on port 82):

```bash
# Step 1: Configure the service to use the new port
# (e.g. add "Listen 82" to Apache config)
vim /etc/httpd/conf/httpd.conf

# Step 2: Find the correct SELinux port type for this service
semanage port -l | grep http
# http_port_t  tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000

# Step 3: Add the new port with the correct type label
semanage port -a -t http_port_t -p tcp 82

# Step 4: Verify the label was added
semanage port -l | grep http_port_t
# http_port_t  tcp  82, 80, 81, 443, ...

# Step 5: Start or restart the service
systemctl restart httpd

# Step 6: Open the port in firewalld
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload

# Step 7: Test from an external client
curl http://servera.lab.example.com:82

# Step 8: Verify the firewall and SELinux both show the change
firewall-cmd --list-all
semanage port -l -C
```

### Diagnosing SELinux Port Denial

If a service fails to start and you suspect a SELinux port issue:

```bash
# Check the audit log for AVC denials
ausearch -m AVC -ts recent
# Look for: avc: denied { name_bind } for ... scontext=...httpd_t...

# Use sealert for a human-readable explanation
sealert -a /var/log/audit/audit.log
# Shows: "SELinux is preventing httpd from name_bind access on tcp_socket port 82"
# Also suggests: semanage port -a -t http_port_t -p tcp 82

# After adding the port label, restart and confirm service is running
systemctl restart httpd
systemctl is-active httpd
```

---

## Quick Reference: All Commands

```bash
# --- firewalld status ---
systemctl status firewalld
systemctl enable --now firewalld

# --- Inspect ---
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
firewall-cmd --list-all
firewall-cmd --list-all --zone=ZONE
firewall-cmd --list-all-zones
firewall-cmd --get-services

# --- Default zone ---
firewall-cmd --set-default-zone=ZONE

# --- Services (use --permanent for production) ---
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=http --zone=ZONE
firewall-cmd --permanent --remove-service=http

# --- Ports ---
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --permanent --add-port=514/udp
firewall-cmd --permanent --remove-port=82/tcp

# --- Sources ---
firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24
firewall-cmd --permanent --zone=internal --remove-source=192.168.0.0/24

# --- Apply permanent changes ---
firewall-cmd --reload

# --- Promote runtime to permanent ---
firewall-cmd --runtime-to-permanent

# --- SELinux port labels ---
semanage port -l                           # List all port labels
semanage port -l | grep http               # Filter by type name
semanage port -l | grep -w 80             # Filter by port number
semanage port -l -C                        # Show only custom labels
semanage port -a -t http_port_t -p tcp 82  # Add a port label
semanage port -m -t http_port_t -p tcp 82  # Modify a port label
semanage port -d -t http_port_t -p tcp 82  # Delete a port label

# --- Diagnose SELinux port issues ---
ausearch -m AVC -ts recent
sealert -a /var/log/audit/audit.log
```

---

## Key Configuration Files

| Path | Purpose |
|---|---|
| `/etc/firewalld/zones/` | Admin-customised zone configs (override vendor defaults) |
| `/usr/lib/firewalld/zones/` | Vendor-supplied zone configs (do not edit) |
| `/usr/lib/firewalld/services/` | Predefined service definitions |
| `/etc/firewalld/services/` | Custom service definitions |

---

## Things to Remember

1. **Both firewalld AND SELinux port labels must allow traffic to a non-standard port.** A port open in the firewall but not labelled in SELinux leaves the service unable to bind. A labelled port with no firewall rule leaves clients unable to connect. Fix both.

2. **`--permanent` writes to disk but does not activate the change.** Always follow `--permanent` changes with `firewall-cmd --reload`. Alternatively, use `--runtime-to-permanent` after testing a runtime change.

3. **Without `--permanent`, a rule is lost on the next `firewall-cmd --reload` or service restart.** Test without `--permanent`, confirm it works, then make it permanent.

4. **The default zone is `public`, which only allows `ssh` and `dhcpv6-client`.** Any new service must be explicitly added to the appropriate zone. New services that just work without a firewall rule are not in a firewalld-protected environment.

5. **Source IP zone assignment takes priority over interface assignment.** A packet from `192.168.0.5` assigned to `internal` zone via source rule uses `internal` rules, even if the interface is in `public`.

6. **`drop` zone gives no response to blocked traffic; `block` zone sends ICMP rejection.** Use `drop` on internet-facing interfaces to conceal host existence. Use `block` internally where quick failure is preferable to timeout.

7. **`semanage port -l | grep SERVICE` finds the correct port type label.** Always look up the type from existing standard ports before adding a new one. Do not guess type names.

8. **You cannot change or delete default SELinux port labels.** You can only add new labels or modify/delete custom labels you have created. Default labels are baked into the policy module.

9. **`semanage port -l -C` shows only your customisations.** This is the quick audit view of all non-default SELinux port labels on the system - directly useful for compliance documentation.

10. **`sealert -a /var/log/audit/audit.log` diagnoses SELinux port denials.** It identifies which port was blocked and suggests the correct `semanage port` command to fix it. The suggestion is usually correct - verify the port type against `semanage port -l | grep SERVICE` before applying.
