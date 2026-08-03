# Lesson 5: Managing System Services with systemd

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-4 (terminal basics, paths, permissions/editors, find/grep/awk)

## Learning Objectives

By end of lesson student can:
- explain what systemd is and what a "unit" is
- control services with `systemctl` (start/stop/restart/reload/enable/disable/status)
- read service logs with `journalctl`, filtered by unit, time range, and priority
- write a custom `.service` unit file from scratch and run it under systemd
- write a `.timer` unit as a systemd-native alternative to cron

## Topics

- systemd concepts: units (.service, .socket, .timer), targets, dependency tree
- systemctl: start, stop, restart, reload, enable, disable, status, is-active, daemon-reload
- journalctl: -u, -f, --since, --until, -n, -p (priority levels)
- Writing a custom .service unit file: [Unit], [Service], [Install] sections
- systemd timers: .timer unit as cron alternative; systemctl list-timers

## Concepts

### What systemd is

**systemd** is the init system on modern Linux distros (including Ubuntu since 15.04) — it's **PID 1**, the very first process the kernel starts at boot, and it's responsible for starting/stopping/supervising every other service and process on the machine. Before systemd, most distros used simpler sequential shell-script-based init systems (SysV init, Upstart) — systemd replaced these with a unified, parallelized, dependency-aware model.

Why it matters for DevOps: virtually every long-running process you'll manage on a Linux server (web servers, databases, your own custom apps) is wrapped as a systemd **unit**, so systemd is the interface you use to start it, stop it, make it survive reboots, and see its logs.

### Units, targets, and the dependency tree

A **unit** is systemd's basic managed object — a single thing systemd knows how to start/stop/monitor. Different unit *types* (identified by file extension) represent different kinds of things:

| Unit type | Represents |
|---|---|
| `.service` | a long-running or one-shot process (the most common type) |
| `.socket` | a network/IPC socket, can lazily start a `.service` when traffic arrives on it (socket activation) |
| `.timer` | a scheduled trigger — starts an associated unit at specified times/intervals |
| `.mount` | a filesystem mount point |
| `.target` | a grouping/synchronization point — not a real process, just a named collection of other units to reach together |

**Targets** are systemd's replacement for old-style "runlevels". Examples: `multi-user.target` (normal system with networking, no GUI — typical for a server), `graphical.target` (adds a display manager, desktop use), `reboot.target`. A target is essentially "get all of these other units running, then consider this milestone reached".

**Dependency tree** — units declare relationships to each other: `Requires=` (hard dependency — if the required unit fails, this one fails too), `Wants=` (soft dependency — try to start it, but don't fail if it doesn't), `After=`/`Before=` (ordering only, not dependency — "if both are starting, do it in this order," without implying one requires the other). systemd uses these declarations to compute a dependency graph and starts as much as possible **in parallel**, only serializing where an explicit ordering/dependency actually exists — this is why systemd boots faster than old sequential init systems.

### systemctl — controlling services

`systemctl` is the main command-line tool for interacting with systemd.

| Command | Effect |
|---|---|
| `systemctl start NAME` | start the unit now |
| `systemctl stop NAME` | stop the unit now |
| `systemctl restart NAME` | stop then start (brief downtime) |
| `systemctl reload NAME` | ask the running process to reload its config **without** restarting (if the service supports it — not all do) |
| `systemctl status NAME` | show current state, recent log lines, main PID, memory usage |
| `systemctl enable NAME` | create the symlinks so the unit starts automatically **at boot** (does NOT start it now) |
| `systemctl disable NAME` | remove those symlinks — stop it from auto-starting at boot (does NOT stop it now if running) |
| `systemctl enable --now NAME` | enable AND start in one command — common combo |
| `systemctl is-active NAME` | quick yes/no: is it currently running? (`active` / `inactive` / `failed`) |
| `systemctl is-enabled NAME` | quick yes/no: will it start at boot? |
| `systemctl daemon-reload` | tell systemd to re-read unit files from disk — **required** after creating/editing any `.service`/`.timer` file, otherwise systemd keeps using its old cached definition |
| `systemctl list-units --type=service` | list all currently loaded service units |
| `systemctl list-timers` | list all timer units and their next/last run time |

Key distinction students always confuse: **start/stop** = right now, this boot only. **enable/disable** = whether it auto-starts on future boots. They're independent — a service can be running-but-not-enabled (won't survive a reboot) or enabled-but-not-running (will start next boot, but isn't active right now).

### journalctl — reading logs

systemd centralizes logs from every unit (plus kernel messages) into a single binary journal, read with `journalctl`.

| Flag | Effect |
|---|---|
| `-u NAME` | logs for a specific unit only |
| `-f` | follow — like `tail -f`, stream new log lines live |
| `--since "TIME"` | only entries after given time, e.g. `--since "1 hour ago"`, `--since "2026-08-01"` |
| `--until "TIME"` | only entries before given time |
| `-n N` | show only the last N lines (like `tail -n`) |
| `-p LEVEL` | filter by minimum priority level |
| `-r` | reverse order — newest first |
| `-b` | only logs since the current boot |
| `-x` | add explanatory help text to messages where available |

Priority levels (syslog standard, from most to least severe): `emerg`(0), `alert`(1), `crit`(2), `err`(3), `warning`(4), `notice`(5), `info`(6), `debug`(7). `-p err` shows `err` and everything more severe (i.e. `err`, `crit`, `alert`, `emerg`), not just exact matches.

```bash
journalctl -u nginx                    # all logs for the nginx unit
journalctl -u nginx -f                  # follow nginx logs live
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -n 50                # last 50 lines
journalctl -p err -b                      # error-or-worse messages from current boot only
```

### Writing a custom .service unit file

Unit files live in `/etc/systemd/system/` (for admin-created units — takes precedence over vendor-supplied units in `/usr/lib/systemd/system/` or `/lib/systemd/system/`). Filename must end in `.service`, e.g. `/etc/systemd/system/myapp.service`.

Basic structure — three sections:

```ini
[Unit]
Description=My sample application
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
User=myappuser
WorkingDirectory=/opt/myapp

[Install]
WantedBy=multi-user.target
```

**`[Unit]` section** — metadata and dependencies:
- `Description=` — human-readable text shown in `systemctl status`
- `After=` / `Before=` / `Requires=`/`Wants=` — ordering and dependency relationships covered above

**`[Service]` section** — how to actually run it:
- `Type=` — `simple` (default; process started by `ExecStart` IS the main process, stays in foreground), `forking` (process forks and the parent exits, common for older-style daemons), `oneshot` (runs once and exits, useful for setup scripts, often paired with a `.timer`)
- `ExecStart=` — the command to run (must be an absolute path)
- `ExecStop=` / `ExecReload=` — optional custom stop/reload commands
- `Restart=` — restart policy: `no` (default), `on-failure` (restart only on non-zero exit), `always`
- `User=` / `Group=` — run as a specific non-root user (security best practice — don't run app services as root unless truly necessary)
- `WorkingDirectory=` — directory the process starts in
- `Environment=KEY=VALUE` — set environment variables for the process

**`[Install]` section** — used only by `enable`/`disable`, defines what target activates this unit:
- `WantedBy=multi-user.target` — most common for standard server services; means "when reaching multi-user.target at boot, also start this"

```bash
# after creating/editing a unit file:
systemctl daemon-reload
systemctl enable --now myapp
systemctl status myapp
journalctl -u myapp -f
```

### System units vs user units

Managing a **system-wide** unit under `/etc/systemd/system/` normally requires root/admin privileges (elevating with `sudo` — full privilege-escalation model is covered in a later lesson). systemd also supports **user units**, which any regular user can create and control entirely on their own, no elevated privileges needed at all:

| | System unit | User unit |
|---|---|---|
| File location | `/etc/systemd/system/` | `~/.config/systemd/user/` |
| Command prefix | `systemctl` | `systemctl --user` |
| Runs as | root (unless `User=` set) | the owning user, tied to their login session |
| `[Install]` target | `multi-user.target` | `default.target` |
| Needs elevated privileges? | yes, to create/enable/start | no |

Today's hands-on lab uses **user units** specifically so every step works without needing elevated access — same directive syntax (`[Unit]`/`[Service]`/`[Install]`), just a different scope and command prefix (`--user`). Real production services (nginx, a database, your deployed app) are almost always system units in practice.

```bash
mkdir -p ~/.config/systemd/user
# create ~/.config/systemd/user/myapp.service here
systemctl --user daemon-reload
systemctl --user enable --now myapp
systemctl --user status myapp
journalctl --user -u myapp -f
```

### systemd timers — a cron alternative

A `.timer` unit triggers another unit (almost always a same-named `.service`) on a schedule, instead of that service running continuously. Two files are needed together: `myjob.service` (what to run — typically `Type=oneshot`) and `myjob.timer` (when to run it).

```ini
# /etc/systemd/system/myjob.service
[Unit]
Description=Nightly cleanup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup.sh
```

```ini
# /etc/systemd/system/myjob.timer
[Unit]
Description=Run myjob nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

`OnCalendar=` accepts systemd's own calendar-event syntax (`*-*-* 02:00:00` = every day at 2 AM; `Mon *-*-* 09:00:00` = every Monday at 9 AM). `OnBootSec=`/`OnUnitActiveSec=` are alternative directives for "N seconds/minutes after boot" or "N seconds after this unit last ran" (interval-style rather than calendar-style). `Persistent=true` means if the machine was off when the timer should have fired, it catches up and runs once as soon as possible after boot — cron has no equivalent built in.

Advantages over classic cron (a full crontab syntax comparison is covered in a later lesson): integrates with `journalctl` for logs automatically, dependency-aware (`After=network.target` etc.), `Persistent=` catch-up behavior, and `systemctl list-timers` gives a clear next-run-time view.

```bash
systemctl enable --now myjob.timer   # note: enable the .timer, NOT the .service
systemctl list-timers                  # shows next/last run time for every timer
```

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `systemctl start NAME` | start now | `systemctl start nginx` |
| `systemctl stop NAME` | stop now | `systemctl stop nginx` |
| `systemctl restart NAME` | stop + start | `systemctl restart nginx` |
| `systemctl reload NAME` | reload config, no restart | `systemctl reload nginx` |
| `systemctl status NAME` | show state + recent logs | `systemctl status nginx` |
| `systemctl enable NAME` | auto-start at boot | `systemctl enable nginx` |
| `systemctl disable NAME` | stop auto-start at boot | `systemctl disable nginx` |
| `systemctl enable --now NAME` | enable + start together | `systemctl enable --now nginx` |
| `systemctl is-active NAME` | running right now? | `systemctl is-active nginx` |
| `systemctl is-enabled NAME` | auto-starts at boot? | `systemctl is-enabled nginx` |
| `systemctl daemon-reload` | reload unit file definitions | `systemctl daemon-reload` |
| `systemctl list-units --type=service` | list loaded services | `systemctl list-units --type=service` |
| `systemctl list-timers` | list timers + next run | `systemctl list-timers` |
| `journalctl -u NAME` | logs for one unit | `journalctl -u nginx` |
| `journalctl -f` | follow logs live | `journalctl -u nginx -f` |
| `journalctl --since/--until` | time-range filter | `journalctl --since "1 hour ago"` |
| `journalctl -n N` | last N lines | `journalctl -n 50` |
| `journalctl -p LEVEL` | filter by priority | `journalctl -p err` |

## Examples / Walkthrough

```bash
# ── exploring existing system services (read-only, no privilege needed) ──
systemctl list-units --type=service
systemctl status ssh                      # common service present on most Ubuntu servers
systemctl is-active ssh
systemctl is-enabled ssh
systemctl status cron                       # cron itself runs as a systemd service (name only, full cron syntax comes later)

# note: start/stop/restart/enable/disable on a SYSTEM unit like ssh/cron
# needs root privileges (sudo) — full privilege-escalation model is covered
# in a later lesson. Everything below uses USER units instead, which any
# account can create and fully control without elevated access.

# ── journalctl basics (read-only, no privilege needed) ────
journalctl -u ssh -n 20
journalctl -u ssh --since "1 hour ago"
journalctl -u ssh -p err
journalctl -u ssh -f          # Ctrl+C to stop following

# ── writing a custom user service ─────────────────────────
mkdir -p ~/lab5 ~/.config/systemd/user
cat << 'SCRIPT' > ~/lab5/hello.sh
#!/bin/bash
while true; do
  echo "hello from myapp at $(date)"
  sleep 5
done
SCRIPT
chmod +x ~/lab5/hello.sh

nano ~/.config/systemd/user/myapp.service
# paste:
# [Unit]
# Description=My sample app
#
# [Service]
# Type=simple
# ExecStart=/home/YOUR_USER/lab5/hello.sh
# Restart=on-failure
#
# [Install]
# WantedBy=default.target

systemctl --user daemon-reload
systemctl --user enable --now myapp
systemctl --user status myapp
systemctl --user is-active myapp
systemctl --user is-enabled myapp
journalctl --user -u myapp -f            # watch it print "hello from myapp" every 5s, Ctrl+C to stop

# enable/start vs disable/stop distinction, same as system units
systemctl --user disable myapp
systemctl --user is-active myapp          # still active! disable didn't stop it
systemctl --user is-enabled myapp          # now disabled — won't start on next login though
systemctl --user stop myapp
systemctl --user is-active myapp            # now inactive
systemctl --user enable --now myapp          # restore for next steps

systemctl --user stop myapp
systemctl --user disable myapp

# ── writing a user timer ──────────────────────────────────
cat << 'SCRIPT' > ~/lab5/oneshot.sh
#!/bin/bash
echo "oneshot ran at $(date)" >> ~/lab5/oneshot.log
SCRIPT
chmod +x ~/lab5/oneshot.sh

nano ~/.config/systemd/user/myjob.service
# [Unit]
# Description=Oneshot demo job
#
# [Service]
# Type=oneshot
# ExecStart=/home/YOUR_USER/lab5/oneshot.sh

nano ~/.config/systemd/user/myjob.timer
# [Unit]
# Description=Run myjob every 2 minutes for demo
#
# [Timer]
# OnBootSec=1min
# OnUnitActiveSec=2min
#
# [Install]
# WantedBy=timers.target

systemctl --user daemon-reload
systemctl --user enable --now myjob.timer
systemctl --user list-timers
# wait a couple minutes...
cat ~/lab5/oneshot.log

# ── cleanup ───────────────────────────────────────────────
systemctl --user disable --now myjob.timer
rm ~/.config/systemd/user/myapp.service ~/.config/systemd/user/myjob.service ~/.config/systemd/user/myjob.timer
systemctl --user daemon-reload
rm -r ~/lab5
```

## Common Pitfalls

- **Forgetting `daemon-reload` after editing a unit file** — systemd caches unit definitions in memory; editing the `.service` file on disk has NO effect until you run `systemctl daemon-reload`. Symptom: your changes seem to be "ignored".
- **Confusing enable/disable with start/stop** — `enable` alone doesn't start anything right now, and `disable` alone doesn't stop a currently running service. Four independent states exist: running+enabled, running+disabled, stopped+enabled, stopped+disabled.
- **`ExecStart=` using a relative path** — systemd requires an **absolute path** for `ExecStart`; a relative path or a bare command name not in a resolvable location fails silently with a cryptic error in the journal.
- **Forgetting the script needs execute permission** — `ExecStart=/path/to/script.sh` fails with permission denied if `chmod +x` wasn't applied first (ties back to Lesson 3).
- **Editing the wrong unit file location** — vendor-shipped units live under `/usr/lib/systemd/system/` or `/lib/systemd/system/`; custom units belong in `/etc/systemd/system/` (which takes precedence). Editing vendor files directly gets overwritten on package upgrades.
- **Enabling the `.service` instead of the `.timer`** — for timer-driven jobs, you `enable`/`start` the `.timer` unit, NOT the `.service` — the service only runs when triggered by its timer (or manually for testing with `systemctl start myjob.service`).
- **`reload` assuming it always works** — not every service implements a graceful reload; some just don't support it and the command has no effect (or errors) — check the specific service's documentation/behavior.
- **Not checking `systemctl status`/`journalctl` after a failed start** — a service that fails to start often gives almost no terminal feedback beyond "Job for X failed"; `systemctl status NAME` and `journalctl -u NAME -n 50` are the actual diagnostic tools, always check both when something won't start.
- **Typo'd `[Section]` header names** — sections are case-sensitive and must be exactly `[Unit]`, `[Service]`, `[Install]`, `[Timer]` — a typo silently drops that whole section's directives rather than erroring clearly.

## FAQ

**Q: What's the difference between `restart` and `reload`?**
A: `restart` fully stops the process and starts a new one — brief downtime, guaranteed fresh state, always works. `reload` asks the still-running process to re-read its config in place (if it supports that) — no downtime, but only works if the specific service implements reload logic (many do, e.g. nginx; some don't).

**Q: If I run `systemctl enable myapp`, does that start it right now?**
A: No. `enable` only sets up the boot-time auto-start symlinks — the service stays in whatever state (running or not) it was already in. Use `systemctl enable --now myapp` to do both at once, or `systemctl start` separately.

**Q: Why isn't my edited `.service` file taking effect?**
A: You almost certainly need `systemctl daemon-reload` after any edit to a unit file — systemd doesn't watch unit files for live changes, it caches its parsed definitions until told to re-read them.

**Q: Where should I put my own custom unit files?**
A: `/etc/systemd/system/` — this location takes priority over vendor-supplied units and is the conventional place for admin/custom-created units. Never edit files under `/usr/lib/systemd/system/` directly (package manager owns those, changes get lost on updates).

**Q: What does `Type=simple` vs `Type=oneshot` vs `Type=forking` actually control?**
A: It tells systemd how to know the service has "started successfully" and what to consider the main process. `simple` (default): the `ExecStart` process itself IS the service, running in the foreground — systemd considers it started as soon as it execs. `oneshot`: the command is expected to run to completion and exit — used for one-time setup/jobs rather than long-running daemons, often combined with a `.timer`. `forking`: the started process forks a child and the parent exits — systemd needs to track the forked child as the "real" process (older daemon style, less common today).

**Q: How do I know if a service failed to start and why?**
A: `systemctl status NAME` shows the current/last state plus a handful of recent log lines directly. For more detail, `journalctl -u NAME -n 50` (or add `-p err`) shows fuller error output from the actual process.

**Q: Why enable/start the `.timer` and not the `.service` for a scheduled job?**
A: The `.timer` unit is what's scheduled — it triggers its matching `.service` at the configured times. The `.service` itself, if `Type=oneshot`, isn't meant to run continuously; enabling it directly wouldn't do anything useful since nothing is scheduling it. (You CAN manually `systemctl start myjob.service` any time to test it runs correctly, independent of the timer.)

**Q: What's the practical benefit of a systemd timer over a plain cron job?**
A: Automatic integration with `journalctl` (logs alongside every other service, no separate log-parsing needed), the `Persistent=true` catch-up behavior for missed runs (cron has nothing built-in for this), dependency awareness (`After=network.target`, etc.), and a single consistent `systemctl`/`journalctl` toolchain instead of a separate cron-specific mental model. Trade-off: systemd timer syntax is arguably more verbose to write than a one-line crontab entry for very simple schedules.

**Q: What does `Persistent=true` do on a timer?**
A: If the system was powered off (or the timer's service was otherwise unable to run) at its scheduled time, `Persistent=true` makes systemd run it once as soon as possible after the system comes back up, rather than just waiting silently for the next scheduled occurrence.

## Practice / Exercise

**Core (everyone should finish):**
1. Pick any service already running on the system (e.g. `ssh`), run `status`, `is-active`, and `is-enabled` against it (read-only, no privileges needed), and explain what each output means.
2. Write a simple shell script that loops and prints a timestamp every few seconds; wrap it in a custom **user** `.service` unit (`~/.config/systemd/user/`, `Type=simple`); `systemctl --user daemon-reload`, `enable --now`, confirm it's running with `status`/`is-active`.
3. Stop that user service, confirm with `is-active`, then start it again.
4. Disable it, confirm with `is-enabled` that it won't auto-start at next login, then re-enable it, without ever stopping it — observe it stayed running the whole time.
5. View its last 20 log lines with `journalctl --user -u NAME -n 20`, and watch it live with `-f`.
6. Filter its logs to only the last hour with `--since`.
7. Stop and disable your custom service, then delete its unit file and `daemon-reload` again to clean up.

**Stretch (fast finishers):**
8. Write a `oneshot` **user** service that appends a timestamp to a log file, paired with a `.timer` set to run every 2 minutes using `OnBootSec=`/`OnUnitActiveSec=`; verify it fires with `systemctl --user list-timers` and by checking the log file grows.
9. Add `Restart=on-failure` to your custom service, then deliberately make the script exit with a non-zero code — observe systemd automatically restart it (check `systemctl --user status` for restart count).
10. Investigate what happens if you `enable` (not `enable --now`) a brand-new user service without ever starting it — describe the state using `is-active` and `is-enabled` together.
11. Look up (read-only, via `systemctl status` or `journalctl -u`) what `Description=` and `After=` are set to for a real installed system service like `ssh` or `cron`, and explain why that specific `After=` ordering makes sense.

## Further Reading

- `man systemd.unit` — full unit file directive reference (applies to all unit types)
- `man systemd.service` — service-specific directives
- `man systemd.timer` — timer-specific directives, including full `OnCalendar=` syntax
- `man systemd.time` — calendar/time span syntax reference used by timers
- `man journalctl` — full journalctl flag reference
