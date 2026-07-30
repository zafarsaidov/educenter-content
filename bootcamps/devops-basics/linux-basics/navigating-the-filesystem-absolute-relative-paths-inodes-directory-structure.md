# Lesson 2: Navigating the Filesystem – Absolute/Relative Paths, Inodes & Directory Structure

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lesson 1 (terminal basics: pwd, ls, cd, mkdir, rmdir, touch, cp, mv, rm, cat, less, head, tail, man/--help/whatis/apropos)

## Learning Objectives

By end of lesson student can:
- explain the purpose of the main top-level Linux directories (/, /etc, /var, /home, /usr, /tmp, /proc, /dev)
- move around confidently using both absolute and relative paths
- explain what an inode is and inspect one with `stat`/`ls -i`
- explain difference between hard link and symbolic (soft) link, create both, and know when each breaks
- use `ls` options and `tree` to inspect directory structure at a glance, including hidden files

## Topics

- Linux directory hierarchy: /, /etc, /var, /home, /usr, /tmp, /proc, /dev
- Absolute vs relative paths: ., .., ~, cd -, pwd
- Inodes: concept, stat, ls -i, inode limits, why they matter
- Hard links vs soft links: ln, ln -s, readlink, differences
- ls options: -la, -lh, -R; tree; hidden files (.dotfiles)

## Concepts

### The Linux directory hierarchy (FHS)

Linux follows a standard layout called the **Filesystem Hierarchy Standard (FHS)** — every distro (Ubuntu, RHEL, etc.) organizes `/` roughly the same way, which is why commands/paths transfer between distros.

Everything starts at `/` — the **root** directory. No drive letters (no `C:\`) — every disk, partition, USB stick, or network share gets *mounted* as a subdirectory somewhere under `/`.

| Directory | Purpose |
|---|---|
| `/` | root — top of the entire filesystem tree |
| `/etc` | system-wide **config files** (plain text, editable) — e.g. `/etc/passwd`, `/etc/hosts`, `/etc/ssh/sshd_config` |
| `/var` | **variable** data — things that change/grow at runtime: logs (`/var/log`), mail queues, caches, databases |
| `/home` | personal directory per user — `/home/alice`, `/home/bob`; your own files live here |
| `/usr` | bulk of installed **user-space programs** and their support files (`/usr/bin`, `/usr/lib`, `/usr/share`) — historically "Unix System Resources", think of it as "everything installed on top of the base OS" |
| `/tmp` | temporary files, world-writable, usually **wiped on reboot** — never store anything important here |
| `/proc` | virtual/synthetic filesystem — doesn't exist on disk, kernel generates it live; window into running processes and kernel state (e.g. `/proc/cpuinfo`, `/proc/<pid>/status`) |
| `/dev` | device files — hardware represented as files (e.g. `/dev/sda` = first disk, `/dev/null` = discard-everything device) |

Additional ones worth a mention (not core bullet but commonly asked):
- `/root` — home directory of the root user specifically (not the same as `/`)
- `/bin`, `/sbin` — essential binaries needed even in single-user/rescue mode (on modern Ubuntu these are usually symlinks into `/usr/bin`, `/usr/sbin`)
- `/opt` — optional/third-party software installed outside the package manager
- `/mnt`, `/media` — conventional mount points for manually-mounted or removable filesystems

Mental model to give students: "if it's a setting → `/etc`. If it grows over time → `/var`. If it's a person's stuff → `/home`. If it's fake/kernel-generated → `/proc`. If it's hardware → `/dev`."

### Absolute vs relative paths

- **Absolute path** — starts from `/`, always unambiguous no matter where you currently are. Example: `/home/alice/notes.txt`.
- **Relative path** — starts from your **current working directory** (whatever `pwd` prints). Example: if you're in `/home/alice` and run `cd projects`, that's relative to where you stood.

Special path shortcuts:
- `.` — current directory
- `..` — parent directory (one level up)
- `~` — your home directory (expands to `/home/<you>`, or `/root` for root user)
- `~username` — another user's home directory, e.g. `~bob` → `/home/bob`
- `-` (with `cd`) — previous directory you were in before the last `cd`
- `pwd` — always prints your current **absolute** path, useful to "reset your bearings"

Chaining relative moves: `cd ../../etc` goes up two levels then down into `etc`. `cd ./scripts` and `cd scripts` are identical (`./` is implicit).

### Inodes

An **inode** ("index node") is a data structure the filesystem uses to store all metadata about a file — everything *except* its name and its actual content location on disk in a human sense. Every file/directory has exactly one inode number, unique per filesystem/partition.

What an inode stores: file type, permissions, owner (UID/GID), size, timestamps (access/modify/change), link count, and pointers to the actual data blocks on disk.

What an inode does **not** store: the filename. Filenames live in the *directory entry*, which is just a mapping of "name → inode number". This is why:
- Multiple names can point to the same inode (hard links).
- Renaming a file (`mv`) is instant regardless of file size — you're just changing the directory entry, not touching the data.

Key commands:
- `ls -i` — show inode number next to each file
- `stat filename` — show full inode metadata: size, permissions, timestamps, inode number, number of hard links, block count

**Inode limits** — a filesystem is formatted with a *fixed* number of inodes at creation time (`mkfs`). It's possible to run out of inodes (`df -i` to check) while still having free disk space — typically happens with millions of tiny files (e.g. cache directories, mail spools). `df -h` alone won't reveal this; you'd see "No space left on device" errors even though `df -h` shows free space. Worth flagging as a classic real-world gotcha.

### Hard links vs symbolic (soft) links

Both created with `ln`, but fundamentally different mechanisms:

**Hard link** (`ln target linkname`)
- Creates a second directory entry pointing to the **same inode** as the original.
- Both names are fully equal — no "original" vs "copy", deleting either one just decrements the link count; data survives until link count hits 0.
- Cannot span filesystems/partitions (inode numbers are only unique within one filesystem).
- Cannot hard-link a directory (to prevent filesystem loops) — most systems disallow this.
- `ls -l` link count column shows how many hard links point to that inode.

**Symbolic link / soft link** (`ln -s target linkname`)
- Creates a **new, separate inode** whose content is literally the path string to the target.
- Can point to directories, and can cross filesystems.
- Can point to a target that doesn't exist ("dangling"/"broken" symlink) — no error until you try to use it.
- If you delete the *original* file, the symlink still exists but points nowhere (breaks). Hard links can't do this since there's no "original" — all names are equal.
- `readlink target` (or `readlink -f` for fully resolved absolute path) shows what a symlink points to.
- `ls -l` shows symlinks with a leading `l` in permissions and an `->` arrow to target.

| | Hard link | Symbolic link |
|---|---|---|
| Same inode as target? | Yes | No (own inode, stores a path string) |
| Can link across filesystems? | No | Yes |
| Can link a directory? | No (normally) | Yes |
| Breaks if original deleted? | No (data persists until all links gone) | Yes (becomes dangling) |
| Visually identifiable? | Looks like a normal file | `ls -l` shows `l` + `->` arrow |

### ls options & hidden files, tree

- `ls -l` — long format: permissions, link count, owner, group, size, modified date, name
- `ls -a` — show all, including hidden files/dirs (names starting with `.`, e.g. `.bashrc`, `.git`)
- `ls -la` — combine both, most commonly used combo
- `ls -lh` — long format with human-readable sizes (K/M/G instead of raw bytes)
- `ls -R` — recursive listing, shows contents of all subdirectories too
- `ls -i` — show inode numbers (ties back to inodes section)
- `tree` — visualize whole directory structure as an actual tree diagram (not installed by default on minimal Ubuntu — `apt install tree`, quick mention that installing packages is Lesson 7's topic)

Hidden files ("dotfiles") convention: anything starting with `.` is hidden from default `ls`. Not a security feature — purely a display convention to reduce clutter (mostly config files: `.bashrc`, `.gitconfig`, `.ssh/`). Every directory always contains two special hidden entries: `.` (itself) and `..` (parent) — visible with `ls -a`.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `pwd` | print absolute path of current directory | `pwd` |
| `cd path` | change directory (relative or absolute) | `cd /etc` |
| `cd ..` | move up one level | `cd ..` |
| `cd ~` | go to home directory | `cd ~` |
| `cd -` | go to previous directory | `cd -` |
| `ls -la` | list all (incl. hidden), long format | `ls -la ~` |
| `ls -lh` | long format, human-readable sizes | `ls -lh /var/log` |
| `ls -R` | recursive listing | `ls -R /etc/ssh` |
| `ls -i` | show inode numbers | `ls -i notes.txt` |
| `tree` | visual directory tree | `tree /etc/ssh` |
| `tree -a` | tree including hidden files | `tree -a ~` |
| `tree -L N` | limit tree depth to N levels | `tree -L 2 /` |
| `stat` | show full metadata/inode info for a file | `stat notes.txt` |
| `ln target link` | create hard link | `ln notes.txt notes-hard.txt` |
| `ln -s target link` | create symbolic link | `ln -s notes.txt notes-soft.txt` |
| `readlink link` | show what symlink points to | `readlink notes-soft.txt` |
| `readlink -f link` | fully resolved absolute target path | `readlink -f notes-soft.txt` |

## Examples / Walkthrough

```bash
# ── absolute vs relative paths ───────────────────────────
pwd                        # e.g. /home/student
cd /etc                    # absolute — works from anywhere
pwd
cd ssh                     # relative — now at /etc/ssh
pwd
cd ../..                   # relative — up two levels, back to /
pwd
cd -                       # jump back to /etc/ssh (previous dir)
pwd
cd ~                       # home directory, absolute shortcut
pwd
cd ~/..                    # parent of home (usually /home)
pwd

# ── exploring the hierarchy ───────────────────────────────
ls /                       # top-level directories
less /etc/passwd             # long file — page through it (from lesson 1)
ls -lh /var/log             # human-readable log file sizes
ls /home                    # who has an account on this machine
cat /etc/os-release          # confirm this is Ubuntu (from lesson 1)
head -n 10 /proc/cpuinfo     # kernel-generated, not a real file on disk
ls /proc                    # numbered dirs = running process IDs
ls -la /dev                  # device files representing hardware

# ── hidden files ──────────────────────────────────────────
cd ~
ls              # normal view
ls -a           # now shows .bashrc, .ssh, ., .. etc.
ls -la          # combine with long format

# ── tree view ─────────────────────────────────────────────
tree -L 2 /etc/ssh          # 2 levels deep
tree -a ~                   # include hidden files too (careful, big output)

# ── inodes ────────────────────────────────────────────────
mkdir -p ~/lab2 && cd ~/lab2
touch original.txt
stat original.txt           # full metadata: inode number, links, size, times
ls -i original.txt           # just the inode number

# ── hard link ─────────────────────────────────────────────
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt   # SAME inode number for both
stat original.txt                  # check the Links: field — now 2

rm original.txt              # delete the "original" name
cat hardlink.txt              # still works! data survives, link count -1
ls -li hardlink.txt            # inode unchanged, link count back to 1

# ── symbolic link ─────────────────────────────────────────
touch target.txt
ln -s target.txt softlink.txt
ls -li target.txt softlink.txt   # DIFFERENT inode numbers
ls -l softlink.txt                # notice `l` permission + -> arrow
readlink softlink.txt              # prints: target.txt
readlink -f softlink.txt           # prints full absolute path

rm target.txt                # delete the real target
cat softlink.txt              # error: No such file or directory — dangling link!
ls -l softlink.txt             # ls still shows it, often in red (broken link)

# ── cleanup ───────────────────────────────────────────────
cd ~
rm -r lab2
```

## Common Pitfalls

- **Forgetting where you are** — running a relative-path command from the wrong directory. Habit: run `pwd` before anything destructive or unfamiliar.
- **`cd -` confusion** — it only remembers ONE previous directory, not a full history stack. Second `cd -` just toggles back to where you were before that.
- **Assuming `/tmp` is safe long-term storage** — it's typically cleared on reboot (and sometimes periodically by `systemd-tmpfiles`). Never rely on it for anything you need to keep.
- **Confusing `/etc` and `/var`** — students sometimes look for logs in `/etc` (wrong, that's config) or look for config files in `/var` (wrong, that's runtime/variable data).
- **Treating `/proc` files like normal files** — `cat /proc/cpuinfo` works, but you can't meaningfully `cp` or back up `/proc` — it's generated live by the kernel, size shown by `ls` is often `0` even though `cat` prints content.
- **Thinking hard links are "copies"** — they are not a copy of data, they're a second name for the exact same data. Editing content through one name changes it for all hard-linked names, since there's genuinely only one file.
- **Trying to hard-link a directory** — most systems refuse (`ln: hard link not allowed for directory`) — students expect it to behave like symlink.
- **Trying to hard-link across drives/partitions** — fails with "Invalid cross-device link" — inode numbers are only meaningful within one filesystem.
- **Deleting a symlink's target and being surprised `cat` fails** — this is expected/correct behavior, not a bug — a dangling symlink is a common real troubleshooting scenario (e.g. deployment scripts pointing at `current -> releases/v12` after `v12` got cleaned up).
- **Relative symlinks moved to a different directory break** — a symlink created with a *relative* target path is relative to the symlink's own location, not to wherever you run commands from. Moving the symlink (without moving target too) breaks it. Prefer absolute targets for symlinks you'll move, or `ln -s "$(pwd)/target" link`.
- **`tree` not installed by default** — minimal Ubuntu images lack it; `apt install tree` needed (foreshadow: package management is Lesson 7).

## FAQ

**Q: What's the actual difference between a hard link and a symbolic link, in one sentence?**
A: A hard link is another name for the exact same inode/data (indistinguishable from the "original"); a symbolic link is a separate tiny file whose only content is a path pointing at another file, which can break if that path stops being valid.

**Q: Why can't I hard-link a directory?**
A: Because directories already contain hard-link-like entries (`.` and `..`) that make the directory tree work; allowing arbitrary hard links between directories could create cycles/loops the filesystem tools can't safely traverse. Symlinks are allowed for directories precisely because they don't have this problem (they're just a path reference).

**Q: If I delete a file that has a hard link, is the data gone?**
A: No — the inode's link count just decreases by one. Data is only actually freed once the link count reaches zero (no names point to it anymore) AND no process still has it open.

**Q: Why does `ls -l` show a `-rw-r--r--` for a normal file but `lrwxrwxrwx` for a symlink?**
A: The leading character is the file *type* indicator: `-` = regular file, `d` = directory, `l` = symbolic link (permissions shown for symlinks are basically cosmetic — the actual permissions that matter are the target's).

**Q: What does "dangling symlink" mean and how do I find them?**
A: A symlink whose target no longer exists. `ls -l` often highlights these (red text, depending on terminal color settings) since `ls` tries to resolve the target. You'll learn `find -L` for actively hunting them down in a later lesson.

**Q: Is `/` the same as `C:\` on Windows?**
A: Conceptually similar (top of the filesystem) but Linux has exactly one root regardless of how many disks exist — additional disks get *mounted* as directories under `/` (e.g. a second drive might appear as `/mnt/data`), rather than getting their own drive letter.

**Q: Why does `/proc/cpuinfo` show size 0 with `ls -l` but `cat` prints real content?**
A: `/proc` isn't backed by real disk blocks — the kernel generates its content on-the-fly when read. Size reporting for these virtual files is essentially meaningless/placeholder; content is dynamically computed at read time, not stored anywhere.

**Q: What happens if I run out of inodes but `df -h` shows plenty of free space?**
A: You'll get "No space left on device" errors on file creation despite having disk space, because the filesystem's fixed inode table is full — usually caused by huge quantities of tiny files. Check with `df -i` (inode usage) instead of `df -h` (block/space usage) to diagnose this.

**Q: Difference between `cd -` and `cd ..`?**
A: `cd ..` always goes to the parent directory of wherever you currently are. `cd -` goes to whatever directory you were in immediately before your last `cd` — could be anywhere, not necessarily a parent.

**Q: Why does my symlink break after I move it to another folder?**
A: If the symlink was created with a relative target path, that path is interpreted relative to the symlink's own location. Move the symlink without moving the target and the relative path no longer resolves. Fix: use absolute paths for symlinks you plan to relocate.

**Q: Can a symlink point to another symlink?**
A: Yes — chains of symlinks are allowed (with a system-imposed max chain length to prevent infinite loops). `readlink -f` resolves through the whole chain to the final real path.

## Practice / Exercise

**Core (everyone should finish):**
1. From your home directory, use only relative paths to reach `/etc/ssh`, then confirm with `pwd`.
2. Use `cd -` twice in a row and predict out loud where you'll land each time before running it.
3. List `/var/log` with human-readable sizes; identify the 3 largest files by eye.
4. Create `~/lab2/data.txt`, get its inode number with both `stat` and `ls -i`.
5. Create a hard link to `data.txt` called `data-hard.txt`; confirm same inode number; delete `data.txt`; confirm `data-hard.txt` still has the content.
6. Create a symlink to `data-hard.txt` called `data-soft.txt`; confirm different inode with `ls -li`; use `readlink` to see what it points to.
7. Delete `data-hard.txt`; try `cat data-soft.txt` and observe the dangling-link error.

**Stretch (fast finishers):**
8. Run `tree -L 2 /` — identify which top-level directories are actually symlinks on this system (hint: look for `->` in `ls -la /`).
9. Pick a directory with many small files (e.g. `/etc`); look at its `ls -la` listing and its disk usage `du -sh` side by side — discuss why a directory can hold thousands of tiny files yet the reported size looks small.
10. Create a symlink using a *relative* target path, `cd` elsewhere, `mv` the symlink to a new location, and observe it break. Recreate using an absolute target path and repeat the move — confirm it survives.
11. Explore `/proc/self` — explain in your own words what "self" means here (hint: it changes depending on which process reads it).

## Further Reading

- `man hier` — full Filesystem Hierarchy Standard manual page on Ubuntu
- Filesystem Hierarchy Standard spec: https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.pdf
- `man 2 stat` — low-level details of what metadata an inode actually stores
- `man ln` — full hard link / symlink options and edge cases
