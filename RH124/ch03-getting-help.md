# Chapter 3 – Getting Help from the Command Line
## RH124 Student Quick Reference

---

## Ways to Get Help in Linux

Unlike Windows where you Google for answers, Linux ships with comprehensive built-in documentation. You can get help without an internet connection.

| Method | Command | Best For |
|---|---|---|
| Manual pages | `man command` | Full reference documentation |
| Built-in help | `command --help` | Quick option summary |
| Keyword search | `man -k keyword` | Finding commands you do not know yet |
| Info pages | `info command` | Detailed GNU tool documentation |
| Local docs | `/usr/share/doc/` | READMEs, examples, changelogs |

---

## The `man` Command

```bash
man ls                  # open the manual page for ls
man cp                  # manual page for cp
man 5 passwd            # section 5 (file format) of passwd
man -k search           # search all man pages for the word "search"
man -f passwd           # show which sections have a page for "passwd"
```

---

## Man Page Sections

Man pages are divided into numbered sections. The same topic name can appear in multiple sections with different meanings:

| Section | Content |
|---|---|
| **1** | User commands — `ls`, `cp`, `grep`, etc. |
| **2** | System calls — kernel functions used by programs |
| **3** | Library functions — C programming functions |
| **4** | Special files — device files in `/dev` |
| **5** | File formats — config file syntax (e.g. `/etc/passwd` format) |
| **6** | Games |
| **7** | Conventions and standards |
| **8** | System administration commands — `fdisk`, `sshd`, `mount` |

**Why this matters:**
```bash
man passwd          # opens passwd(1) — the command to change passwords
man 5 passwd        # opens passwd(5) — the /etc/passwd file FORMAT
```

> When you see documentation refer to `passwd(1)` or `crontab(5)`, the number in brackets is the section.

---

## Navigating Inside a Man Page

Man pages open in a pager called `less`. The same navigation keys apply everywhere you use `less`.

| Key | Action |
|---|---|
| `Space` or `PageDown` | Scroll forward one screen |
| `PageUp` or `b` | Scroll backward one screen |
| `↓` | Scroll one line down |
| `↑` | Scroll one line up |
| `/word` | Search forward for *word* |
| `n` | Next search result |
| `Shift+N` | Previous search result |
| `g` | Go to the beginning |
| `G` | Go to the end |
| `q` | **Quit** |

---

## Man Page Structure

Most man pages follow the same layout:

| Section | What You Will Find |
|---|---|
| **NAME** | Command name and one-line description |
| **SYNOPSIS** | Command syntax — how to use it |
| **DESCRIPTION** | Full explanation of what it does |
| **OPTIONS** | Every available flag and option |
| **EXAMPLES** | Real usage examples (very useful) |
| **FILES** | Related files and directories |
| **SEE ALSO** | Related commands and man pages |
| **BUGS** | Known issues |

> **Tip:** Jump straight to EXAMPLES with `/EXAMPLES` then press Enter.

---

## Searching for Commands by Keyword

Do not know which command to use? Search by keyword:

```bash
man -k copy             # find all man pages related to "copy"
man -k "disk space"     # multi-word search (use quotes)
apropos copy            # same as man -k copy
```

Example output:
```
cp (1)        - copy files and directories
dd (1)        - convert and copy a file
rsync (1)     - fast, versatile, remote file-copying tool
```

---

## Quick Help

For a fast reminder of options without opening a full man page:

```bash
ls --help
grep --help
tar --help
```

This shows a short summary — good when you just need to check a flag quickly.

---

## Other Documentation Sources

```bash
info ls                         # GNU info pages — more detailed for some tools
                                # navigate with arrow keys, q to quit

ls /usr/share/doc/              # documentation installed with packages
ls /usr/share/doc/bash/         # docs for bash specifically
cat /usr/share/doc/bash/README  # read a doc file
```

---

## Things to Remember

- `man` pages are installed locally — they work offline, on any server, without Google
- The section number matters when a topic exists in multiple sections
- `man -k` is invaluable when you forget a command name — describe what you want
- `less` is the pager used by man — learn its keys once, use them everywhere
- `/usr/share/doc/` contains example config files — copy and adapt rather than write from scratch
- `apropos` and `whatis` are just shorter aliases for `man -k` and `man -f`
