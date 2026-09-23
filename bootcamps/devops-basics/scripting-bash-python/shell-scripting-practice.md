# Lesson 18: Shell Scripting Practice

**Module:** Scripting (Bash & Python)
**Duration:** 120-150 min
**Prerequisites:** Lesson 16 (Basics & Syntax), Lesson 17 (Loops & Functions) — this lesson applies both directly to a real project

## Learning Objectives

By end of lesson student can:
- Read and explain the architecture of a small, real, multi-file Bash application
- Build a monitoring script from scratch, function by function, following a repeatable build order
- Use `source` to split a script across multiple files, `sed -i` for in-place config editing, heredocs for generating a file, and `$EUID` for a root check
- Package a long-running script as a systemd service with an install/uninstall pair
- Identify and fix real bugs found in an existing, unpolished script

## Topics

- Project walkthrough: `monitoring-app` — a CPU/disk monitor that alerts to Telegram, packaged as a systemd service
- New syntax needed for this build: `source`, heredoc (`<<EOF`), `sed -i`, `$EUID`, `bc` for non-integer arithmetic
- Build order (cheat sheet): config → check functions → main loop with a state file → install/uninstall + systemd unit
- Reading real code critically: spotting bugs, missing quoting, missing `set -euo pipefail`, in an unpolished real repo
- Homework: 4 independent take-home scripts (technical task briefs, not built in class)

## Concepts

### Project overview: what `monitoring-app` does

`monitoring-app` (a small real repo used as this lesson's build target) is a lightweight, dependency-free CPU and disk usage monitor for a Linux server. It runs continuously in the background as a systemd service, checks CPU load and disk usage on a fixed interval, and sends a Telegram message when a threshold is crossed — and another message when things return to normal. No database, no web dashboard — just a loop, two check functions, and an HTTP call to Telegram's Bot API.

Its file layout mirrors a pattern worth learning on its own — splitting a script by responsibility instead of one giant file:

| File | Responsibility |
|---|---|
| `monitoring.conf` | All configuration in one place: hostname, Telegram credentials, thresholds, check interval |
| `functions/cpu.sh` | One function: check CPU load, alert/resolve on threshold crossing |
| `functions/storage.sh` | One function: check disk usage per mountpoint, alert/resolve on threshold crossing |
| `start.sh` | The main loop: loads config + functions, runs both checks every `INTERVAL` seconds |
| `install.sh` | Interactive setup: fills in missing config, generates and enables a systemd unit |
| `uninstall.sh` | Stops and removes the systemd unit |

This "one file per concern, `source`d together" pattern scales far better than a single 300-line script once a project grows past a couple of checks.

### New syntax this build introduces

A few pieces of syntax show up in this real project that haven't been needed yet:

| Syntax | What it does |
|---|---|
| `source file` (or `. file`) | Runs another script's content *in the current shell*, not a subshell — any variables/functions it defines become available here. This is how `start.sh` gets `monitoring.conf`'s variables and `cpu.sh`/`storage.sh`'s functions without copy-pasting them in |
| `$EUID` | The *effective* user ID of whoever is running the script (0 = root) — checking `[[ $EUID -ne 0 ]]` is the standard way to require root before an install script proceeds |
| `cat > file <<EOF ... EOF` | A **heredoc**: everything between `<<EOF` and the closing `EOF` is fed as-is into the redirected command — the standard way to generate a multi-line file (like a systemd unit) from inside a script without a long chain of `echo`/`>>` lines |
| `sed 's/pattern/replacement/' -i file` | In-place find-and-replace on a file — used here to fill a blank `HOSTNAME=` line in the config file with a value the user typed at the install prompt |
| `bc` | An external command-line calculator — needed because Bash's own arithmetic (`$(( ))`) is integer-only and can't handle a load average like `0.52` directly |

### Building block by block

**1. The config file (`monitoring.conf`)** is a plain list of `KEY=value` lines. Because `start.sh` `source`s it directly, it's not really a separate "config format" — it's literally Bash variable assignments, loaded straight into the script's own environment. This is a common, simple pattern for small tools: no parsing library needed, at the cost of the config file technically being executable shell code (worth being aware of, if the file's contents were ever untrusted — not a concern here, since only the admin installing the tool edits it).

**2. Each check function** follows the same shape: read the current value (CPU load via `uptime`, disk usage via `df`), compare it against a configured limit, and track a small piece of *state* (a variable like `cpu_in_alarm`) so it only sends an alert once when crossing the threshold — not on every single loop iteration — and sends a second "resolved" message once the value drops back down. This alarm/resolved state-tracking pattern is the core idea behind almost any alerting logic, at any scale.

**3. The main loop (`start.sh`)** is deliberately simple: create a "running" marker file, then loop calling both check functions and sleeping `INTERVAL` seconds, for as long as that marker file still exists. This is a lightweight way to make the loop stoppable from outside — `uninstall.sh`'s systemd `ExecStop` just deletes the marker file, and the loop notices on its next `while` condition check and exits naturally, without needing a signal handler.

**4. `install.sh`** checks for root (`$EUID`), prompts for any missing config values with `read -p` in a `while [[ -z ... ]]` loop (so it keeps asking until a non-empty answer is given — straight out of Lesson 16/17), writes them into `monitoring.conf` with `sed -i`, then generates a systemd `.service` unit file with a heredoc and installs it — tying together Lesson 5 (systemd) with this lesson's new syntax.

### Reading real code critically

This repository, like most real-world scripts, has a few rough edges worth spotting deliberately as a learning exercise rather than treating the code as a flawless reference — this is a normal, healthy way to read any codebase you didn't write yourself.

## Commands / Syntax Reference

| Syntax | Purpose | Example |
|---|---|---|
| `source file` | Load variables/functions from another file into the current shell | `source ./monitoring.conf` |
| `$EUID` | Effective user ID of the current process | `[[ $EUID -ne 0 ]]` |
| `cat > file <<EOF` | Heredoc: write a multi-line block to a file | see walkthrough |
| `sed 's/a/b/' -i file` | In-place substitution | `sed 's/^X=/X=1/' -i file.conf` |
| `bc` | Command-line calculator (handles decimals) | `echo "10/3" \| bc` |
| `readlink -f "$0"` | Resolve the script's own absolute path | `dirname $(readlink -f "$0")` |

## Examples / Walkthrough

### Cheat sheet: build order

Follow this order in class — each step is runnable/testable on its own before moving to the next.

```bash
# --- step 1: project skeleton ---
mkdir -p monitoring-app/functions
cd monitoring-app
touch monitoring.conf start.sh install.sh uninstall.sh functions/cpu.sh functions/storage.sh
chmod +x start.sh install.sh uninstall.sh
```

```bash
# --- step 2: monitoring.conf — all config in one place, nothing else ---
cat > monitoring.conf <<'EOF'
HOSTNAME=
BOT_TOKEN=
CHAT_ID=

INTERVAL=7

CPU_LIMIT_PEAK_COUNT=3
CPU_LIMIT=20

MOUNTPOINTS="/"
STORAGE_LIMIT=30

UNIT_NAME=devops-monitor
EOF
```

```bash
# --- step 3: functions/cpu.sh — one function, one job ---
cat > functions/cpu.sh <<'EOF'
#!/bin/bash
cpu_in_alarm=0
cpu_peak_count=0

send_alert() {
    local text="$1"
    curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d chat_id="${CHAT_ID}" -d text="${text}" > /dev/null
}

cpu_check() {
    local load
    load=$(uptime | awk -F'load average: ' '{ print $2 }' | awk -F', ' '{ print $1 }')
    local load_pct
    load_pct=$(echo "$load * 100 / $(nproc)" | bc)     # bc: handles the decimal load average

    echo "$(date): CPU load is ${load_pct}%"

    if [[ $load_pct -ge $CPU_LIMIT && $cpu_in_alarm -eq 0 ]]; then
        ((cpu_peak_count++))
        if [[ $cpu_peak_count -ge $CPU_LIMIT_PEAK_COUNT ]]; then
            send_alert "ALERT: ${HOSTNAME} CPU load is ${load_pct}%"
            cpu_in_alarm=1
            cpu_peak_count=0
        fi
    elif [[ $load_pct -lt $CPU_LIMIT && $cpu_in_alarm -eq 1 ]]; then
        send_alert "RESOLVED: ${HOSTNAME} CPU load back to ${load_pct}%"
        cpu_in_alarm=0
    fi
}
EOF
```

```bash
# --- step 4: functions/storage.sh — same pattern, applied to disk usage ---
cat > functions/storage.sh <<'EOF'
#!/bin/bash
declare -A storage_alarm_state    # associative array: one alarm flag PER mountpoint

storage_check() {
    local mnt used
    for mnt in $MOUNTPOINTS; do
        used=$(df -h "$mnt" | awk 'NR==2 { print $5 }' | tr -d '%')

        if [[ $used -ge $STORAGE_LIMIT && ${storage_alarm_state[$mnt]:-0} -eq 0 ]]; then
            send_alert "ALERT: ${HOSTNAME} disk ${mnt} is ${used}% full"
            storage_alarm_state[$mnt]=1
        elif [[ $used -lt $STORAGE_LIMIT && ${storage_alarm_state[$mnt]:-0} -eq 1 ]]; then
            send_alert "RESOLVED: ${HOSTNAME} disk ${mnt} back to ${used}%"
            storage_alarm_state[$mnt]=0
        fi
    done
}
EOF
```

```bash
# --- step 5: start.sh — the main loop, ties everything together ---
cat > start.sh <<'EOF'
#!/bin/bash
set -uo pipefail    # -e deliberately omitted: one failed check shouldn't kill the whole loop

script_dir=$(dirname "$(readlink -f "$0")")
running_flag="${script_dir}/running"

source "${script_dir}/monitoring.conf"
source "${script_dir}/functions/cpu.sh"
source "${script_dir}/functions/storage.sh"

touch "$running_flag"
trap 'rm -f "$running_flag"' EXIT    # Lesson 17: guarantee the flag is cleaned up on any exit

while [[ -f "$running_flag" ]]; do
    cpu_check
    storage_check
    sleep "$INTERVAL"
done
EOF
chmod +x start.sh

# quick manual test, without installing as a service yet:
# fill in monitoring.conf's HOSTNAME/BOT_TOKEN/CHAT_ID by hand first, then:
# ./start.sh &
# tail the output, confirm both checks print a line every $INTERVAL seconds
# rm running   -> confirm the loop stops itself cleanly
```

```bash
# --- step 6: install.sh — root check, prompt for missing config, generate + enable systemd unit ---
cat > install.sh <<'EOF'
#!/bin/bash
set -euo pipefail

if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root (use sudo)." >&2
    exit 1
fi

work_dir=$(dirname "$(readlink -f "$0")")
source "${work_dir}/monitoring.conf"

prompt_if_missing() {
    local var_name="$1" prompt_text="$2" value=""
    if [[ -z "${!var_name}" ]]; then          # ${!var_name}: indirect reference, reads the var NAMED by $var_name
        while [[ -z "$value" ]]; do
            read -p "$prompt_text: " value
        done
        sed -i "s|^${var_name}=.*|${var_name}=\"${value}\"|" "${work_dir}/monitoring.conf"
    fi
}

prompt_if_missing HOSTNAME "Hostname"
prompt_if_missing BOT_TOKEN "Telegram bot token"
prompt_if_missing CHAT_ID "Telegram chat ID"

source "${work_dir}/monitoring.conf"          # reload: pick up whatever was just written

cat > "/etc/systemd/system/${UNIT_NAME}.service" <<UNIT
[Unit]
Description=Monitoring service (${UNIT_NAME})
After=network.target

[Service]
Type=simple
ExecStart=${work_dir}/start.sh
ExecStop=/usr/bin/rm -f ${work_dir}/running

[Install]
WantedBy=multi-user.target
UNIT

systemctl daemon-reload
systemctl enable --now "${UNIT_NAME}.service"
echo "Installed and started ${UNIT_NAME}.service"
EOF
chmod +x install.sh
```

```bash
# --- step 7: uninstall.sh — mirror of install.sh ---
cat > uninstall.sh <<'EOF'
#!/bin/bash
set -euo pipefail

if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root (use sudo)." >&2
    exit 1
fi

work_dir=$(dirname "$(readlink -f "$0")")
source "${work_dir}/monitoring.conf"

systemctl stop "${UNIT_NAME}.service" || true
systemctl disable "${UNIT_NAME}.service" || true
rm -f "/etc/systemd/system/${UNIT_NAME}.service"
systemctl daemon-reload
echo "Uninstalled ${UNIT_NAME}.service"
EOF
chmod +x uninstall.sh
```

```bash
# --- step 8: install and verify, end to end ---
sudo ./install.sh                        # prompts for HOSTNAME/BOT_TOKEN/CHAT_ID, installs the systemd unit
sudo systemctl status devops-monitor      # confirm it's active and running
sudo journalctl -u devops-monitor -f      # watch live CPU/disk check output (Lesson 5)

sudo ./uninstall.sh                        # tear it down when done
```

## Common Pitfalls

*(a mix of general Bash mistakes, and specific bugs found by reading the real `monitoring-app` repo critically)*

- **Real bug found in the repo — `$date` vs `$dt`** — `cpu.sh` in the original repo assigns `dt=$(date)` but then prints `echo "$date | ..."` — `$date` is a different, never-set variable, so it silently prints an empty value instead of the timestamp. A great example of why consistent naming matters, and why `set -u` (Lesson 17) would have caught this immediately by erroring on the undefined `$date` instead of silently printing nothing.
- **Real bug found in the repo — unquoted `$(dirname $(readlink -f "$0"))`** — the inner `readlink -f "$0"` is quoted correctly, but the outer `dirname $(...)` is not; if the script's path ever contained a space, this would break. Always quote command substitutions: `"$(dirname "$(readlink -f "$0")")"`.
- **Real bug found in the repo — no `set -e`/`set -u` anywhere** — none of the original scripts use strict mode (Lesson 17), so a missing `curl`, an unset variable typo, or a failed `sed` all fail silently and the script carries on regardless. Adding `set -euo pipefail` (with `-e` deliberately relaxed in the long-running loop, as shown above) catches entire classes of bugs like the `$date` one above automatically.
- **Forgetting `local` in `send_alert`/helper functions** — a variable like `text` or `value` not marked `local` inside a helper function can silently overwrite a same-named variable elsewhere in the script; every function in this build deliberately uses `local`.
- **Using `[[ ... ]]` numeric comparison on a value that might contain a decimal** — `[[ $load_pct -ge $CPU_LIMIT ]]` requires `load_pct` to be a plain integer; this is exactly why `bc` is used to compute a percentage rather than trying to compare `0.52` directly with `-ge`, which would error out.
- **`sed -i` with an unescaped `/` in the replacement value** — if a config value itself could contain a `/` (a token, a path), the classic `sed 's/pattern/replacement/'` breaks because `/` is also the delimiter. Using `sed -i "s|pattern|replacement|"` (with `|` as the delimiter instead) sidesteps this, as used in `install.sh` above.

## FAQ

**Q: Why does `start.sh` use a "running" file instead of just running forever until killed?**
A: It gives `uninstall.sh`/systemd a clean way to ask the loop to stop on its own terms (finish the current iteration, then exit) rather than forcibly killing it mid-check — `ExecStop` just deletes the file, and the `while [[ -f ... ]]` condition naturally becomes false on the next check.

**Q: Why `source` the functions instead of just putting everything in one file?**
A: Splitting by responsibility (config, one file per check, the loop that ties them together) makes each piece independently readable and testable, and means adding a third check later (e.g. memory usage) is just "add `functions/memory.sh`, `source` it, call its function in the loop" without touching the existing files.

**Q: Why compute CPU load with `bc` instead of plain Bash arithmetic?**
A: Bash's built-in `$(( ))` arithmetic only handles integers — `uptime`'s load average is a decimal (e.g. `0.52`), and dividing that further by core count needs real division `bc` can do that Bash's integer arithmetic can't.

**Q: What does `${!var_name}` mean in `prompt_if_missing`?**
A: That's indirect variable reference — it reads the value of *whatever variable is named by* `$var_name`, not `$var_name` itself. It's what lets one generic function prompt for `HOSTNAME`, `BOT_TOKEN`, or `CHAT_ID` without three near-identical copies of the same prompting logic.

**Q: Is it safe that `monitoring.conf` is just sourced as executable shell code?**
A: For this project, yes — it's a file only the machine's admin edits, on their own machine, so there's no different trust boundary being crossed. It's worth knowing as a general pattern that `source`-based config is only appropriate when the config file itself is as trusted as the script running it; a config file taking input from an untrusted source would need a real parser instead.

## Practice / Exercise

**Core (build during class, following the cheat sheet above):**
1. Build the project skeleton and `monitoring.conf`.
2. Write `functions/cpu.sh`, test `cpu_check` in isolation by sourcing the config and function file directly in an interactive shell and calling it manually.
3. Write `functions/storage.sh` the same way, and test it in isolation too.
4. Write `start.sh`, run it manually in the foreground (not yet as a service), confirm both checks run every `INTERVAL` seconds, and confirm `rm running` stops it cleanly.
5. Write `install.sh` and `uninstall.sh`, then install the whole thing as a real systemd service on a lab VM, confirm it's `active (running)`, and watch its output with `journalctl -u <unit> -f`.
6. Deliberately trigger an alert (e.g. temporarily lower `CPU_LIMIT` to something your machine's idle load already exceeds) and confirm a Telegram message arrives; confirm a second "resolved" message arrives once you raise the limit back and the alarm clears.

**Stretch:**
1. Fix the two real bugs called out in Common Pitfalls (`$date`/`$dt`, the unquoted `dirname $(...)`) in your own copy, and add `set -euo pipefail` throughout.
2. Add a third check function (e.g. memory usage via `free`) following the exact same file-per-concern pattern, and wire it into `start.sh`.
3. Compare your build against the actual `monitoring-app` repository this lesson is based on, and list every difference you introduced and why.

## Homework — Take-Home Scripts

These four scripts are **not** built in class — write them independently before the next lesson, applying everything from Lessons 16-17 (and this lesson's `source`/heredoc/`sed` where useful).

**1. System info report**
Write a script that prints a clean, readable report covering: hostname, uptime, current disk usage (`df -h`), current memory usage (`free -h`), and the top 5 processes by CPU usage (Lesson 8's `ps`). Accept an optional `-o <file>` flag to write the report to a file instead of the terminal (hint: positional-parameter/flag parsing, covered conceptually in Lesson 16 — look up `getopts` if you want to go further than a simple `if [[ "$1" == "-o" ]]` check).

**2. Log file analyzer**
Write a script that takes a log file path as its one required argument, and reports: total line count, count of lines containing "error" (case-insensitive), and the 5 most common error messages (or lines) in the file. Use `grep`/`awk` (Lesson 4) for the filtering, and a `while` loop or `sort | uniq -c` style counting for the summary. Exit with a non-zero code and a usage message if no file argument is given, or the file doesn't exist.

**3. Automated backup script**
Write a script that takes a source directory as an argument, creates a `tar.gz` archive of it named with the current date (e.g. `backup-2026-09-23.tar.gz`), and stores it in a configurable backup directory. Then implement rotation: after creating a new backup, delete any backup files older than N days (configurable), so the backup directory doesn't grow unbounded. Use functions to separate "create backup" from "rotate old backups."

**4. Bulk user creation from CSV**
Write a script that reads a CSV file (`username,fullname` per line) and creates a local Linux user for each row that doesn't already exist (Lesson 6's `useradd`, `id` to check existence first). The script must require root (`$EUID` check, from this lesson), skip and warn about malformed lines instead of crashing, and print a summary at the end: how many users were created, how many already existed and were skipped.

## Further Reading

- [monitoring-app repository](https://github.com/zafarsaidov/monitoring-app) — this lesson's build target
- `man bash` (search "Here Documents", "Parameter Expansion" for `${!var}`)
- `man sed`, `man systemd.unit`
