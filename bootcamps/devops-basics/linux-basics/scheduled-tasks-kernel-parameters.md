# Lesson 9: Scheduled Tasks & Kernel Parameters

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-8 (terminal, filesystem, permissions, find/grep/piping, systemd, users/sudo, packages, process/job management)

## Learning Objectives

By end of lesson student can:
- Write valid cron schedule expressions and explain the 5-field syntax plus special strings
- Manage a user's crontab with `crontab -e/-l/-r`, and explain the system-wide cron drop-in locations
- Schedule a one-time future task with `at`, and manage the queue with `atq`/`atrm`
- Explain what `/proc/sys` and `sysctl` expose, and safely view/change a kernel parameter

## Topics

- cron syntax: 5 fields (min hour day month weekday), special strings (`@reboot`, `@daily`)
- crontab: `-e`, `-l`, `-r`, `-u`; `/etc/cron.d`, `/etc/cron.daily/weekly/monthly`
- `at`, `atd`: scheduling one-time tasks; `atq`, `atrm`
- Kernel parameters: `/proc/sys/`, `/etc/sysctl.conf`, `sysctl -w`/`-p`; common params (`ip_forward`, `swappiness`, `file-max`)

## Concepts

### Why scheduling exists

Two different needs come up constantly in operations: "run this every day at 2am forever" (recurring) and "run this once, in 20 minutes, then forget it" (one-time). Linux ships a tool for each: **cron** for recurring jobs, **at** for one-time jobs. Both rely on a background daemon (`cron`/`crond`, `atd`) that wakes up, checks if anything is due, and runs it — you never have to keep a terminal open.

### cron and crontab syntax

A **crontab** (cron table) is a file listing scheduled commands, one per line. Each line has 5 time fields followed by the command to run:

```
# minute hour day-of-month month day-of-week   command
  *      *    *            *     *              command_to_run
  0-59   0-23 1-31         1-12  0-7 (0 and 7 = Sunday)
```

Each field accepts:

| Syntax | Meaning | Example |
|---|---|---|
| `*` | every value | `* * * * *` → every minute |
| `N` | exact value | `30 2 * * *` → 02:30 every day |
| `N,M` | list of values | `0,30 * * * *` → minute 0 and 30 of every hour |
| `N-M` | range | `9-17 * * * *` → hours 9 through 17 |
| `*/N` | every N units | `*/15 * * * *` → every 15 minutes |

All 5 fields must match the current time for the job to fire that minute. `0-59 0-23 1-31 1-12 0-7` — read left to right as minute, hour, day-of-month, month, day-of-week; day-of-month and day-of-week are OR'd together when both are restricted (either match triggers it).

cron also accepts special strings in place of the 5 fields, as shortcuts:

| String | Equivalent to |
|---|---|
| `@reboot` | Run once, at system startup |
| `@yearly` / `@annually` | `0 0 1 1 *` |
| `@monthly` | `0 0 1 * *` |
| `@weekly` | `0 0 * * 0` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@hourly` | `0 * * * *` |

### Where cron jobs live

Each user has their own crontab, edited with `crontab -e` and stored (managed by the `crontab` tool — you should never edit it directly) under `/var/spool/cron/crontabs/`. This is the right place for personal, per-user scheduled tasks.

For system-wide jobs, Ubuntu also reads:

| Location | Purpose |
|---|---|
| `/etc/cron.d/` | Drop-in files, same 5-field syntax but with an extra **user** field (system crontabs run as any user, not just the file owner) |
| `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/` | Directories of executable scripts (no cron syntax at all) — cron runs everything inside on the matching schedule, driven by `/etc/crontab` |
| `/etc/crontab` | The system-wide crontab that triggers the `cron.daily`/`weekly`/`monthly` directories, and can hold its own 6-field entries (5 time fields + user) |

Rule of thumb: personal task → `crontab -e`. System task that should survive even if a particular user account is removed → `/etc/cron.d/`.

### `at` — one-time scheduled jobs

`at` schedules a command to run exactly once at a specified future time, then the job is gone. It depends on the `atd` daemon running in the background (much like `cron` depends on `cron`/`crond`) — the job sits in a queue until its time arrives, even across a reboot as long as `atd` starts again. `at` is the right tool when cron would be overkill — e.g. "run this cleanup script in 30 minutes" doesn't need a permanent recurring schedule entry.

### Kernel parameters: `/proc/sys` and `sysctl`

The Linux kernel exposes hundreds of tunable runtime parameters through the virtual filesystem `/proc/sys/` (a sibling of the per-process `/proc/<PID>/` directories you used in Lesson 8 — same virtual filesystem, different subtree). Each file under `/proc/sys/` holds one parameter's current value; reading the file shows the value, writing to it changes the value immediately, live, with no restart required.

`sysctl` is a friendlier command-line wrapper around the same `/proc/sys/` files — it translates dotted parameter names (e.g. `net.ipv4.ip_forward`) to their file path (`/proc/sys/net/ipv4/ip_forward`) automatically. Changes made with `sysctl -w` (or by writing to `/proc/sys/` directly) are **not persistent** — they're lost on reboot. To make a change persistent, it must also be written into `/etc/sysctl.conf` (or a file under `/etc/sysctl.d/`), which `sysctl -p` reads and re-applies.

Some commonly tuned parameters:

| Parameter | Meaning |
|---|---|
| `net.ipv4.ip_forward` | Whether the kernel forwards IP packets between interfaces (0=off, 1=on) — required for a machine acting as a router/NAT gateway |
| `vm.swappiness` | How aggressively the kernel swaps memory to disk (0-100; lower = prefer keeping things in RAM) |
| `fs.file-max` | System-wide maximum number of open file handles allowed at once |

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `crontab -e` | Edit current user's crontab | `crontab -e` |
| `crontab -l` | List current user's crontab | `crontab -l` |
| `crontab -r` | Remove current user's entire crontab | `crontab -r` |
| `crontab -u` | Operate on another user's crontab (needs root) | `sudo crontab -u alice -l` |
| `at` | Schedule a one-time job | `echo 'command' \| at 15:30` |
| `atq` | List pending `at` jobs | `atq` |
| `atrm` | Remove a pending `at` job | `atrm 3` |
| `sysctl <param>` | Read a kernel parameter's current value | `sysctl vm.swappiness` |
| `sysctl -w` | Set a parameter live (until reboot) | `sudo sysctl -w vm.swappiness=10` |
| `sysctl -p` | Reload persistent parameters from `/etc/sysctl.conf` | `sudo sysctl -p` |
| `sysctl -a` | List every current kernel parameter | `sysctl -a \| head` |

## Examples / Walkthrough

```bash
# --- crontab: personal recurring jobs ---
crontab -l                              # show current user's crontab (likely empty at first)

# opens crontab in your $EDITOR (vim/nano, from Lesson 3)
crontab -e
# add a line like:
#   */5 * * * * echo "heartbeat: $(date)" >> /home/$USER/heartbeat.log
# this runs every 5 minutes: minute field is */5, everything else is *

crontab -l                              # confirm the entry was saved

# a few more example schedules (add/remove lines as practice, don't leave junk running):
#   0 2 * * *        echo "daily 2am job" >> ~/cron-test.log      # every day at 02:00
#   30 9 * * 1-5      echo "weekday 9:30am" >> ~/cron-test.log     # 09:30, Mon-Fri only
#   0 0 1 * *         echo "first of month" >> ~/cron-test.log     # midnight, 1st of every month
#   @reboot           echo "booted at $(date)" >> ~/cron-test.log  # once, at startup
#   @daily            echo "daily shortcut" >> ~/cron-test.log     # same as "0 0 * * *"

crontab -r                              # remove the whole crontab when done experimenting

# --- system-wide cron ---
cat /etc/crontab                        # see the system crontab that drives cron.daily/weekly/monthly
ls /etc/cron.daily/                     # scripts run once a day, no cron syntax needed
sudo ls /etc/cron.d/                    # system drop-in files; note the extra "user" field per line
# example /etc/cron.d entry (5 time fields + user + command):
#   0 3 * * *   root   /usr/local/bin/nightly-cleanup.sh

# --- at: one-time future jobs ---
echo "echo 'one-time job ran' >> ~/at-test.log" | at now + 2 minutes
atq                                     # list pending at jobs, shows job number and scheduled time
# wait 2 minutes, then:
cat ~/at-test.log                       # confirm it ran

echo "echo 'cancel me' >> ~/at-test.log" | at now + 1 hour
atq                                     # note the job number, e.g. "5"
atrm 5                                  # cancel it before it runs
atq                                     # confirm it's gone

# --- kernel parameters: /proc/sys and sysctl ---
cat /proc/sys/net/ipv4/ip_forward       # read directly from /proc/sys — shows 0 or 1
sysctl net.ipv4.ip_forward              # same value, via sysctl's friendlier interface

sysctl vm.swappiness                    # current swappiness (0-100)
sudo sysctl -w vm.swappiness=10         # change it live — takes effect immediately, no reboot
sysctl vm.swappiness                    # confirm the new value

sysctl fs.file-max                      # system-wide max open file handles

# a live change is lost on reboot unless also written to a persistent config file
grep -r "vm.swappiness" /etc/sysctl.conf /etc/sysctl.d/ 2>/dev/null   # check if already persisted
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf                # persist it (Lesson 4's tee)
sudo sysctl -p                          # reload persistent settings from /etc/sysctl.conf

sysctl -a | head -n 10                  # sample of the hundreds of tunable parameters available
```

## Common Pitfalls

- **Assuming `crontab -e` edits `/etc/crontab`** — it doesn't. It opens (and creates if needed) your own user's private crontab. `/etc/crontab` is a separate, system-wide file you edit directly (with `sudo`) if you need it.
- **Forgetting `sysctl -w` changes don't survive reboot** — a value set with `sysctl -w` or by writing straight to `/proc/sys/` is gone the next time the machine boots. Anything meant to be permanent must also go into `/etc/sysctl.conf` or `/etc/sysctl.d/`.
- **Wrong day-of-week numbering assumption** — cron's day-of-week field runs `0-7`, where both `0` and `7` mean Sunday. Assuming Monday is `0` (like some other systems) silently schedules the job on the wrong day.
- **Cron jobs "not running" because of a missing `$PATH`** — cron runs jobs with a minimal environment, not your interactive shell's `$PATH` or aliases. A script that works fine when you run it manually can fail silently under cron because it can't find a command. Fix by using absolute paths (`/usr/bin/python3` instead of `python3`) inside cron jobs and scripts.
- **Confusing `atq` job numbers with PIDs** — the number `atq` shows is the `at` job's own queue ID, used with `atrm`; it is unrelated to the PID the job will get once it actually runs.
- **Editing `/proc/sys/` files with a text editor** — some editors (like vim, depending on settings) rewrite a file by creating a new one and renaming it over the original, which breaks the special semantics of `/proc/sys/` files. Always write to them with `sysctl -w` or a plain `echo value > /proc/sys/...` redirect, never open them in vim/nano to edit and save.

## FAQ

**Q: What's the difference between cron and at?**
A: cron is for jobs that repeat on a schedule (every day, every 5 minutes, etc.). `at` is for a job that runs exactly once, at a specific future time, and is then gone. Use cron for recurring maintenance tasks, `at` for "do this one thing later."

**Q: Do I need `sudo` to use `crontab -e`?**
A: No — every user has their own crontab and can edit it without `sudo`. You only need `sudo` to edit another user's crontab (`sudo crontab -u otheruser -e`) or the system-wide files under `/etc/`.

**Q: My cron job works when I run it by hand but not under cron — why?**
A: Almost always a `$PATH` or working-directory assumption. cron jobs run with a minimal environment (no interactive shell config loaded), so relative paths and bare command names that rely on your shell's `$PATH` can fail. Use absolute paths for both the interpreter and any files/commands the job touches.

**Q: What happens to a cron or at job if the daemon isn't running?**
A: Nothing runs. Both `cron` and `atd` are systemd services (Lesson 5); if stopped, scheduled jobs simply don't fire until the service is running again — `at` jobs whose time already passed while `atd` was down do not automatically "catch up," they're just missed.

**Q: Is changing a kernel parameter with `sysctl -w` dangerous?**
A: It can be, since it takes effect immediately and system-wide — e.g. flipping `net.ipv4.ip_forward` on a production firewall could suddenly let it route traffic it shouldn't. Always know exactly what a parameter controls before changing it, and prefer testing on a non-critical system first.

**Q: Why does `/proc/sys` exist separately from `sysctl`?**
A: `/proc/sys` is the actual kernel interface — `sysctl` is just a convenience tool built on top of it that adds dotted-name lookup and a `-p`/`-w` workflow. Reading/writing files under `/proc/sys` directly works exactly the same; `sysctl` just saves you from remembering paths.

## Practice / Exercise

**Core:**
1. Add a crontab entry that appends the current date to `~/cron-heartbeat.log` every 2 minutes. Wait for at least two runs, then confirm with `cat` and remove the crontab with `crontab -r`.
2. Write out (on paper or in a text file, don't necessarily install it) the cron expression for "every weekday at 6:45pm" and for "every 10 minutes, only during the first 12 hours of the day."
3. Schedule a one-time `at` job that appends a message to a file 3 minutes from now. Use `atq` to confirm it's queued, then wait and confirm it ran.
4. Schedule a second `at` job for an hour from now, find its job number with `atq`, and cancel it with `atrm` before it runs.
5. Check the current value of `vm.swappiness` and `net.ipv4.ip_forward` using both `cat /proc/sys/...` and `sysctl`, and confirm they return the same value.
6. Temporarily change `vm.swappiness` to `10` with `sysctl -w`, confirm the change, then explain (without necessarily rebooting) why this change would not survive a reboot.

**Stretch:**
1. Look at `/etc/crontab` and explain, in your own words, how it relates to the scripts in `/etc/cron.daily/`.
2. Write an `/etc/cron.d/` style line (5 time fields + user + command) that would run a backup script as the `www-data` user every night at 1am — you don't need root to write this out, just get the syntax right.
3. List 5 more parameters under `/proc/sys/net/ipv4/` besides `ip_forward` using `ls /proc/sys/net/ipv4/`, and look up what any two of them control.

## Further Reading

- `man crontab`, `man 5 crontab` (the file-format man page, different from the command man page), `man at`, `man sysctl`, `man 5 proc`
- [Ubuntu Server Guide: cron](https://ubuntu.com/server/docs)
