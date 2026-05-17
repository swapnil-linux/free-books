# Chapter 8 – Editing Text Files
## RH124 Student Quick Reference

---

## Why Text Editors Matter in Linux

In Linux, **everything is configured through plain text files** — there is no GUI equivalent of the Windows Registry or Control Panel for most settings. To manage a Linux server you need to be comfortable editing text files from the command line.

This matters even more on servers — most Linux servers have no graphical desktop installed at all.

---

## Your Options: Which Editor to Use?

| Editor | Learning Curve | Available | Best For |
|---|---|---|---|
| **Vim / vi** | Steep | On virtually every Linux system ever made | Production servers, SSH sessions, scripting |
| **nano** | Gentle | Most systems, or easily installed | Beginners, quick edits |
| **gedit / geany** | Easy | GUI systems only | Desktop Linux with a graphical interface |
| **VS Code** | Easy | Install required | Developers on desktop Linux |

> **Why learn Vim?** Because `vi` or `vim` is available on **every** Linux, Unix, and macOS system — including minimal server installs where nothing else is available. If you can only learn one editor, make it Vim.

> **For beginners:** `nano` is perfectly valid for everyday use. It shows its commands on screen — no memorisation needed.

---

## nano — The Beginner-Friendly Editor

nano shows available commands at the bottom of the screen. `^` means `Ctrl`.

```bash
nano filename.txt           # open (or create) a file
```

### nano Key Commands

| Key | Action |
|---|---|
| `Ctrl+O` then `Enter` | Save the file (Write Out) |
| `Ctrl+X` | Exit (will prompt to save if unsaved changes) |
| `Ctrl+K` | Cut current line |
| `Ctrl+U` | Paste (Uncut) |
| `Ctrl+W` | Search (Where is) |
| `Ctrl+\` | Find and replace |
| `Ctrl+G` | Show help |
| `Ctrl+C` | Show current cursor position (line/column) |
| `Alt+U` | Undo |
| `Alt+E` | Redo |

> nano is available on most systems. On RHEL: `sudo dnf install nano`. On Ubuntu: `sudo apt install nano`.

---

## Vim — The Powerful Editor

Vim is **modal** — the same keys do different things depending on which mode you are in. This is the biggest difference from every other editor you have used.

### The Golden Rule

> **If in doubt, press `Esc` a few times. It always returns you to command mode.**

---

### Vim Modes

```
┌─────────────────────────────────────────────────────────┐
│                    COMMAND MODE                          │
│               (default when Vim opens)                   │
│    navigate, delete, copy, paste, search                 │
│                                                          │
│  press i, a, o, O  ──────────────→  INSERT MODE         │
│  (type text)              press Esc ←────────────────┘  │
│                                                          │
│  press v, V, Ctrl+V ─────────────→  VISUAL MODE         │
│  (select text)            press Esc ←────────────────┘  │
│                                                          │
│  press :  ───────────────────────→  EXTENDED CMD MODE    │
│  (save, quit, search/replace)     press Esc ←────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

### Opening and Closing Vim

```bash
vim filename.txt            # open a file (creates it if it does not exist)
vi filename.txt             # same — vi and vim are usually the same on modern Linux
vim +42 filename.txt        # open file with cursor on line 42
vimtutor                    # built-in interactive tutorial — highly recommended
```

### Saving and Quitting (Extended Command Mode — press `:` first)

| Command | Action |
|---|---|
| `:w` | Save (write) the file |
| `:wq` or `:x` | Save and quit |
| `:q` | Quit (only works if no unsaved changes) |
| `:q!` | **Force quit without saving** — discards all changes |
| `:w filename.txt` | Save as a new filename |

---

### Entering Insert Mode (from Command Mode)

| Key | Where Cursor Goes |
|---|---|
| `i` | Insert before the cursor |
| `a` | Append after the cursor |
| `I` | Insert at beginning of line |
| `A` | Append at end of line |
| `o` | Open new line below and insert |
| `O` | Open new line above and insert |

---

### Navigation (Command Mode — no modifier keys needed)

```
        k  (up)
        ↑
h ←     ·     → l
        ↓
        j  (down)
```

| Key / Command | Movement |
|---|---|
| `h j k l` | Left / down / up / right |
| Arrow keys | Also work for navigation |
| `w` | Jump forward one word |
| `b` | Jump backward one word |
| `0` (zero) | Jump to beginning of line |
| `$` | Jump to end of line |
| `gg` | Jump to first line of file |
| `G` | Jump to last line of file |
| `:42` | Jump to line 42 |
| `Ctrl+F` | Page down |
| `Ctrl+B` | Page up |

---

### Editing in Command Mode (without entering Insert mode)

| Key | Action |
|---|---|
| `x` | Delete character under cursor |
| `dd` | Delete (cut) entire current line |
| `3dd` | Delete 3 lines |
| `dw` | Delete from cursor to end of word |
| `D` | Delete from cursor to end of line |
| `yy` | Yank (copy) current line |
| `3yy` | Yank 3 lines |
| `p` | Paste after cursor |
| `P` | Paste before cursor |
| `u` | Undo last change |
| `Ctrl+R` | Redo |
| `r` | Replace single character under cursor |
| `cw` | Change (delete + enter insert) to end of word |
| `.` | Repeat last change |

---

### Searching in Vim (Command Mode)

| Command | Action |
|---|---|
| `/word` | Search forward for *word* |
| `?word` | Search backward for *word* |
| `n` | Next match |
| `N` | Previous match |
| `:%s/old/new/g` | Replace all occurrences of *old* with *new* |
| `:%s/old/new/gc` | Replace all, with confirmation prompt |

---

### Visual Mode — Selecting Text

| Key | Selects |
|---|---|
| `v` | Character-by-character selection |
| `V` (Shift+V) | Whole line selection |
| `Ctrl+V` | Block/column selection |

After selecting: `y` to yank (copy), `d` to delete, `>` to indent, `<` to unindent.

---

### Extended Commands (press `:` from Command Mode)

| Command | Action |
|---|---|
| `:set number` | Show line numbers |
| `:set nonumber` | Hide line numbers |
| `:set hlsearch` | Highlight search results |
| `:syntax on` | Enable syntax highlighting |
| `:help` | Open built-in help |

---

## Vim Config File (Optional Customisation)

To make settings permanent, add them to `~/.vimrc` (your personal config):

```bash
vim ~/.vimrc
```

Useful settings to add:
```
set number          " show line numbers
set hlsearch        " highlight search results
set tabstop=4       " tab = 4 spaces
set autoindent      " auto-indent new lines
syntax on           " syntax highlighting
```

---

## Quick-Edit Alternatives (No Editor Needed)

For simple operations, you can edit files without opening an editor:

```bash
# Append a line to a file
echo "new line of text" >> /etc/hosts

# Overwrite a file completely
echo "only this line" > file.txt

# Replace a specific word in a file (sed)
sed -i 's/oldword/newword/g' filename.txt

# Replace on a specific line number
sed -i '5s/old/new/' filename.txt
```

> ⚠️ `>` **overwrites** the entire file. `>>` **appends** to the end. Do not confuse them.

---

## The `vimtutor` Command

The best way to learn Vim is to use the built-in interactive tutorial:

```bash
vimtutor
```

It takes about 30 minutes and teaches all the essential commands hands-on. It is available on any system with `vim-enhanced` installed.

---

## Windows Comparison

| Windows | Linux Equivalent |
|---|---|
| Notepad (no CLI equivalent) | `nano` — simple, shows commands on screen |
| Notepad++ / VS Code | `vim` — powerful, available everywhere |
| `type file > newfile` | `cat file > newfile` or `cp file newfile` |
| Find in file (Ctrl+F) | `/searchterm` in vim, `grep` on command line |
| Find and Replace (Ctrl+H) | `:%s/old/new/g` in vim, `sed -i` on command line |

---

## Things to Remember

- **Press `Esc` whenever you are lost in Vim** — it always returns to command mode
- `:q!` discards all changes and exits — useful when you have made a mess
- `nano` is perfectly fine for most day-to-day editing — do not let Vim intimidate you
- Vim is modal — the same key does different things in different modes
- `vimtutor` is the single best way to learn Vim — it takes 30 minutes and covers everything
- `vi` is available even on the most minimal Linux install — learning it pays off on any server
- All Linux config files are plain text — if something breaks after editing, you can always revert
