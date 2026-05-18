# RH134 Chapter 2 - Using Regular Expressions for Practical Applications

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Efficiently complete system administration tasks by using regular expressions to match text.

---

## Windows vs Linux: Pattern Matching Equivalents

| Windows Concept | Linux Equivalent |
|---|---|
| `findstr` in CMD | `grep` |
| `Select-String` in PowerShell | `grep` (or `grep -P` for Perl regex) |
| Wildcard `*` in Explorer/CMD (matches anything) | Shell glob `*` (filename matching only) |
| Regex in PowerShell (`-match`, `-replace`) | Regex in `grep`, `sed`, `awk`, `vim` |
| `findstr /I` (case insensitive) | `grep -i` |
| `findstr /V` (invert match) | `grep -v` |
| `findstr /N` (line numbers) | `grep -n` |
| `findstr /R` (recursive) | `grep -r` |

> **Important distinction:** The `*` wildcard in Windows Explorer/CMD and the `*` in regular expressions are NOT the same thing. Shell glob `*` matches any filename. Regex `*` means "zero or more of the preceding character." `grep 'a*' file` matches everything, because zero a's is still valid.

---

## 1. What Are Regular Expressions?

Regular expressions (regex) are patterns used to match text. They are supported by many tools and languages including `grep`, `sed`, `vim`, `less`, `awk`, Python, Rust, and C.

There are two types used in Linux:

| Type | Used by | Notes |
|---|---|---|
| Basic Regular Expressions (BRE) | `grep`, `sed`, `vim` | Default mode |
| Extended Regular Expressions (ERE) | `grep -E`, `egrep`, `less` | More features, cleaner syntax |

> The main practical difference: in BRE, special characters like `+`, `?`, `|`, `(`, `)`, `{`, `}` must be escaped with `\` to have special meaning. In ERE, they have special meaning by default.

---

## 2. Regex Building Blocks

### Anchors - Control Where on the Line to Match

| Pattern | Matches | Example |
|---|---|---|
| `^` | Start of a line | `^root` matches lines that begin with `root` |
| `$` | End of a line | `bash$` matches lines that end with `bash` |
| `^word$` | Exact line match only | `^root$` matches only a line containing exactly `root` |
| `\b` | Word boundary | `\bcat\b` matches `cat` but not `concatenate` |
| `\<` | Start of a word | `\<cat` matches `cat` and `category` |
| `\>` | End of a word | `cat\>` matches `cat` and `tomcat` |

```bash
# Lines starting with root
grep '^root' /etc/passwd

# Lines ending with /bin/bash
grep '/bin/bash$' /etc/passwd

# Lines that are ONLY a number (nothing else)
grep '^[0-9]*$' file.txt

# Strip blank lines and comment lines from a config file
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
```

### Wildcards - Match Any Character

| Pattern | Matches |
|---|---|
| `.` | Any single character (except newline) |
| `c.t` | `cat`, `cut`, `cot`, `c3t`, `c t` |
| `c..t` | Any four characters starting with `c` and ending with `t` |

### Character Classes - Match Specific Sets

| Pattern | Matches |
|---|---|
| `[abc]` | Any one of: `a`, `b`, or `c` |
| `[a-z]` | Any single lowercase letter |
| `[A-Z]` | Any single uppercase letter |
| `[0-9]` | Any single digit |
| `[^abc]` | Any character EXCEPT `a`, `b`, or `c` (negation) |
| `[aeiou]` | Any vowel |

```bash
# Match cat, cot, or cut
grep 'c[aou]t' file.txt

# Match any line with a digit
grep '[0-9]' file.txt

# Match lines NOT starting with a lowercase letter
grep '^[^a-z]' file.txt
```

### POSIX Character Classes - Locale-Safe Alternatives

Use these instead of `[a-z]` for reliable results across different locales and system languages.

| Class | Equivalent | Matches |
|---|---|---|
| `[[:alpha:]]` | `[a-zA-Z]` | Any alphabetic character |
| `[[:digit:]]` | `[0-9]` | Any digit |
| `[[:alnum:]]` | `[a-zA-Z0-9]` | Any alphanumeric character |
| `[[:lower:]]` | `[a-z]` | Any lowercase letter |
| `[[:upper:]]` | `[A-Z]` | Any uppercase letter |
| `[[:space:]]` | tab, space, newline etc. | Any whitespace character |
| `[[:blank:]]` | space and tab only | Space or tab |
| `[[:punct:]]` | `!`, `"`, `#`, `$` ... | Punctuation characters |
| `[[:print:]]` | All printable characters | Alphanumeric + punct + space |
| `[[:xdigit:]]` | `[0-9A-Fa-f]` | Hexadecimal digits |

```bash
# Match lines containing only digits
grep '^[[:digit:]]*$' file.txt

# Match lines starting with an uppercase letter
grep '^[[:upper:]]' file.txt
```

### Shorthand Character Classes (ERE / `grep -E`)

| Pattern | Equivalent | Matches |
|---|---|---|
| `\w` | `[_[:alnum:]]` | Word characters (letters, digits, underscore) |
| `\W` | `[^_[:alnum:]]` | Non-word characters |
| `\s` | `[[:space:]]` | Whitespace |
| `\S` | `[^[:space:]]` | Non-whitespace |

---

## 3. Multipliers (Quantifiers)

Multipliers apply to the character or group immediately before them.

| BRE Syntax | ERE Syntax | Meaning |
|---|---|---|
| `*` | `*` | Zero or more of the preceding item |
| `\+` | `+` | One or more of the preceding item |
| `\?` | `?` | Zero or one of the preceding item (optional) |
| `\{n\}` | `{n}` | Exactly n occurrences |
| `\{n,\}` | `{n,}` | n or more occurrences |
| `\{,m\}` | `{,m}` | At most m occurrences |
| `\{n,m\}` | `{n,m}` | Between n and m occurrences (inclusive) |

```bash
# Zero or more digits (matches even empty strings)
grep '[0-9]*' file.txt

# One or more digits (BRE)
grep '[0-9]\+' file.txt

# One or more digits (ERE)
grep -E '[0-9]+' file.txt

# Exactly 3 digits
grep '[0-9]\{3\}' file.txt

# Exactly 3 digits (ERE)
grep -E '[0-9]{3}' file.txt

# 3 to 5 lowercase letters
grep -E '[a-z]{3,5}' file.txt

# Match a word of exactly 4 characters
grep -E '^\w{4}$' file.txt
```

---

## 4. The `grep` Command

### Basic Syntax

```bash
grep 'PATTERN' FILE
grep 'PATTERN' FILE1 FILE2 FILE3
command | grep 'PATTERN'
```

> **Always use single quotes** around the pattern. Double quotes allow the shell to expand `$`, `*`, `{}`, etc. before `grep` sees the pattern, which causes unexpected behaviour.

### Common Options

| Option | Long form | Function |
|---|---|---|
| `-i` | `--ignore-case` | Case-insensitive matching |
| `-v` | `--invert-match` | Show lines that do NOT match |
| `-n` | `--line-number` | Show line numbers with output |
| `-c` | `--count` | Show count of matching lines only |
| `-l` | `--files-with-matches` | Show only filenames that contain a match |
| `-r` | `--recursive` | Search recursively through directories |
| `-E` | `--extended-regexp` | Use extended regex (ERE) syntax |
| `-e` | `--regexp` | Specify multiple patterns (logical OR) |
| `-A N` | `--after-context=N` | Show N lines after each match |
| `-B N` | `--before-context=N` | Show N lines before each match |
| `-C N` | `--context=N` | Show N lines before AND after each match |
| `-w` | `--word-regexp` | Match whole words only |
| `-o` | `--only-matching` | Print only the matched part, not the whole line |

### `grep` Examples

```bash
# Case-insensitive search
grep -i 'serverroot' /etc/httpd/conf/httpd.conf

# Find lines NOT containing a pattern
grep -v 'nologin' /etc/passwd

# Show line numbers
grep -n 'PermitRootLogin' /etc/ssh/sshd_config

# Count occurrences (useful in scripts)
grep -c 'Failed password' /var/log/secure

# Search recursively through a directory
grep -r 'Include' /etc/httpd/

# Context lines - see what surrounds a match
systemctl status httpd | grep 'Active' -B 2 -A 3

# Multiple patterns (OR logic)
grep -e '^ServerRoot' -e '^DocumentRoot' /etc/httpd/conf/httpd.conf

# Whole word match only
grep -w 'cat' file.txt     # matches 'cat' but not 'concatenate'

# Use extended regex
grep -E '[0-9]{1,3}\.[0-9]{1,3}' /etc/hosts

# Show only the matched part
grep -o '[0-9]\+' file.txt
```

### Stripping Noise from Config Files (Real-World Pattern)

```bash
# Remove comment lines (starting with #) and blank lines
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'

# Same, combined into one grep using extended regex
grep -Ev '^(#|$)' /etc/ssh/sshd_config

# Works on any config file:
grep -Ev '^(#|;|$)' /etc/systemd/system/rsyslog.service
```

---

## 5. BRE vs ERE: Side-by-Side

| Task | BRE (`grep`) | ERE (`grep -E`) |
|---|---|---|
| One or more digits | `[0-9]\+` | `[0-9]+` |
| Optional character | `colou\?r` | `colou?r` |
| Alternation (OR) | `cat\|dog` | `cat\|dog` |
| Grouping | `\(abc\)\+` | `(abc)+` |
| Exact count | `[0-9]\{3\}` | `[0-9]{3}` |
| Range count | `[0-9]\{2,4\}` | `[0-9]{2,4}` |

> **Tip:** When in doubt, use `grep -E`. ERE syntax is cleaner, matches what most programming languages use, and avoids the confusing backslash escaping of BRE.

---

## 6. Using Regex in `vim` and `less`

### Searching in `vim`

```
/pattern        Search forward for pattern
?pattern        Search backward for pattern
n               Jump to next match
N               Jump to previous match
/^ServerAdmin   Find lines starting with ServerAdmin
/[Nn][Tt][Pp]   Find NTP, ntp, Ntp, etc. (case-insensitive via class)
```

### Searching in `less`

```
/pattern        Search forward
?pattern        Search backward
n               Next match
N               Previous match
```

> `less` uses extended regex by default, so you can use `+`, `?`, `|`, `()` without backslashes.

---

## 7. Practical Real-World Examples

```bash
# Find all users with a login shell (not nologin or false)
grep -v 'nologin\|false' /etc/passwd

# Check for failed SSH logins
grep 'Failed password' /var/log/secure

# Find all accounts with UID 0 (should only be root - security check)
grep -E '^[^:]+:[^:]+:0:' /etc/passwd

# Find lines with valid IPv4-like patterns
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' /etc/hosts

# Count failed sudo attempts
grep -c 'authentication failure' /var/log/secure

# Find all enabled systemd services
systemctl list-unit-files | grep 'enabled'

# Search multiple log files at once
grep 'error' /var/log/messages /var/log/secure /var/log/boot.log

# Find NTP-related config in chrony
grep -i '[Nn][Tt][Pp]\|chrony' /etc/chrony.conf

# The self-match trick - avoid grep showing itself in ps output
ps aux | grep '[h]ttpd'    # bracket around first char prevents self-match
```

---

## 8. Regex Quick-Build Guide

Build a regex left to right, one piece at a time:

| What you want to match | Pattern |
|---|---|
| Lines starting with a digit | `^[[:digit:]]` |
| Lines ending with `.conf` | `\.conf$` |
| Lines that are only digits | `^[[:digit:]]\+$` |
| Lines with at least one uppercase letter | `[[:upper:]]` |
| Lines starting with a comment | `^#` |
| Lines that are blank | `^$` |
| Lines that are NOT blank and NOT comments | `grep -Ev '^(#\|$)'` |
| A word of exactly 8 characters | `\b[[:alpha:]]\{8\}\b` |
| An IP address pattern | `[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}` |

> **Note:** To match a literal `.` (dot), escape it: `\.`. An unescaped `.` matches any character.

---

## Things to Remember

1. **Regex `*` is NOT the same as shell glob `*`.** In regex, `*` means "zero or more of the preceding character." In shell globbing, `*` matches any filename string. Do not mix them up.

2. **Always single-quote your regex patterns** with `grep`. Without quotes, the shell may expand `$`, `*`, `{`, and `!` before `grep` ever sees the pattern.

3. **`^` inside brackets means negation, not anchor.** `[^abc]` means "not a, b, or c." Outside brackets, `^` means "start of line."

4. **Use POSIX classes instead of `[a-z]` on shared systems.** `[[:lower:]]` is locale-safe. `[a-z]` may behave differently depending on the system locale setting.

5. **`-v` (invert) is one of the most useful flags.** `grep -v '^#' file | grep -v '^$'` strips comments and blank lines from any config file, leaving only the active directives.

6. **`-e` is OR, not AND.** To require two patterns on the same line, pipe: `grep 'pattern1' file | grep 'pattern2'`.

7. **Use `grep -E` for cleaner syntax.** Extended regex lets you write `+`, `?`, `{3}`, `(group)` without backslashes. Easier to read, easier to write, and matches what most programming languages use.

8. **`grep -c` counts matching lines.** Use it instead of `grep | wc -l` in scripts - it is cleaner and one less process.

9. **`grep` on `ps` output will match itself.** Use the bracket trick: `grep '[h]ttpd'` instead of `grep 'httpd'` to prevent the grep process from appearing in its own output.

10. **A `.` in regex matches any character, including a literal dot.** To match an actual period (e.g. in an IP address or a filename), escape it: `\.`.
