# Lesson 6: User and Group Management

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-5 (terminal basics, permissions, find/grep/awk, systemd)

## Learning Objectives

By end of lesson student can:
- explain the structure of `/etc/passwd`, `/etc/shadow`, `/etc/group` and read any field from them
- create, modify, and delete users with `useradd`/`usermod`/`userdel`
- create, modify, and delete groups, and manage a user's secondary group memberships
- manage password aging policy with `passwd`/`chage`
- grant privileged access safely via `sudo`/`/etc/sudoers`/`visudo`, including scoped `NOPASSWD` rules
- explain and use `newgrp`, `id`, and `groups` to inspect/change effective group context

## Topics

- /etc/passwd, /etc/shadow, /etc/group: file structure and fields
- useradd (-m, -s, -G, -d), usermod (-aG, -s, -l), userdel (-r)
- groupadd, groupmod, groupdel; managing secondary groups
- passwd, chage: password aging, expiry policies, -l (list)
- sudo: /etc/sudoers, visudo, NOPASSWD, /etc/sudoers.d/ drop-in files
- newgrp: temporary login to a group without re-login; id, groups commands

## Concepts

### /etc/passwd — user account database

Despite the name, `/etc/passwd` no longer stores actual passwords (that moved to `/etc/shadow` decades ago for security) — it stores account metadata. World-readable, one line per user, 7 colon-separated fields:

```
alice:x:1001:1001:Alice Smith,,,:/home/alice:/bin/bash
│     │ │    │    │              │           │
│     │ │    │    │              │           └─ login shell
│     │ │    │    │              └─ home directory
│     │ │    │    └─ GECOS field: full name / contact info (comma-separated, mostly unused today beyond full name)
│     │ │    └─ primary GID (group ID)
│     │ └─ UID (user ID)
│     └─ password placeholder — always "x", meaning "real hash lives in /etc/shadow"
└─ username
```

Key UID ranges to know: `0` = root (always). System/service accounts typically `1`-`999` (or up to 999 on Debian/Ubuntu — varies slightly by distro convention). Regular human user accounts typically start at `1000` on Ubuntu. This is exactly why Lesson 4's exercise used `awk -F: '$3 >= 1000'` to isolate "real" human accounts from system accounts.

### /etc/shadow — password hashes and aging

Root-readable only (`600` or `640` permissions) — contains the actual password hash plus aging metadata, 9 colon-separated fields:

```
alice:$6$abc123...:19500:0:90:7:::
│     │            │     │  │  │ │└─ expiration date (days since epoch), usually empty = never
│     │            │     │  │  │└─ inactive: days after expiry before account is fully disabled
│     │            │     │  │└─ warn: days before expiry to start warning user
│     │            │     │└─ max: max days password is valid before must change
│     │            │└─ min: min days before password can be changed again
│     │            └─ last changed (days since Jan 1 1970 epoch)
│     └─ password hash (`$6$` = SHA-512; `!` or `*` = account locked/no password login)
└─ username
```

Why the separation from `/etc/passwd`: `/etc/passwd` needs to be world-readable (many programs need to look up usernames/UIDs), but password hashes must NOT be world-readable — splitting them into `/etc/shadow` with restrictive permissions solves both needs at once.

### /etc/group — group database

World-readable, one line per group, 4 colon-separated fields:

```
devs:x:1002:alice,bob
│    │ │    └─ member usernames, comma-separated (secondary/supplementary members only)
│    │ └─ GID (group ID)
│    └─ password placeholder (group passwords are essentially unused/legacy today)
└─ group name
```

Important distinction: a user's **primary group** (set in `/etc/passwd` field 4) does NOT need to list them in `/etc/group`'s member list — only **secondary/supplementary** group memberships show up there. This trips people up when grepping `/etc/group` and not finding a user who's clearly "in" a group as their primary one.

### useradd — creating users

```bash
useradd [options] username
```

| Flag | Effect |
|---|---|
| `-m` | create the home directory (copying skeleton files from `/etc/skel`) — **without `-m`, no home directory is created**, a very common beginner mistake |
| `-s /bin/bash` | set login shell (default varies by distro config, sometimes `/bin/sh` or no shell at all) |
| `-G group1,group2` | set **secondary** groups at creation time (comma-separated, no spaces) |
| `-g group` | set **primary** group (must already exist) |
| `-d /path` | custom home directory path (instead of default `/home/username`) |
| `-c "Full Name"` | GECOS comment field (full name) |
| `-u UID` | force a specific UID instead of next-available |

```bash
useradd -m -s /bin/bash -G devs,docker -c "Alice Smith" alice
```

By default (no `-g`), most distros create a new **private group** matching the username (UPG — User Private Group scheme), so `alice`'s primary group is a group also called `alice`, not a shared `users` group — worth explaining since it surprises people expecting one shared "users" group for everyone.

### usermod — modifying existing users

| Flag | Effect |
|---|---|
| `-aG group1,group2` | **append** to secondary groups (adds without removing existing ones) |
| `-G group1,group2` | **replace** secondary groups entirely (removes any not listed — dangerous without `-a`!) |
| `-s /bin/zsh` | change login shell |
| `-l newname` | change login name (rename account; does NOT rename home directory automatically) |
| `-d /new/path -m` | change home directory, `-m` moves existing contents there |
| `-L` / `-U` | lock / unlock the account (prepends/removes `!` in shadow password field) |

```bash
usermod -aG docker alice        # add alice to docker group, keep her existing groups
usermod -s /bin/zsh alice         # change her shell
usermod -l alicia alice             # rename login alice -> alicia
```

**Critical gotcha to emphasize:** `usermod -G docker alice` (without `-a`) **replaces** all of alice's secondary groups with just `docker` — if she was also in `devs` and `sudo`, those memberships are silently removed. Always use `-aG` when the intent is "add one more group", never bare `-G`.

### userdel — deleting users

```bash
userdel username          # removes account entries from passwd/shadow/group, leaves home directory
userdel -r username         # also removes home directory and mail spool
```

`-r` is destructive — the user's files under `/home/username` are permanently deleted. Good practice: archive/backup a user's home directory before running `userdel -r` in any real environment.

### groupadd, groupmod, groupdel

```bash
groupadd devs                   # create new group
groupadd -g 3000 devs             # create with specific GID
groupmod -n newname oldname         # rename a group
groupdel devs                         # delete a group (fails if it's still someone's PRIMARY group)
```

### Managing secondary groups

A user can belong to exactly one **primary** group but many **secondary** groups simultaneously. Secondary groups are how you grant access to shared resources (e.g. the `docker` group lets a user run Docker commands without being root; a `devs` group might own shared project files).

```bash
usermod -aG groupname username     # add user to an additional group
gpasswd -d username groupname        # remove user from a secondary group (alternative to editing usermod -G directly)
groups username                        # list all groups a user belongs to
id username                              # list UID, GID, and all group memberships with names+numbers
```

### passwd and chage — password aging

`passwd` — change a password (interactively) or manage account password state:

```bash
passwd username             # (as root/sudo) set/change another user's password
passwd                       # change your OWN password
passwd -l username             # lock account (prevents password login, prepends ! to hash)
passwd -u username               # unlock
passwd -e username                 # expire immediately — force change at next login
passwd -S username                   # show short status: locked/unlocked, last change date, aging info
```

`chage` — fine-grained password aging control (reads/writes the same `/etc/shadow` fields described above):

```bash
chage -l username                # list current aging settings for a user (readable summary)
chage -M 90 username               # max days password valid (force change every 90 days)
chage -m 7 username                  # min days before password can be changed again
chage -W 7 username                    # warn 7 days before expiration
chage -E 2026-12-31 username             # set an explicit account expiration date
chage -d 0 username                        # force password change at NEXT login (sets last-changed to epoch)
```

### sudo — controlled privilege escalation

`sudo command` runs a single command as another user (root by default), after the calling user authenticates with **their own** password (not root's) — this is the entire point: users get scoped, audited, logged privileged access without ever knowing the actual root password.

**`/etc/sudoers`** — the config file defining who can run what as whom. **Never edit it directly with a normal editor** — always use **`visudo`**, which locks the file and validates syntax before saving, preventing a broken sudoers file from locking every admin out of `sudo` entirely (a syntax error in a directly-edited sudoers file can be a very bad day).

```bash
visudo                    # safely edit /etc/sudoers
```

Basic sudoers rule syntax: `user host=(runas) commands`

```
alice ALL=(ALL) ALL              # alice can run any command as any user on any host, with password prompt
alice ALL=(ALL) NOPASSWD: ALL      # same, but never prompts for a password (use very sparingly!)
%devs ALL=(ALL) /usr/bin/systemctl restart myapp    # % prefix = a GROUP rule, not a single user
```

`NOPASSWD:` scoped to specific commands is the security-conscious pattern for automation (e.g. a CI deploy user that only needs to restart one specific service without a password prompt) — `NOPASSWD: ALL` for a human user defeats most of the point of having a password at all and should be flagged as a red flag in review.

**`/etc/sudoers.d/`** — drop-in directory for modular sudo rules, instead of editing the monolithic `/etc/sudoers` file directly. Each file inside follows the same syntax; convention is one file per user/team/purpose, keeps changes isolated and easy to review/revert individually. Also edited safely with `visudo -f /etc/sudoers.d/filename`.

```bash
visudo -f /etc/sudoers.d/alice
# add: alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
```

`sudo -l` — list what commands the CURRENT user is allowed to run via sudo (great self-service diagnostic, no need to inspect sudoers files directly).

### newgrp, id, groups

- **`id`** — shows the full identity picture: UID, primary GID, and every supplementary group (both names and numbers) for the current user or a specified user (`id alice`).
- **`groups`** — simpler, just lists group names the current (or specified) user belongs to.
- **`newgrp groupname`** — starts a new shell session where the specified group becomes your **active** primary group for that session, without needing to log out and back in. Useful right after being added to a new group (group membership changes don't take effect in your *current* login session until you either re-login or use `newgrp`) — a very common "why isn't my new group access working?!" moment resolved by explaining this.

```bash
id                    # your own full identity
id alice                # alice's identity
groups                    # your group list, simple form
newgrp devs                  # temporarily make devs your active group for this new shell
exit                            # return to previous shell/group context
```

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `useradd -m -s SHELL -G groups user` | create user with home dir, shell, groups | `useradd -m -s /bin/bash -G devs alice` |
| `usermod -aG group user` | append secondary group | `usermod -aG docker alice` |
| `usermod -s SHELL user` | change shell | `usermod -s /bin/zsh alice` |
| `usermod -l new old` | rename login | `usermod -l alicia alice` |
| `userdel -r user` | delete user + home dir | `userdel -r alice` |
| `groupadd name` | create group | `groupadd devs` |
| `groupmod -n new old` | rename group | `groupmod -n developers devs` |
| `groupdel name` | delete group | `groupdel devs` |
| `gpasswd -d user group` | remove user from secondary group | `gpasswd -d alice devs` |
| `passwd user` | set/change password | `passwd alice` |
| `passwd -l/-u user` | lock/unlock account | `passwd -l alice` |
| `chage -l user` | show password aging info | `chage -l alice` |
| `chage -M N user` | set max password age (days) | `chage -M 90 alice` |
| `visudo` | safely edit /etc/sudoers | `visudo` |
| `visudo -f path` | safely edit a sudoers.d drop-in | `visudo -f /etc/sudoers.d/alice` |
| `sudo -l` | list your own allowed sudo commands | `sudo -l` |
| `id [user]` | full identity: UID/GID/groups | `id alice` |
| `groups [user]` | list group memberships | `groups alice` |
| `newgrp group` | switch active group for new shell | `newgrp devs` |

## Examples / Walkthrough

```bash
# ── reading the account databases ─────────────────────────
cat /etc/passwd | head -n 5
awk -F: '{print $1, $3, $7}' /etc/passwd     # username, UID, shell — from lesson 4
awk -F: '$3 >= 1000 {print $1}' /etc/passwd    # human users only (UID 1000+)
sudo cat /etc/shadow | head -n 3                 # needs sudo — root-only file
cat /etc/group | head -n 5

# ── creating a user ────────────────────────────────────────
sudo useradd -m -s /bin/bash -c "Test User" testuser
cat /etc/passwd | grep testuser
ls -la /home/testuser              # home dir created by -m, populated from /etc/skel
id testuser
groups testuser

# ── setting a password and aging policy ───────────────────
sudo passwd testuser
# (interactively type a password twice)
sudo chage -l testuser
sudo chage -M 90 -m 7 -W 7 testuser
sudo chage -l testuser                # confirm new values

# ── groups ──────────────────────────────────────────────────
sudo groupadd devs
sudo usermod -aG devs testuser
groups testuser                          # now includes devs
cat /etc/group | grep devs                 # testuser listed as secondary member

sudo usermod -aG docker testuser 2>/dev/null || echo "docker group may not exist on this system"

# demonstrate the -aG vs -G danger
sudo usermod -G devs testuser              # REPLACES all secondary groups with just devs
groups testuser                              # confirm docker (if it was added) is now gone

# ── modifying and locking ─────────────────────────────────
sudo usermod -s /bin/sh testuser
grep testuser /etc/passwd                    # confirm shell changed
sudo passwd -l testuser                        # lock account
sudo passwd -S testuser                          # confirm status shows locked
sudo passwd -u testuser                            # unlock again

# ── sudo configuration ────────────────────────────────────
sudo visudo -f /etc/sudoers.d/testuser
# add this line, save and exit:
# testuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl status testuser.service

sudo -l -U testuser                 # confirm the rule is recognized (as root, checking testuser's rules)

# ── newgrp / id / groups in practice ──────────────────────
id testuser
groups testuser
# (as testuser, in their own session): newgrp devs   — makes devs active group for that shell

# ── cleanup ───────────────────────────────────────────────
sudo rm -f /etc/sudoers.d/testuser
sudo userdel -r testuser
sudo groupdel devs
```

## Common Pitfalls

- **Forgetting `-m` on `useradd`** — no home directory gets created at all; the user technically exists but has nowhere to log into cleanly, and any `~/.bashrc`-style setup is missing.
- **Using `-G` instead of `-aG` on `usermod`** — silently wipes out all existing secondary group memberships, replacing them with only whatever was just specified. This is one of the most common real-world sysadmin mistakes.
- **Editing `/etc/sudoers` directly with `vim`/`nano` instead of `visudo`** — a syntax error saved directly can break `sudo` system-wide with no safety net, since `visudo` is exactly what validates syntax before committing the change.
- **Assuming `/etc/passwd` contains passwords** — it hasn't for decades; actual hashes live in the much more restricted `/etc/shadow`.
- **Newly added group membership "not working"** — group changes don't apply to an already-open login session; the user needs to either log out/in again or use `newgrp groupname` to get the new group active without a full re-login.
- **`groupdel` failing unexpectedly** — a group can't be deleted while it's still some user's **primary** group; the user's primary group must be reassigned (`usermod -g othergroup user`) first.
- **`NOPASSWD: ALL` for a human sudoer** — grants full unrestricted, passwordless root — essentially removes any accountability/audit value `sudo` was providing. Scope `NOPASSWD` to specific commands whenever possible, especially for automation accounts.
- **Confusing primary vs secondary group listing** — grepping `/etc/group` for a username and not finding them when they clearly have access — because their primary group membership (set in `/etc/passwd`) isn't listed as a member in `/etc/group` at all, only secondary memberships are.
- **`userdel` without `-r` leaving orphaned files** — the account is gone but `/home/username` and mail spool remain, potentially confusing later audits ("who owns these files with a UID that doesn't exist anymore?").
- **Renaming a user with `usermod -l` and expecting the home directory to follow** — it doesn't automatically; home directory path and ownership must be updated separately (`usermod -d /home/newname -m`).

## FAQ

**Q: If `/etc/passwd` doesn't store real passwords, why does it still have a password field at all?**
A: Historical/compatibility reasons — the field format is preserved but is always just `x` (or sometimes `*`) on modern systems, signaling "the real hash is in `/etc/shadow`". Removing the field entirely would break decades of tooling that expects `/etc/passwd`'s 7-field format.

**Q: What's the actual difference between a user's primary group and secondary groups?**
A: Every user has exactly ONE primary group (recorded directly in `/etc/passwd`) — new files they create default to this group's ownership. Secondary (supplementary) groups are additional memberships (recorded in `/etc/group`'s member list) that grant extra access without changing what group new files get by default.

**Q: Why did adding myself to the `docker` group not let me run docker commands right away?**
A: Group membership is evaluated when you log in / start a session, not live. You need to either fully log out and back in, or run `newgrp docker` to activate it in a new shell without a full re-login.

**Q: What's the actual danger of `usermod -G group user` versus `usermod -aG group user`?**
A: `-G` alone SETS (replaces) the complete list of secondary groups to exactly what you specify — anything not listed is removed. `-aG` APPENDS the specified group(s) to whatever the user already had. Using bare `-G` when you meant to just add one group silently strips all their other group access.

**Q: Why must I use `visudo` instead of just editing `/etc/sudoers` with `nano`?**
A: `visudo` locks the file against concurrent edits and, critically, validates the syntax before allowing the save to actually take effect — catching a typo before it can break `sudo` for everyone. A broken sudoers file edited directly can leave a system with no way to gain root access without physical/console recovery.

**Q: What does the `%` prefix mean in a sudoers rule like `%devs ALL=(ALL) ALL`?**
A: It marks the rule as applying to a GROUP (`devs`) rather than an individual username — every member of that group gets the granted access, which is usually easier to maintain than per-user rules as team membership changes.

**Q: When is `NOPASSWD` actually a reasonable thing to use?**
A: Scoped tightly to a specific command and typically for automation accounts (e.g. a CI/CD deploy user that only needs to run `systemctl restart myapp` without a password prompt in a non-interactive pipeline) — never broadly as `NOPASSWD: ALL` for an interactive human account, which defeats the audit/accountability purpose of requiring authentication.

**Q: What's the difference between `passwd -l` and actually deleting the user?**
A: `passwd -l` (lock) disables password-based login while leaving the account, home directory, and all files completely intact and reversible with `passwd -u`. Deleting (`userdel`) removes the account entirely (and optionally the home directory with `-r`) — not reversible without restoring from backup.

**Q: What's the difference between `id` and `groups`?**
A: `groups` gives a quick simple list of group names. `id` gives the complete picture — numeric UID, primary GID, and every group (name AND number) — more detail, useful when debugging permission issues where the actual numeric IDs matter (e.g. comparing against a file's numeric owner/group shown by `ls -n` or `stat`).

**Q: Why would I ever need `chage` if `passwd` already changes passwords?**
A: `passwd` changes the password itself (or locks/unlocks). `chage` controls the *aging policy* around it — how often it must be changed, how much advance warning is given, when the account itself expires — security/compliance controls that `passwd` alone doesn't manage.

## Practice / Exercise

**Core (everyone should finish):**
1. Read `/etc/passwd` and, using `awk` (from Lesson 4), print only username and shell for every account with UID ≥ 1000.
2. Create a new user with a home directory, a specific shell, and a full-name comment, in one `useradd` command.
3. Set that user's password and view their password-aging info with `chage -l`.
4. Set a max password age of 60 days and a 7-day warning period with `chage`.
5. Create a new group, add your test user to it with `usermod -aG`, and confirm with both `groups` and by grepping `/etc/group`.
6. Deliberately run `usermod -G` (no `-a`) with a different single group and observe the user's other group memberships disappear — explain why, then fix it back with `-aG`.
7. Lock the test user's account with `passwd -l`, confirm with `passwd -S`, then unlock it.
8. Use `visudo -f` to create a sudoers.d drop-in file granting your test user `NOPASSWD` access to exactly one harmless read-only command (e.g. `systemctl status testuser.service`) — never a broad `ALL`.
9. Clean up: remove the sudoers.d file, delete the user with `userdel -r`, delete the group.

**Stretch (fast finishers):**
10. Compare `id yourtestuser` and `groups yourtestuser` output side by side — identify which one shows numeric IDs and which doesn't.
11. Investigate what happens when you try `groupdel` on a group that is still some user's PRIMARY group — read the error, then fix it by reassigning that user's primary group first.
12. Research (or ask instructor) what `/etc/skel` is and why files placed there appear automatically in every new user's home directory created with `useradd -m`.
13. Write a sudoers.d rule using the `%group` syntax instead of a username, granting an entire group access to one specific command — test by adding your test user to that group and running `sudo -l`.

## Further Reading

- `man 5 passwd`, `man 5 shadow`, `man 5 group` — full field-by-field format reference for each file
- `man useradd`, `man usermod`, `man userdel` — full flag reference
- `man sudoers` — complete sudoers file syntax, including advanced aliasing and defaults
- `man chage` — full password aging flag reference
