# Chapter 7 – Managing Files from the Command Line
## RH124 Student Quick Reference

---

## Windows vs Linux File Commands

| Action | Windows (cmd) | Windows (PowerShell) | Linux |
|---|---|---|---|
| List files | `dir` | `Get-ChildItem` | `ls -l` |
| Copy file | `copy` | `Copy-Item` | `cp` |
| Move / rename | `move` / `rename` | `Move-Item` | `mv` |
| Delete file | `del` | `Remove-Item` | `rm` |
| Create folder | `mkdir` | `New-Item -Type Directory` | `mkdir` |
| Delete folder | `rmdir /s /q` | `Remove-Item -Recurse` | `rm -rf` |
| Create empty file | `type nul > file` | `New-Item file` | `touch file` |

---

## Creating Files and Directories

```bash
touch filename.txt              # create an empty file
touch file1 file2 file3         # create multiple files at once
mkdir dirname                   # create a directory
mkdir dir1 dir2 dir3            # create multiple directories
mkdir -p projects/2026/reports  # create entire path including all parents
```

> `mkdir -p` is safe to use in scripts — it does not error if the directory already exists.

---

## Copying Files

```bash
cp source.txt copy.txt              # copy a file to a new name
cp source.txt /tmp/                 # copy a file into a directory
cp file1 file2 file3 /tmp/          # copy multiple files to a directory
cp -r sourcedir/ destdir/           # copy a directory and all its contents
cp -p source.txt copy.txt           # copy and preserve permissions + timestamps
cp -a sourcedir/ destdir/           # full archive copy (preserves everything)
cp -v source.txt copy.txt           # verbose — shows what is being copied
```

> ⚠️ If the destination file already exists, `cp` overwrites it silently. No warning.

---

## Moving and Renaming

```bash
mv oldname.txt newname.txt          # rename a file (move = rename in Linux)
mv file.txt /tmp/                   # move a file to another directory
mv file1 file2 /tmp/                # move multiple files
mv dirname/ newdirname/             # rename a directory
mv -v file.txt /tmp/                # verbose — shows what was moved
```

> `mv` within the same disk is instant regardless of file size — it just updates the directory entry, no data is copied.

---

## Removing Files and Directories

```bash
rm file.txt                         # delete a file — no recycle bin, no undo
rm file1 file2 file3                # delete multiple files
rm -i file.txt                      # interactive — asks before deleting each file
rm -r dirname/                      # delete a directory and everything inside it
rm -rf dirname/                     # force delete, no prompts (use with care)
rmdir dirname/                      # delete an EMPTY directory only
```

> ⚠️ **There is no recycle bin at the Linux command line. Deletions are permanent.**
> Always run `ls` or `pwd` first to confirm you are in the right place.

---

## Viewing Directory Trees

```bash
tree                                # tree view of current directory
tree /var/log                       # tree of a specific path
tree -L 2 /etc                      # limit to 2 levels deep
tree -a ~                           # include hidden files
```

> Install with: `sudo dnf install tree` (RHEL) or `sudo apt install tree` (Ubuntu/Debian)

---

## File Information

```bash
ls -l file.txt                      # size, permissions, owner, date
stat file.txt                       # full detail — all 3 timestamps, inode, etc.
file mystery.bin                    # identify what type of file it is
```

---

## Hard Links vs Symbolic Links

Linux supports two types of links — multiple names pointing to the same file.

| Feature | Hard Link | Symbolic Link (Symlink / Soft Link) |
|---|---|---|
| Points to | The data directly (same inode) | Another filename |
| Cross-filesystem? | No — same disk only | Yes |
| Can link directories? | No | Yes |
| If original deleted | Data still accessible | Link breaks (dangles) |
| `ls -l` shows | Normal file appearance | `l` prefix, shows `→ target` |

```bash
ln source.txt hardlink.txt              # create hard link
ln -s /path/to/target symlinkname       # create symbolic link
ln -s /etc /home/student/myetc         # symlink to a directory
ls -li file1 hardlink1                  # compare inodes — same number = hard linked
```

---

## Shell Wildcards (Globbing)

The shell expands wildcard patterns **before** passing them to the command.

### Wildcard Characters

| Pattern | Matches |
|---|---|
| `*` | Any number of any characters (including none) |
| `?` | Exactly one character |
| `[abc]` | One character from the set: a, b, or c |
| `[a-z]` | One character in a range |
| `[!abc]` | One character NOT in the set |
| `[[:alpha:]]` | Any letter |
| `[[:digit:]]` | Any digit 0–9 |
| `[[:upper:]]` | Any uppercase letter |
| `[[:lower:]]` | Any lowercase letter |
| `[[:alnum:]]` | Any letter or digit |

### Wildcard Examples

```bash
ls *.txt                        # all files ending in .txt
ls file?.txt                    # file1.txt, file2.txt, fileA.txt ...
ls file[123].txt                # exactly file1.txt, file2.txt, or file3.txt
ls [[:upper:]]*                 # files starting with an uppercase letter
cp *.conf /tmp/                 # copy all .conf files to /tmp
rm *.tmp                        # delete all .tmp files
```

---

## Brace Expansion

Generates multiple strings — very useful for batch operations:

```bash
mkdir {jan,feb,mar,apr,may,jun}     # create 6 directories
touch report_{2024,2025,2026}.txt   # create 3 files
touch file{1..10}.txt               # create file1.txt through file10.txt
cp config.txt{,.bak}                # quick backup: copies to config.txt.bak
mkdir -p project/{src,bin,lib,doc}  # create nested structure in one command
```

---

## Variable and Command Substitution

```bash
echo $HOME                          # expand a variable
echo "I am: $(whoami)"              # embed command output in a string
today=$(date +%Y-%m-%d)             # store output in a variable
cp file.txt file-$today.txt         # use variable in a filename
```

### Quoting Rules

| Quotes | Behaviour |
|---|---|
| `"double quotes"` | Variables and `$(commands)` expand; wildcards do NOT expand |
| `'single quotes'` | Everything is treated as literal text — nothing expands |
| `\` backslash | Escapes the very next character only |

```bash
echo "$HOME"            # prints your home path, e.g. /home/student
echo '$HOME'            # prints literally: $HOME
echo "Files: $(ls)"     # embeds the output of ls
```

---

## Things to Remember

- **`rm` is permanent** — no recycle bin, no undo, no warning by default
- Always `ls` or `pwd` before running `rm -rf` with a relative path
- `mv` is rename AND move — they are the same operation in Linux
- `mkdir -p` creates the full path and never errors if it already exists
- Wildcards are expanded by the **shell**, not the command — `ls *.txt` never sees `*.txt`, it sees the list of files
- Hard links share data — editing one edits all; deleting one leaves the others intact
- Symlinks can dangle — if you delete the target, the symlink still exists but points to nothing
- Spaces in filenames are legal but painful — always quote them: `rm "my file.txt"` or avoid spaces entirely
