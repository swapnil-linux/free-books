# Chapter 1 – Introduction to Linux
## RH124 Student Quick Reference

---

## What Is Linux?

Linux is an **open source operating system kernel** created by Linus Torvalds in 1991.
It is the foundation of hundreds of operating systems used worldwide.

**Open source means:**
- The source code is publicly available — anyone can read it, modify it, and share it
- It is not the same as "free of charge" — support, certifications, and enterprise products cost money
- Development is transparent — you can see exactly how the OS works

---

## Where Is Linux Used?

You almost certainly interact with Linux every day without knowing it:

| Where | Example |
|---|---|
| Web servers | Most websites on the internet run on Linux |
| Cloud platforms | AWS, Azure, and Google Cloud all run Linux underneath |
| Android phones | Android is built on the Linux kernel |
| Smart TVs and streaming | Netflix, smart appliances, in-flight entertainment |
| Supercomputers | The majority of the world's top 500 supercomputers run Linux |
| Stock exchanges | Most financial trading platforms run on Linux |
| Containers | Docker, Kubernetes, and all container platforms are Linux-native |

---

## Linux vs Windows — The Key Differences

| Aspect | Windows | Linux |
|---|---|---|
| Source code | Closed — Microsoft only | Open — anyone can read it |
| Cost | Licence fee per device | Usually free (support costs money) |
| Configuration | Mostly GUI, Registry | Mostly text files, command line |
| Updates | Large, infrequent | Smaller, more frequent |
| Package management | .exe / .msi installers | Package manager (dnf, apt, etc.) |
| File system | NTFS, case-insensitive | ext4/XFS, case-sensitive |
| Multiple versions | One Windows 11 | Many distributions (distros) |

---

## What Is a Linux Distribution?

A **distribution (distro)** is a complete operating system built from the Linux kernel plus:
- System tools and utilities
- A package manager
- Default applications
- Support model

Think of it like this: the Linux kernel is the engine, and a distribution is the complete car.

**Common distributions:**

| Distribution | Audience | Notes |
|---|---|---|
| **Red Hat Enterprise Linux (RHEL)** | Enterprise, commercial | Paid support, long lifecycle |
| **Ubuntu** | General, cloud, desktop | Very popular, large community |
| **Debian** | Servers, developers | Stable, conservative release cycle |
| **Fedora** | Developers, enthusiasts | Cutting edge, upstream for RHEL |
| **CentOS Stream** | Developers | Free, upstream preview of RHEL |
| **SUSE Linux Enterprise** | Enterprise | Common in SAP environments |
| **Kali Linux** | Security professionals | Pre-loaded security tools |
| **Amazon Linux** | AWS cloud | Optimised for AWS |

> **Skills transfer:** Command-line skills you learn on RHEL work on Ubuntu, Debian, and most other distros. The core tools (`ls`, `cp`, `mv`, `grep`, etc.) are the same everywhere.

---

## The Red Hat Ecosystem (for This Course)

```
Fedora → CentOS Stream → RHEL
(cutting edge)  (development preview)  (stable / supported)
```

- **Fedora** — community distro, newest features, rapid updates
- **CentOS Stream** — continuous preview of the next RHEL minor version
- **RHEL** — enterprise-ready, commercially supported, long lifecycle (10 years per major version)

---

## Ways to Get RHEL

| Method | Cost | Best For |
|---|---|---|
| Paid subscription | Paid | Production systems |
| **Red Hat Developer Subscription** | **Free** | Learning, dev, testing (up to 16 systems) |
| RHEL Evaluation | Free (time-limited) | Short-term evaluation |
| CentOS Stream | Free | Development / community use |

> Sign up at [developer.redhat.com](https://developer.redhat.com) for a free Developer Subscription.

---

## Key Portals

| URL | Purpose |
|---|---|
| `console.redhat.com` | System management, subscriptions, Red Hat Insights |
| `access.redhat.com` | Documentation, knowledge base, downloads |
| `developer.redhat.com` | Free developer subscriptions and resources |

---

## Things to Remember

- **Open source ≠ free** — it means the code is open and auditable
- Linux skills are highly transferable — the core is the same across distros
- The command line is the primary tool for Linux administration — the GUI is optional
- Linux is not just a server OS — it underpins almost everything in modern IT
- Case sensitivity matters in Linux: `File.txt` and `file.txt` are two different files
