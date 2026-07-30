# Lesson 3: File Permissions, Ownership & Text Editors (vim, nano)

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-2 (terminal basics, paths, `ls -l` output, inodes)

## Learning Objectives

By end of lesson student can:
- read and interpret full `ls -l` permission string
- change permissions with `chmod` using both numeric and symbolic modes
- change ownership with `chown`/`chgrp`, including recursively
- explain SUID, SGID, sticky bit and identify them in `ls -l` output
- explain what `umask` does and predict default permissions it produces
- edit files in the terminal using both `nano` and `vim`, including basic vim navigation/editing/search

## Topics

- Permission model: user/group/other, rwx, ls -l output interpretation
- chmod: numeric (755, 644) and symbolic (+x, g-w, o=r) modes
- chown, chgrp: changing ownership; recursive (-R); chown user:group
- Special permissions: SUID, SGID, sticky bit; umask
- vim: normal/insert/visual modes, navigation, :wq :q! dd yy p /search
- nano: open, edit, Ctrl+O save, Ctrl+X exit

## Concepts

### Permission model: user / group / other

Every file/directory has exactly one **owner (user)** and one **owning group**, plus a third bucket for **everyone else**. Permissions are defined separately for each of these three:

- **u**ser (owner) — the account that owns the file
- **g**roup (owning group) — any account that's a member of that group
- **o**ther — everyone else on the system

For each of the three, three permission bits apply:

- **r** (read, value 4) — file: view content; directory: list contents
- **w** (write, value 2) — file: modify/delete content; directory: create/delete/rename entries inside
- **x** (execute, value 1) — file: run as a program/script; directory: **enter it** (`cd` into it) / traverse through it

Important nuance: `r` on a directory lets you *list names*, but `x` on a directory is what lets you actually access things inside it (stat them, cd into them). A directory with `r` but no `x` shows filenames via `ls` but you can't `cd` into it or read file details.

### Reading `ls -l` output

```
-rwxr-xr--  1 alice  devs  1240  Jul 30 10:15  deploy.sh
```

Breaking down the leading 10-character string:

```
-  rwx  r-x  r--
│   │    │    │
│   │    │    └─ other: read only
│   │    └────── group: read + execute, no write
│   └─────────── owner: read + write + execute
└─────────────── file type: - regular file, d directory, l symlink
```

Rest of the line: link count (`1`), owner (`alice`), group (`devs`), size in bytes (`1240`), last modified timestamp, filename.

Numeric shorthand for the same permissions: **r=4, w=2, x=1**, sum per group of three.
- `rwx` = 4+2+1 = **7**
- `r-x` = 4+0+1 = **5**
- `r--` = 4+0+0 = **4**
- `rw-` = 4+2+0 = **6**

So `rwxr-xr--` = **754**. This is exactly what `chmod` numeric mode uses.

### chmod — numeric mode

`chmod NNN file` — three digits, one per user/group/other, each digit 0-7 summing r/w/x values.

Common combos:

| Numeric | Meaning | Typical use |
|---|---|---|
| `755` | owner rwx, group r-x, other r-x | executable scripts, directories |
| `644` | owner rw-, group r--, other r-- | normal data files, configs meant to be read by others |
| `600` | owner rw-, no access for group/other | private files (SSH keys, secrets) |
| `700` | owner rwx, no access for group/other | private scripts/directories |
| `777` | everyone full access | **almost never correct** — security red flag, flag this explicitly |
| `750` | owner rwx, group r-x, other none | shared team script, no outside access |

```bash
chmod 755 deploy.sh
chmod 600 id_rsa
chmod -R 644 configs/    # recursive, applies to every file inside
```

### chmod — symbolic mode

Symbolic mode changes bits *relative* to current state, or sets them explicitly, without needing to know/recompute the full numeric value.

Syntax: `chmod [who][operator][permission] file`

- **who**: `u` (user/owner), `g` (group), `o` (other), `a` (all three)
- **operator**: `+` (add), `-` (remove), `=` (set exactly, clearing anything not listed)
- **permission**: any combination of `r`, `w`, `x`

```bash
chmod +x script.sh        # add execute for all three (u+g+o) — most common one
chmod u+x script.sh       # add execute for owner only
chmod g-w file.txt        # remove write from group
chmod o=r file.txt        # set other to read-only exactly, clearing w/x if present
chmod a+r file.txt        # add read for everyone
chmod u+x,g+x file.sh     # combine multiple clauses with commas, no spaces
```

When to prefer symbolic over numeric: when you want to change *one* bit without affecting the others and don't want to mentally recompute the full 3-digit number (e.g. "just make it executable, don't touch anything else" → `chmod +x`, not guessing `755`).

### chown / chgrp — ownership

- `chown user file` — change owner only
- `chown user:group file` — change owner AND group in one command
- `chown :group file` — change group only (equivalent to `chgrp`)
- `chgrp group file` — change group only, dedicated command
- `-R` flag — apply recursively to a directory and everything inside it

```bash
chown alice deploy.sh          # alice becomes owner
chown alice:devs deploy.sh     # alice owner, devs group
chown :devs deploy.sh          # keep owner, change group to devs
chgrp devs deploy.sh           # same group change, dedicated command
chown -R alice:devs /srv/app   # whole directory tree, recursively
```

Only **root** (or via `sudo`) can change a file's owner to someone else — a regular user can't give their own files away to another user (this is a common point of confusion — "why does `chown` fail with Permission denied even on my own file?").

### Special permissions: SUID, SGID, sticky bit

These are a 4th, optional permission "digit" prepended before the normal 3 digits (e.g. `4755`), or represented symbolically inside the normal `rwx` string.

- **SUID (Set User ID)** — on an *executable file*: when run, process executes with the **file owner's** privileges, not the caller's. Classic example: `/usr/bin/passwd` is owned by root with SUID set, so any user can change their own password (which needs write access to `/etc/shadow`, normally root-only) without being root themselves. Shown as `s` in owner's execute position: `rwsr-xr-x`. Numeric: `4` prefix, e.g. `chmod 4755 file`.
- **SGID (Set Group ID)** — on an executable: runs with the **file's group** privileges. On a *directory*: any new file/subdirectory created inside **inherits the directory's group** automatically instead of the creating user's primary group — very useful for shared team directories. Shown as `s` in group's execute position. Numeric: `2` prefix, e.g. `chmod 2775 dir`.
- **Sticky bit** — on a directory: even if a user has write permission to the directory, they can only delete/rename their **own** files inside it, not other users' files. Classic example: `/tmp` has sticky bit set — everyone can write there, but you can't delete someone else's temp files. Shown as `t` in other's execute position: `rwxrwxrwt`. Numeric: `1` prefix, e.g. `chmod 1777 /tmp`.

```bash
chmod 4755 /usr/bin/somebinary   # SUID
chmod 2775 /srv/shared-team-dir  # SGID on a directory
chmod 1777 /tmp                  # sticky bit
chmod u+s file                   # symbolic SUID
chmod g+s dir                    # symbolic SGID
chmod +t dir                     # symbolic sticky bit
```

A lowercase `s`/`t` means the underlying execute bit is also set; an uppercase `S`/`T` means the special bit is set but the corresponding execute bit is NOT — usually a misconfiguration worth pointing out when it shows up in `ls -l`.

### umask

`umask` = a mask that determines the **default** permissions new files/directories get when created, by *subtracting* from the maximum.

- Max default for a new **file**: `666` (rw-rw-rw-) — files never get execute by default from creation (`touch`), only chmod adds it explicitly later.
- Max default for a new **directory**: `777` (rwxrwxrwx).

Default permission = max minus umask, bit by bit. Common default umask on Ubuntu is `022`:
- New file: `666 - 022` = `644` (rw-r--r--)
- New directory: `777 - 022` = `755` (rwxr-xr-x)

```bash
umask           # show current mask, e.g. 0022
umask 077       # much more restrictive: new files 600, new dirs 700
touch test1     # created under whatever umask is currently active
ls -l test1
```

Setting `umask` in a shell only affects that shell session (and its children) unless placed in a startup file (`~/.bashrc` /etc/profile) — good to mention it's often configured system-wide for security-sensitive servers.

### vim

Vim is a **modal** editor — same keys do different things depending on which mode you're in. This is the single biggest conceptual hurdle for beginners.

**Modes:**
- **Normal mode** (default on open) — keys are commands (navigate, delete, copy), not text input. Press `Esc` to always return here from any other mode.
- **Insert mode** — actual typing goes into the file as text. Enter with `i` (insert before cursor), `a` (append after cursor), `o` (open new line below and insert), `O` (open new line above).
- **Visual mode** — select text (like mouse-drag selection) to then act on it. Enter with `v` (character-wise), `V` (line-wise), `Ctrl+v` (block-wise).
- **Command-line mode** — enter with `:`, used for save/quit/search-replace/settings, e.g. `:wq`.

**Navigation (normal mode):**
- `h j k l` — left, down, up, right (arrow keys also work, but hjkl keeps hands on home row)
- `0` / `$` — start / end of current line
- `w` / `b` — jump forward/backward one word
- `gg` / `G` — jump to top / bottom of file
- `:N` (e.g. `:42`) — jump to line N

**Editing (normal mode):**
- `dd` — delete (cut) current line
- `yy` — yank (copy) current line
- `p` — paste after cursor/line
- `x` — delete single character under cursor
- `u` — undo; `Ctrl+r` — redo

**Search:**
- `/pattern` then Enter — search forward for pattern
- `?pattern` — search backward
- `n` / `N` — jump to next / previous match

**Saving and quitting (command-line mode, all start with `:`):**
- `:w` — write (save), stay in editor
- `:q` — quit (fails if unsaved changes)
- `:wq` or `:x` — write and quit
- `:q!` — quit WITHOUT saving, discard changes
- `:wq!` — force write and quit (e.g. overriding a read-only warning)

Beginner survival flow: `i` to start typing → `Esc` when done → `:wq` Enter to save and exit. If things go wrong and you just want out: `Esc` then `:q!` Enter.

### nano

Nano is a much simpler, non-modal editor — you open it and start typing immediately, no mode-switching. Preferred for quick edits by beginners; vim preferred long-term for speed/power once muscle memory builds.

- Open: `nano filename` (creates the file if it doesn't exist)
- Just type normally — no insert mode needed
- `Ctrl+O` — "Write Out" = save (prompts to confirm filename, press Enter to confirm)
- `Ctrl+X` — exit (prompts to save if there are unsaved changes)
- `Ctrl+K` — cut current line
- `Ctrl+U` — paste (uncut)
- `Ctrl+W` — search (Where Is)
- `Ctrl+G` — help menu inside nano
- Bottom of screen always shows a cheat-sheet of available shortcuts (`^` symbol = Ctrl key) — point this out, it's the built-in safety net for beginners

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `chmod NNN file` | set permissions numerically | `chmod 755 script.sh` |
| `chmod who±perm file` | set permissions symbolically | `chmod u+x script.sh` |
| `chmod -R` | apply recursively | `chmod -R 644 configs/` |
| `chown user file` | change owner | `chown alice file.txt` |
| `chown user:group file` | change owner and group | `chown alice:devs file.txt` |
| `chgrp group file` | change group only | `chgrp devs file.txt` |
| `umask` | show/set default permission mask | `umask 022` |
| `chmod 4NNN` / `u+s` | SUID | `chmod u+s file` |
| `chmod 2NNN` / `g+s` | SGID | `chmod g+s dir` |
| `chmod 1NNN` / `+t` | sticky bit | `chmod +t dir` |
| `vim file` | open file in vim | `vim notes.txt` |
| `nano file` | open file in nano | `nano notes.txt` |

## Examples / Walkthrough

```bash
# ── setting up lab files ─────────────────────────────────
mkdir -p ~/lab3 && cd ~/lab3
touch script.sh secret.key shared.txt
ls -l                          # note default perms, umask-driven

# ── reading current permissions ──────────────────────────
stat script.sh                 # from lesson 2, shows Access as octal too
ls -l script.sh

# ── chmod numeric ─────────────────────────────────────────
chmod 755 script.sh
ls -l script.sh                 # rwxr-xr-x
chmod 600 secret.key
ls -l secret.key                 # rw-------
chmod 644 shared.txt
ls -l shared.txt                 # rw-r--r--

# ── chmod symbolic ────────────────────────────────────────
chmod +x script.sh               # equivalent add-execute-for-all
chmod u-x script.sh               # remove execute from owner only
chmod g+w,o-r shared.txt           # combine multiple clauses
ls -l script.sh shared.txt

# ── ownership ──────────────────────────────────────────────
whoami                             # confirm your own username
ls -l script.sh                    # currently owned by you, your primary group
chown root script.sh                # try giving it to root — this will fail:
                                     # "chown: changing ownership: Operation not permitted"
                                     # (only root itself can reassign ownership away
                                     #  from a file — privilege escalation covered
                                     #  properly with sudo in a later lesson)
ls -l script.sh                     # unchanged — confirms it failed safely
chgrp devs shared.txt                # group change works if you belong to "devs"
                                     # (creating groups covered in a later lesson —
                                     #  use whatever group you're already a member of)
ls -l shared.txt

# ── special permissions ───────────────────────────────────
mkdir teamdir
chmod 2775 teamdir                 # SGID on a directory
ls -ld teamdir                      # note the 's' in group execute position

mkdir stickytest
chmod 1777 stickytest
ls -ld stickytest                   # note the 't' at the very end

# ── umask ──────────────────────────────────────────────────
umask                               # show current mask
touch defaultfile1
ls -l defaultfile1                   # e.g. 644 under umask 022
umask 077
touch defaultfile2
ls -l defaultfile2                    # now 600 — much stricter
umask 022                             # reset back to normal for rest of lab

# ── nano quick edit ────────────────────────────────────────
nano shared.txt
# type: "hello from nano"
# Ctrl+O, Enter to save, Ctrl+X to exit
cat shared.txt

# ── vim quick edit ─────────────────────────────────────────
vim script.sh
# press i to enter insert mode
# type: #!/bin/bash
# press Enter, type: echo "hello from vim"
# press Esc to return to normal mode
# type :wq and Enter to save and quit
cat script.sh

# vim navigation/edit practice on an existing file
vim /etc/hostname
# gg (top), G (bottom), /localhost (search), n (next match)
# :q  (quit without editing, since this is a system file — don't save changes here)

# ── cleanup ───────────────────────────────────────────────
cd ~
rm -r lab3
```

## Common Pitfalls

- **`chmod 777` "to make it work"** — students hit a permission error and reach for `777` as a shortcut. Flag this immediately as a security anti-pattern — diagnose the *actual* needed permission instead (usually `644` or `755` is correct, not "everyone gets everything").
- **Confusing numeric digit order** — `chmod 754` sets owner=7, group=5, other=4, in that fixed left-to-right order; swapping digits by mistake gives wrong result silently (no error, just wrong permissions).
- **Forgetting execute bit on scripts** — `./script.sh` fails with "Permission denied" even though the file is readable; students forget scripts need `x` to *run*, `r` alone only lets you view the source.
- **Directory needs `x` to enter, not just `r`** — `chmod 600` on a directory means you can list nothing useful and can't `cd` into it; very confusing error ("Permission denied" on `cd`) until this distinction clicks.
- **`chown` fails with "Operation not permitted"** — regular (non-root) users cannot give away ownership of their own files to someone else; needs `sudo`. Common source of "why doesn't this work" confusion.
- **`S`/`T` (uppercase) instead of `s`/`t` (lowercase) in `ls -l`** — means special bit set but underlying execute bit missing — usually an unintended misconfiguration, worth explicitly pointing out when spotted.
- **Assuming `umask` persists across sessions** — running `umask 077` only affects the current shell (and children); doesn't survive opening a new terminal unless added to a shell startup file.
- **Vim mode confusion** — typing text while accidentally in Normal mode executes it as commands instead of inserting it, scrambling the file (classic beginner panic moment). Remedy: `Esc` then `:q!` to bail out without saving, reopen, try again starting with `i`.
- **Can't figure out how to quit vim** — extremely common (often joked about online). Teach `Esc` then `:q!` as the universal "get me out" escape hatch on day one.
- **Nano "File Name to Write" prompt on Ctrl+O confuses students** — they think it's asking something new; just press Enter to confirm the same filename.
- **SGID on directory misunderstood as "SUID for directories"** — SUID has no meaningful effect on directories at all; only SGID and sticky bit are meaningful on directories. Worth explicitly correcting this mix-up.

## FAQ

**Q: What's the difference between `r` and `x` on a directory?**
A: `r` lets you list the *names* of entries inside (e.g. plain `ls`). `x` lets you actually access/traverse into it — `cd` into it, `stat` files inside, open files inside. You need both `r` and `x` together for normal usable access to a directory's contents.

**Q: Why does `chmod 777` fix my permission problem but my instructor says it's wrong?**
A: It "fixes" it by giving everyone full read/write/execute — including users who should have zero access. It's a security hole, not a real fix. Correct approach is diagnosing exactly which bit was missing for exactly who needs it (usually `chmod +x` for owner, or fixing group membership instead of loosening permissions to everyone).

**Q: How is `750` different from `755`?**
A: `750` = owner rwx, group r-x, other **none** (0). `755` = owner rwx, group r-x, other r-x (readable/executable by everyone). Use `750` when outside/other users shouldn't have any access at all, `755` when it's fine for anyone on the system to read/run it.

**Q: Can I `chown` a file I own to give it to someone else?**
A: Not without `sudo`/root privileges. As a regular user you can `chgrp` to any group you belong to, but you cannot give away *ownership* of your own file to another user — this prevents users from dodging disk quotas or accountability by handing off files.

**Q: What's the practical use case for SUID?**
A: Letting a normal user run a specific privileged operation without giving them full root access. Classic example: `passwd` binary runs as root (via SUID) so any user can update their own password entry in `/etc/shadow`, which normally only root can write to.

**Q: What's the practical use case for SGID on a directory?**
A: Team-shared directories. Normally, new files take the creating user's *primary* group. With SGID set on the directory, every new file/subdirectory created inside automatically inherits the *directory's* group instead — keeps a whole team's shared files under one consistent group without everyone manually `chgrp`-ing afterward.

**Q: What's the practical use case for the sticky bit?**
A: World-writable shared directories (like `/tmp`) where you want everyone to be able to create files, but nobody should be able to delete or rename someone *else's* files. Without it, anyone with write access to the directory could delete anyone else's files there.

**Q: If `umask` is 022, why do new directories end up 755 but new files end up 644, not 755?**
A: The maximum starting point differs: directories start from `777`, regular files start from `666` (files never get execute by default at creation — only chmod or the program creating them explicitly sets `x`). Subtract the same `022` mask from each different starting point and you get `755` vs `644` respectively.

**Q: I'm stuck in vim and can't get out, what do I press?**
A: Press `Esc` (to guarantee you're in Normal mode), then type `:q!` and press Enter — this quits without saving any changes. If you actually want to save first, use `:wq` instead of `:q!`.

**Q: Why did vim start scrambling my file when I just started typing?**
A: You were in Normal mode, where letters are commands, not text — e.g. typing "dog" executes `d` (start of a delete command), `o` (open new line + insert), `g` (part of `gg`/`G` navigation), none of which is "typing text". Always press `i` (or `a`/`o`) first to enter Insert mode before typing content.

**Q: Do I need to learn vim if nano is easier?**
A: Nano is fine for quick beginner edits, but vim (or a vim-like modal editor) is present by default on nearly every Linux server, often the *only* editor pre-installed, and its modal design is much faster once memorized. For DevOps work — SSH'd into a remote server with no GUI — vim fluency saves real time over a career.

**Q: What's the difference between `:q`, `:q!`, and `:wq`?**
A: `:q` quits only if there are no unsaved changes (refuses otherwise). `:q!` force-quits and discards any unsaved changes. `:wq` saves first, then quits.

## Practice / Exercise

**Core (everyone should finish):**
1. Create `script.sh`, set permissions to `755` numerically, verify with `ls -l`.
2. Create `secret.key`, set permissions to `600` numerically, verify.
3. Use symbolic mode to add execute for owner only on a new file, then remove it again, checking `ls -l` after each step.
4. Change the group of a file to a group you belong to using both `chown :group` and `chgrp group` — confirm both produce identical results.
5. Create a directory, apply SGID (`chmod g+s` or `2775`), create a file inside as a demo, and confirm its inherited group.
6. Check your current `umask`, create a file, note its permissions; change `umask` to `077`, create another file, compare permissions between the two.
7. Open a file in `nano`, type 3 lines, save with `Ctrl+O`, exit with `Ctrl+X`, then `cat` the file to confirm.
8. Open the same file in `vim`, enter insert mode, add a 4th line, save and quit with `:wq`, confirm with `cat`.

**Stretch (fast finishers):**
9. In `vim`, practice `dd`/`yy`/`p` on a multi-line file: delete one line, then paste it back in a different position.
10. In `vim`, use `/pattern` to search for a word inside `/etc/hostname` or another small system file (view only — quit with `:q`, do not save).
11. Deliberately set a directory to `600` (no execute), try to `cd` into it, observe the error, then fix it to `700` and retry.
12. Create a sticky-bit directory (`1777`), and (working with a partner if in a multi-user lab environment) confirm you can't delete each other's files inside it even though both have write access.
13. Find (via `ls -l` on system directories like `/usr/bin`) at least one real SUID binary on your system (hint: look for lowercase `s` in the owner execute position) and explain in your own words why it needs SUID.

## Further Reading

- `man chmod`, `man chown`, `man umask` — full official flag reference
- `vimtutor` — interactive vim tutorial built into vim itself, run directly from terminal, ~30 min, strongly recommended for practice outside class
- `man nano` — full nano options and shortcuts
- explainshell.com — paste any `chmod`/`chown` invocation to see it broken down
