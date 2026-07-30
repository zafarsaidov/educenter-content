# Lesson 1: DevOps Concept. Introduction to Linux & Essential Commands

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** none

## Learning Objectives

By end of lesson student can:
- explain what DevOps is, why it exists, what problem it solves vs traditional Dev/Ops split
- name main stages of CI/CD lifecycle and give tool example for each
- explain why Linux (and specifically Ubuntu) dominates server/DevOps world
- navigate filesystem and manage files/directories with core Linux commands
- read and write files from terminal without an editor
- get help for any unfamiliar command without leaving terminal or googling

## Topics

- What is DevOps: culture, CI/CD lifecycle, toolchain overview
- Linux distributions overview, why Ubuntu for DevOps
- Terminal basics: pwd, ls, cd, mkdir, rmdir, touch
- File operations: cp, mv, rm, cat, echo, less, head, tail
- Getting help: man, --help, whatis, apropos

## Concepts

### What is DevOps

DevOps = culture/practice/set of tools merging **Dev**elopment and **Op**erations into one team, one responsibility, instead of two teams throwing work over a wall. Term coined ~2009 (Patrick Debois, "DevOpsDays").

Problem it solves — traditional split:
- Devs write code, "it works on my machine", throw it to Ops.
- Ops runs production, doesn't understand the code, afraid of change → slows releases down.
- Result: blame culture, slow releases (months), fragile deploys, 3am firefighting.

DevOps principles (sometimes called **CALMS**):
- **C**ulture — shared responsibility, no blame, "you build it, you run it"
- **A**utomation — scripts/pipelines instead of manual steps (manual = error-prone, unrepeatable)
- **L**ean — small batches, minimize waste, fast feedback
- **M**easurement — metrics/monitoring drive decisions, not gut feeling
- **S**haring — knowledge, tools, incidents shared across team, not siloed

Business outcome: teams ship code multiple times a day instead of once a quarter, and rollback is minutes not days.

### CI/CD lifecycle

- **CI (Continuous Integration)** — every code change is automatically built and automatically tested the moment it's pushed/merged. Goal: catch bugs within minutes of writing them, not weeks later during a "release freeze".
- **Continuous Delivery** — every change that passes CI is automatically packaged into a deployable artifact and staged, ready to release with one click/approval.
- **Continuous Deployment** — goes one step further: passes CI → auto-deployed to production with **no human approval step**. Requires strong automated test coverage and monitoring/rollback safety nets.

Typical pipeline stages, in order:
1. **Source** — developer pushes to Git (trigger)
2. **Build** — compile code / build container image
3. **Test** — unit tests, integration tests, linting
4. **Scan** — security/vulnerability scanning of code and images
5. **Package/Artifact** — produce versioned build output (Docker image, jar, binary)
6. **Deploy** — push artifact to target environment (dev → staging → prod)
7. **Verify** — smoke tests, health checks post-deploy
8. **Monitor** — metrics/logs/alerts feed back into next dev cycle

This whole bootcamp = one lesson block per stage of this pipeline, deep-dived.

### DevOps toolchain overview

| Stage | Example tools | Covered in bootcamp |
|---|---|---|
| Version control | Git, GitHub, GitLab | Lessons 14-15 |
| Scripting/automation glue | Bash, Python | Lessons 16-19 |
| Infrastructure as Code | Terraform | Lessons 20-23 |
| Config management | Ansible | Lessons 24-29 |
| Containers | Docker | Lessons 30-33 |
| Orchestration | Kubernetes | Lessons 34-52 |
| Packaging for k8s | Helm, Kustomize | Lessons 53-58 |
| CI/CD | GitLab CI, GitHub Actions, ArgoCD | Lessons 59-66 |
| Monitoring/Logging | Prometheus, Grafana, Loki | Lessons 67-72 |

Point for students: DevOps isn't one tool, it's a toolchain — this course walks the whole chain left to right, one link at a time. Today's Linux lesson is the foundation everything else runs on top of.

### Linux distributions & why Ubuntu

**Kernel vs distribution:** Linux itself is just the kernel (core talking to hardware). A "distribution" (distro) = kernel + package manager + default utilities + init system bundled and shipped together.

Major distro families:
- **Debian family** — Debian, Ubuntu, Linux Mint. Package manager: `apt`/`dpkg`, `.deb` packages.
- **Red Hat family** — RHEL, CentOS/Rocky/AlmaLinux, Fedora. Package manager: `yum`/`dnf`, `.rpm` packages.
- **Alpine** — minimal, musl-libc based, common as Docker base image (small size).
- **Arch** — rolling release, `pacman`, popular for desktops/enthusiasts, rare in production servers.

Why Ubuntu chosen for DevOps work and this bootcamp:
- Massive community + documentation, easiest to Google-fix issues as beginner
- APT package manager is straightforward, dependency resolution "just works" most of the time
- Default/first-class OS image on AWS, GCP, Azure, DigitalOcean — spin up in seconds
- LTS (Long Term Support) releases every 2 years, supported 5 years — production wants stability, not bleeding edge
- Most Docker official images use Ubuntu/Debian base or are compatible with apt-based tooling taught here

Worth mentioning live: RHEL/CentOS still common in enterprise/banking — commands differ (`yum install` vs `apt install`) but concepts identical. Once you know Ubuntu, switching distro is a day of relearning package manager syntax, not a new skillset.

### Shell vs terminal vs kernel (clear this confusion early)

- **Kernel** — core program managing hardware, processes, memory.
- **Shell** — program that reads your typed commands and asks kernel to execute them (bash, zsh, sh, fish). Bash is default/most common for DevOps.
- **Terminal / terminal emulator** — the window/app you type into, which runs a shell inside it.

Analogy: kernel = car engine, shell = steering wheel/pedals, terminal = the car's cabin you sit in.

## Commands / Syntax Reference

### Navigation

| Command | Purpose | Example |
|---|---|---|
| `pwd` | print current working directory | `pwd` |
| `ls` | list directory contents | `ls -la` |
| `ls -l` | long format (permissions, owner, size, date) | `ls -l` |
| `ls -a` | show hidden files (dotfiles) | `ls -a` |
| `ls -h` | human-readable sizes (with -l) | `ls -lh` |
| `cd` | change directory | `cd /var/log` |
| `cd ~` or `cd` | go to home directory | `cd` |
| `cd -` | go to previous directory | `cd -` |
| `cd ..` | go up one level | `cd ..` |

### Creating / removing

| Command | Purpose | Example |
|---|---|---|
| `mkdir` | create directory | `mkdir project` |
| `mkdir -p` | create nested dirs, no error if exists | `mkdir -p a/b/c` |
| `rmdir` | remove empty directory only | `rmdir project` |
| `touch` | create empty file, or update timestamp of existing | `touch notes.txt` |
| `rm` | remove file | `rm notes.txt` |
| `rm -r` | remove directory recursively | `rm -r old_dir` |
| `rm -rf` | force recursive remove, no prompts | `rm -rf tmp_dir` |
| `rm -i` | prompt before every removal (safety) | `rm -i *.log` |

### Copy / move

| Command | Purpose | Example |
|---|---|---|
| `cp` | copy file | `cp a.txt b.txt` |
| `cp -r` | copy directory recursively | `cp -r dir1 dir2` |
| `cp -v` | verbose (print what's copied) | `cp -v a.txt b.txt` |
| `mv` | move or rename file/dir | `mv a.txt archive/` |
| `mv` (rename) | same command, same dir = rename | `mv old.txt new.txt` |

### Viewing file content

| Command | Purpose | Example |
|---|---|---|
| `cat` | print whole file to stdout | `cat notes.txt` |
| `cat -n` | print with line numbers | `cat -n notes.txt` |
| `echo` | print text to stdout | `echo "hi"` |
| `less` | page through file, searchable, doesn't load whole file into memory | `less bigfile.log` |
| `more` | older pager, less features than `less` | `more file.log` |
| `head` | show first N lines (default 10) | `head -n 20 file.log` |
| `tail` | show last N lines (default 10) | `tail -n 20 file.log` |
| `tail -f` | follow file live (great for logs) | `tail -f /var/log/dpkg.log` |
| `wc` | count lines/words/chars | `wc -l file.log` |

> Note: writing text *into* a file (`>`, `>>`) is redirection — covered properly in Lesson 4 (Piping and redirection). Today `echo` is used only to print to the terminal; file-viewing commands are practiced against files that already exist on the system.

### Getting help

| Command | Purpose | Example |
|---|---|---|
| `man` | full manual page for command | `man ls` |
| `man -k` | search man pages by keyword (same as apropos) | `man -k copy` |
| `--help` | quick usage summary, built into most tools | `ls --help` |
| `whatis` | one-line description of command | `whatis grep` |
| `apropos` | search commands by keyword in description | `apropos "copy files"` |
| `info` | alternative to man, more structured for GNU tools | `info coreutils` |
| `type` | shows if word is builtin, alias, or binary + path | `type cd` |
| `which` | shows path of executable that would run | `which python3` |

### Inside the `less`/`man` pager (very common beginner blocker)

| Key | Action |
|---|---|
| `Space` / `f` | next page |
| `b` | previous page |
| `↑`/`↓` or `j`/`k` | scroll one line |
| `/pattern` | search forward |
| `n` / `N` | next/previous search match |
| `g` / `G` | go to top / bottom of file |
| `q` | quit |

## Examples / Walkthrough

```bash
# ── where am I, what's here ──────────────────────────────
pwd
echo "hello devops"    # just prints to terminal, no file involved yet
ls -la                 # long + hidden files
ls -lh                 # long + human-readable sizes

# ── set up a lab workspace ───────────────────────────────
mkdir -p ~/devops-lab
cd ~/devops-lab
pwd

# ── create empty files (content-writing comes in Lesson 4) ──
touch app.py config.yaml README.md
ls -l                  # notice all sizes are 0

# ── viewing content of files that already exist on the system ──
cat /etc/os-release          # small file, whole content at once
cat -n /etc/hostname         # with line numbers
head -n 5 /etc/passwd        # first 5 lines
tail -n 5 /etc/passwd        # last 5 lines
wc -l /etc/passwd            # count lines
less /var/log/dpkg.log       # page through a real log, q to quit
tail -f /var/log/dpkg.log    # follow live — this log grows whenever software
                              # gets installed on the system (more on that in
                              # a later lesson); Ctrl+C to stop following

# ── copy vs move vs rename ───────────────────────────────
cp README.md README.bak
ls -l
mkdir backup
mv README.bak backup/README.bak
mv config.yaml config.yml     # rename (same directory = rename)
ls -l
ls -l backup/

# ── cleanup ───────────────────────────────────────────────
rm -i backup/README.bak       # -i asks for confirmation
rmdir backup                  # empty now, rmdir works
rm app.py config.yml README.md
ls -la                        # only . and .. left
cd ..
rmdir devops-lab              # empty directory, rmdir succeeds

# ── getting help live-demo ───────────────────────────────
man tail          # full doc, q to quit
tail --help       # quick flags only
whatis tail       # one-liner
apropos "list directory contents"
type cd           # shell builtin, not a program
which python3     # path to binary
```

## Common Pitfalls

- **`rm` has no undo, no trash bin** — terminal deletion is permanent. Always double-check path before `rm -r`, especially with wildcards (`rm -r *`) or when `cd`-ing somewhere unfamiliar first.
- **`rm -rf /` or `rm -rf ~` disasters** — classic horror story; emphasize checking `pwd` before destructive commands run in a script.
- **Confusing `cp`/`mv` argument order** — syntax always `source destination`. Swapping them silently overwrites the wrong file with no warning by default.
- **Forgetting `-r` for directories** — `cp`/`rm` refuse or behave unexpectedly on directories without `-r` ("omitting directory" or "is a directory" error).
- **`rmdir` fails on non-empty directory** — students expect it to work like `rm -r`; it intentionally doesn't (safety feature).
- **`man` vs `--help` confusion** — `man` opens a full pager requiring `q` to exit; students think terminal froze. `--help` just prints and returns immediately — good habit: try `--help` first, `man` when you need full detail.
- **Case sensitivity** — `README.md` ≠ `readme.md` ≠ `Readme.MD` on Linux (unlike Windows/macOS default). Typos here are a very common source of "file not found".
- **Spaces in filenames** — `mv my file.txt other.txt` is parsed as 3 arguments, not 2. Need quotes: `mv "my file.txt" other.txt`. Best practice: avoid spaces in filenames entirely (use `-` or `_`).
- **Relative vs absolute path mistakes** — running a command from wrong directory because student didn't check `pwd` first.
- **Tab completion underused** — beginners type full paths/filenames manually; press `Tab` to autocomplete and avoid typos, `Tab Tab` to list options.

## FAQ

**Q: What's the difference between DevOps and just being a sysadmin?**
A: A sysadmin traditionally only runs/maintains infrastructure after developers hand off finished code. A DevOps engineer is involved across the whole lifecycle — build, test, deploy, monitor — and focuses on automating as much of that as possible instead of doing manual, one-off ops work.

**Q: Is DevOps a job title or a culture/methodology?**
A: Both in practice. Originally a culture/methodology (breaking down Dev/Ops silos). Industry later turned it into a job title ("DevOps Engineer") for someone who builds the automation/tooling that enables that culture.

**Q: Why not just use Windows for servers?**
A: Linux dominates server space because it's open-source (free, auditable, customizable), has a much lighter resource footprint, and virtually the entire modern cloud-native tooling ecosystem (Docker, Kubernetes, most CI/CD runners) is built Linux-first. Windows Server exists and is used (especially .NET shops) but is a minority in DevOps/cloud-native contexts.

**Q: Difference between Continuous Delivery and Continuous Deployment? Everyone mixes these up.**
A: Delivery = every passing change is automatically packaged and ready to deploy, but a human clicks "release". Deployment = no human click at all, passing change goes straight to production automatically. Deployment requires more trust in your automated test suite.

**Q: Difference between `rm` and `rmdir`?**
A: `rmdir` only removes empty directories — safer, limited. `rm -r` removes a directory and everything inside it recursively — more powerful, more dangerous, no confirmation by default.

**Q: I typed `man ls` and the terminal looks stuck — what happened?**
A: Not stuck — you're inside a pager program. Press `q` to quit. Use arrow keys or space to scroll, `/word` then Enter to search inside the page.

**Q: What's the difference between `cat`, `less`, `head`, `tail`?**
A: `cat` dumps the whole file at once — fine for short files, unusable for huge logs. `less` lets you scroll through large files without loading everything into your terminal at once and lets you search. `head`/`tail` show just the first/last N lines — useful for peeking at the start or end of a large log without opening the whole thing.

**Q: When would I use `tail -f`?**
A: When watching a log file that's actively being written to (e.g. an application log during a deploy) and you want new lines to appear live in your terminal as they're written, instead of re-running `tail` repeatedly.

**Q: `touch` made my files but `ls -l` shows them as 0 bytes — how do I put text in them?**
A: `touch` only creates an empty file (or updates its timestamp) — it never writes content. Writing content into a file from the terminal uses redirection (`>`, `>>`), covered in Lesson 4. For now, practice viewing commands against files that already exist on the system, or use an editor (`nano`, next lesson) if you want to see real content.

**Q: Why does `ls` show some files but not others in my home directory?**
A: Files/directories starting with `.` (dotfiles, e.g. `.bashrc`, `.ssh`) are hidden by default — convention for config files that shouldn't clutter normal listings. Use `ls -a` to see them.

**Q: Can I undo a `mv` or `rm` by mistake?**
A: `mv` — yes, just `mv` it back (if you remember destination). `rm` — no built-in undo. Some systems have recovery tools (`extundelete`, backups) but don't rely on them; treat `rm` as permanent.

**Q: What's the difference between `apropos` and `man -k`? Both were mentioned.**
A: They're the same thing — `man -k` is literally an alias/wrapper for `apropos`. Either works.

**Q: Do I need to memorize every flag for every command?**
A: No. Memorize the handful used constantly (`-la`, `-r`, `-f`, `-h`). For everything else, `--help` or `man` is one keystroke away — knowing *how to look it up fast* matters more than memorizing everything.

## Practice / Exercise

**Core (everyone should finish):**
1. Create directory `~/devops-lab`, move into it.
2. Inside, create 3 empty files using `touch`: `app.log`, `config.txt`, `readme.md`.
3. Show first 5 lines, then last 5 lines, of `/etc/passwd`.
4. Count how many lines are in `/etc/passwd` using `wc -l`.
5. Copy `config.txt` to `config.bak`, then rename `readme.md` to `README.md`.
6. Use `apropos` (or `man -k`) to find the command name for "print a line matching a pattern" — don't use it yet, just find its name (it's coming in Lesson 4).
7. Delete `config.bak`, then remove the whole `devops-lab` directory in one command.

**Stretch (fast finishers):**
8. Create nested directories `project/src/utils` in a single `mkdir` command.
9. Create a file with a space in its name, then successfully `cat` it (practice quoting).
10. Run `tail -f /var/log/dpkg.log` in one terminal, leave it running a minute, note whether anything new appears — discuss what would cause new lines to show up. Ctrl+C to stop following.
11. Use `type` to check whether `cd`, `ls`, and `pwd` are shell builtins or separate binaries — discuss why `cd` must be a builtin (hint: think about what changing a *subprocess's* directory would/wouldn't affect).
12. Compare `man ls`, `ls --help`, and `whatis ls` output side by side — note what level of detail each gives.

## Further Reading

- Ubuntu Server docs: https://ubuntu.com/server/docs
- `man bash` — full shell manual
- explainshell.com — paste any command to see flag-by-flag breakdown
- "The Phoenix Project" (novel) / "The DevOps Handbook" — foundational DevOps culture reading, good optional recommendation for motivated students
- tldr.pages.dev — community-driven, example-first alternative to man pages (much faster to read than official man pages)
