# Lesson 7: Linux Package Management

**Module:** Linux Basics
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-6 (terminal basics, permissions, find/grep/awk, systemd, users/groups/sudo)

## Learning Objectives

By end of lesson student can:
- explain the APT ecosystem: repositories, PPAs, GPG signing
- install/update/remove/purge software safely with `apt`
- search for and inspect package information before installing
- use `dpkg` for low-level package inspection and installing standalone `.deb` files
- use `snap` as an alternative packaging system and explain how it differs from APT

## Topics

- APT ecosystem: /etc/apt/sources.list, PPAs (add-apt-repository), GPG keys
- apt: update, upgrade, full-upgrade, install, remove, purge, autoremove
- apt: search, show, list --installed, --upgradable; apt-cache
- dpkg: -i (install .deb), -r, -l, -s, -L (list installed files)
- snap: install, list, remove, refresh, channels (stable/edge/beta)

## Concepts

### The APT ecosystem

**APT** (Advanced Package Tool) is Ubuntu/Debian's high-level package management system — it resolves dependencies, downloads packages, and installs them, all from configured **repositories** (remote servers hosting collections of `.deb` packages).

**`/etc/apt/sources.list`** — the main file listing which repositories APT should pull from. Modern Ubuntu also uses `/etc/apt/sources.list.d/*.list` — a drop-in directory (same pattern as `/etc/sudoers.d/` from Lesson 6) so third-party repos can add their own file instead of editing the main one.

A typical line:
```
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
```
- `deb` — binary packages (vs `deb-src` for source packages)
- URL — the repository server
- `jammy` — the Ubuntu release codename (this repo is release-specific)
- `main restricted universe multiverse` — **components**: `main` (officially supported, free), `restricted` (officially supported, proprietary drivers), `universe` (community-maintained, free), `multiverse` (community-maintained, restricted/proprietary)

**PPA (Personal Package Archive)** — a way for individuals/teams to publish their own APT repository (usually via Launchpad), for software not in Ubuntu's official repos or newer versions than the default release ships. Added with:
```bash
sudo add-apt-repository ppa:someuser/some-ppa
sudo apt update            # required after adding any new repo
```
Trade-off worth flagging: PPAs are NOT vetted by Ubuntu/Canonical the way official repos are — installing one means trusting that maintainer's build pipeline; fine for well-known, widely-used PPAs, riskier for obscure ones.

**GPG keys** — every official/trusted repository signs its package index with a GPG key; APT verifies this signature before trusting any package list, protecting against a compromised mirror or man-in-the-middle serving malicious packages. Modern Ubuntu stores trusted keys under `/etc/apt/trusted.gpg.d/` (or per-repo keyrings referenced from the `.list`/`.sources` file) rather than one shared legacy keyring. Adding a new third-party repo (not a PPA via `add-apt-repository`, which handles keys automatically) requires manually importing that maintainer's GPG key first, or `apt update` will refuse the repo with a signature-verification error.

### apt — everyday package operations

| Command | Effect |
|---|---|
| `apt update` | refresh the local package **index** (list of available packages/versions) from configured repos — does NOT install/upgrade anything itself |
| `apt upgrade` | upgrade all installed packages to latest available version, **without** removing any currently-installed package (won't install new deps that would require removing something) |
| `apt full-upgrade` | like `upgrade`, but will also remove packages if needed to complete an upgrade (handles more complex dependency changes) |
| `apt install pkg` | install a package (and its dependencies) |
| `apt install pkg=version` | install a specific version |
| `apt remove pkg` | uninstall the package, but **leave its configuration files** in place |
| `apt purge pkg` | uninstall the package AND delete its configuration files too |
| `apt autoremove` | remove packages that were auto-installed as dependencies and are no longer needed by anything |

```bash
sudo apt update                       # always run this first — refresh what's "available"
sudo apt install tree                   # install a package
sudo apt remove tree                      # uninstall, config files kept
sudo apt purge tree                         # uninstall, config files also deleted
sudo apt autoremove                           # clean up orphaned dependencies
sudo apt upgrade                                # safe, conservative upgrade of everything installed
sudo apt full-upgrade                             # more thorough, may remove packages if truly necessary
```

**Critical habit to instill:** `apt update` refreshes the index (what COULD be installed/what version is available); it never installs or changes anything on its own. Students often confuse `update` with `upgrade` — "I ran update, why didn't my packages get newer?" is one of the single most common beginner questions in this space.

### apt — searching and inspecting before installing

| Command | Purpose |
|---|---|
| `apt search keyword` | search package names/descriptions for a keyword |
| `apt show pkgname` | full details: description, version, size, dependencies, maintainer |
| `apt list --installed` | list every currently installed package |
| `apt list --upgradable` | list packages with a newer version available |
| `apt-cache policy pkgname` | show installed vs candidate version, and which repo it'd come from — useful for pinning/troubleshooting version conflicts |

```bash
apt search nginx
apt show nginx
apt list --installed | head -n 20
apt list --upgradable
apt-cache policy nginx
```

`apt-cache` is technically the older/lower-level tool that `apt search`/`apt show` largely wrap with friendlier output — still useful directly for things like `apt-cache policy` that don't have as clean an `apt`-native equivalent.

### dpkg — low-level package tool

`dpkg` operates on individual `.deb` package files directly and manages the local package database — it does **not** resolve dependencies or fetch anything from the internet (that's exactly what APT adds on top of dpkg).

| Command | Purpose |
|---|---|
| `dpkg -i package.deb` | install a local `.deb` file directly |
| `dpkg -r packagename` | remove a package (config files kept, like `apt remove`) |
| `dpkg -l` | list all installed packages (with version + short description) |
| `dpkg -l pattern` | list installed packages matching a pattern |
| `dpkg -s packagename` | show status/info for one installed package |
| `dpkg -L packagename` | list every file installed by a package — where did this binary/config actually go? |

```bash
dpkg -l | head -n 20
dpkg -l | grep nginx           # from Lesson 4
dpkg -s bash
dpkg -L bash                     # every file bash's package put on disk
```

**When dpkg alone breaks:** `dpkg -i somepackage.deb` can fail with unmet dependencies it can't resolve itself (no internet-fetching capability). Fix: `sudo apt install -f` (`-f` = "fix broken") right after — this tells APT to find and install whatever dependencies are missing to satisfy the already-registered-but-broken `dpkg` install.

### snap — Ubuntu's alternative packaging system

**Snap** packages are self-contained ("sandboxed") — bundling their own dependencies/runtime, isolated from the rest of the system, updating independently of APT's release cycle. Different philosophy from `.deb`/APT: trades some disk space and startup-time overhead for consistency across distros and automatic background updates.

| Command | Purpose |
|---|---|
| `snap install name` | install a snap |
| `snap list` | list installed snaps |
| `snap remove name` | uninstall a snap |
| `snap refresh` | update all snaps (or `snap refresh name` for one) |
| `snap info name` | details about a snap, including available channels |

**Channels** — snaps can track different release tracks simultaneously available side by side: `stable` (default, production-safe), `candidate`, `beta`, `edge` (bleeding-edge/nightly-equivalent). Install a non-default channel with `snap install name --channel=beta` or `snap install name --edge`.

```bash
snap list
sudo snap install hello-world
hello-world
snap info hello-world
sudo snap refresh hello-world
sudo snap remove hello-world
```

APT vs snap, the practical takeaway: use APT/`.deb` for most system packages and anything where you want tight integration with the base OS; snap fills gaps for software that needs frequent auto-updates, cross-distro portability, or stronger sandboxing (e.g. some desktop apps) — many DevOps toolchains still ship as `.deb`/binary tarballs rather than snaps, so APT remains the primary daily tool.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `apt update` | refresh package index | `sudo apt update` |
| `apt upgrade` | upgrade installed packages, conservative | `sudo apt upgrade` |
| `apt full-upgrade` | upgrade, may remove packages | `sudo apt full-upgrade` |
| `apt install pkg` | install package | `sudo apt install tree` |
| `apt remove pkg` | uninstall, keep config | `sudo apt remove tree` |
| `apt purge pkg` | uninstall, remove config too | `sudo apt purge tree` |
| `apt autoremove` | clean orphaned dependencies | `sudo apt autoremove` |
| `apt search kw` | search by keyword | `apt search nginx` |
| `apt show pkg` | show package details | `apt show nginx` |
| `apt list --installed` | list installed packages | `apt list --installed` |
| `apt list --upgradable` | list packages with updates available | `apt list --upgradable` |
| `apt-cache policy pkg` | installed vs candidate version | `apt-cache policy nginx` |
| `add-apt-repository ppa:x/y` | add a PPA | `sudo add-apt-repository ppa:x/y` |
| `dpkg -i file.deb` | install local .deb | `sudo dpkg -i app.deb` |
| `dpkg -l` | list installed packages | `dpkg -l` |
| `dpkg -s pkg` | package status/info | `dpkg -s bash` |
| `dpkg -L pkg` | files owned by a package | `dpkg -L bash` |
| `snap install name` | install a snap | `sudo snap install hello-world` |
| `snap list` | list installed snaps | `snap list` |
| `snap remove name` | uninstall a snap | `sudo snap remove hello-world` |
| `snap refresh` | update snaps | `sudo snap refresh` |

## Examples / Walkthrough

```bash
# ── refreshing and searching ───────────────────────────────
sudo apt update
apt search tree
apt show tree
apt list --upgradable | head -n 10

# ── installing and inspecting ─────────────────────────────
sudo apt install tree
tree --version
dpkg -l | grep tree
dpkg -s tree
dpkg -L tree                        # every file the tree package installed

# ── removing vs purging ───────────────────────────────────
sudo apt remove tree
dpkg -l | grep tree                    # note: still listed, marked "rc" (config remains)
sudo apt purge tree
dpkg -l | grep tree                       # gone completely now

# ── autoremove demo ────────────────────────────────────────
apt list --installed | wc -l              # baseline count (from Lesson 4)
sudo apt install nginx                       # pulls in several dependencies
sudo apt purge nginx
sudo apt autoremove                            # cleans up now-orphaned dependencies nginx pulled in

# ── upgrade vs full-upgrade ────────────────────────────────
apt list --upgradable
sudo apt upgrade                    # conservative — run during regular maintenance
# sudo apt full-upgrade              # only when needed (e.g. release-to-release dependency shifts)

# ── PPA example (illustrative — only run if instructor confirms a safe PPA to demo) ──
sudo add-apt-repository ppa:example/example
sudo apt update
apt-cache policy somepackage-from-that-ppa

# ── dpkg on a standalone .deb ──────────────────────────────
# (assuming a .deb file has been downloaded, e.g. to ~/lab7/somepkg.deb)
mkdir -p ~/lab7
sudo dpkg -i ~/lab7/somepkg.deb 2>/dev/null || echo "may report missing dependencies"
sudo apt install -f                   # fixes any dependency gaps dpkg alone couldn't resolve

# ── snap ────────────────────────────────────────────────────
snap list
sudo snap install hello-world
hello-world
snap info hello-world
sudo snap refresh hello-world
sudo snap remove hello-world
snap list                               # confirm it's gone

# ── cleanup ───────────────────────────────────────────────
rm -r ~/lab7
```

## Common Pitfalls

- **Confusing `apt update` with `apt upgrade`** — `update` only refreshes what's *available* (the index); it installs nothing. Forgetting to `update` before `install`/`upgrade` means working from a stale package list — possibly missing recent security patches or trying to install a version that no longer exists upstream.
- **Running `apt upgrade` without ever running `apt update` first** — upgrade decisions are made against a potentially stale index; always `update` immediately before `upgrade`/`install` in any real workflow or script.
- **`apt remove` leaving config files, expecting a clean slate** — a subsequent reinstall may reuse old config, confusing "fresh install" expectations. Use `purge` when a truly clean removal is needed.
- **Adding a PPA blindly without checking its trustworthiness** — PPAs are unvetted third-party repos; a malicious or poorly maintained one can install compromised packages with full system privileges via `apt install`.
- **Forgetting `sudo apt update` after adding a PPA/new repo** — the newly added repository's package list isn't fetched until the next `update`; trying to `install` immediately after adding a PPA (without updating first) fails with "unable to locate package".
- **`dpkg -i` failing with dependency errors and not knowing how to fix it** — `dpkg` alone can't fetch missing dependencies from the internet; the fix is `sudo apt install -f` immediately afterward, which lets APT complete what dpkg started.
- **Assuming `snap` and `apt` versions of "the same" software behave identically** — snaps are sandboxed and may have different file paths, permissions behavior, or auto-update timing than the APT-packaged equivalent; don't assume perfect parity.
- **Not realizing snaps auto-update in the background by default** — unlike APT (which only updates when you explicitly run `upgrade`), snaps refresh themselves automatically on a schedule — can surprise people expecting fully manual control.
- **Running `apt full-upgrade` casually, expecting it to behave exactly like `upgrade`** — it CAN remove packages to resolve upgrade paths; fine for planned maintenance, riskier to run blindly on a production system without reviewing what it proposes to remove first.

## FAQ

**Q: What's the real difference between `apt update` and `apt upgrade`?**
A: `update` refreshes APT's local knowledge of what packages/versions are *available* from configured repositories — it changes nothing installed on your system. `upgrade` actually installs newer versions of packages you already have, based on whatever the most recent `update` says is available. You virtually always want `update` immediately before `upgrade`.

**Q: When would I use `full-upgrade` instead of plain `upgrade`?**
A: When a regular `upgrade` can't proceed because completing it would require removing an existing package (e.g. due to a dependency restructuring between releases) — `upgrade` refuses in that case to avoid surprise removals; `full-upgrade` allows it. Use `full-upgrade` for planned, reviewed maintenance windows, not routine casual updates.

**Q: What's the difference between `remove` and `purge`?**
A: `remove` uninstalls the package's binaries but leaves its configuration files behind (in case you reinstall later and want your settings preserved). `purge` removes everything, including configuration — use it when you want a genuinely clean slate.

**Q: Why would I need a PPA if Ubuntu already has an official package for the same software?**
A: Usually for a newer version than what the current Ubuntu release ships (Ubuntu's official repos favor stability over bleeding-edge versions), or for software that isn't in the official repos at all.

**Q: Is it safe to add any PPA I find online?**
A: No — PPAs aren't vetted by Canonical/Ubuntu the way official repos are; you're trusting whoever maintains that PPA with the ability to run arbitrary code as root on your system via `apt install`. Stick to well-known, widely-used, actively maintained PPAs from reputable maintainers.

**Q: Why did `dpkg -i mypackage.deb` fail even though the file looked fine?**
A: Almost always missing dependencies — `dpkg` installs the package's own files but doesn't fetch anything else it needs from the internet. Run `sudo apt install -f` right after to let APT resolve and install those missing dependencies, completing the process `dpkg` started.

**Q: What does `dpkg -L packagename` actually show me, and when is it useful?**
A: The full list of every file that package placed on the filesystem during installation — useful when you know a config file or binary exists somewhere but aren't sure which package owns it, or want to fully understand what a package touched before removing it.

**Q: How is a snap different from a regular `.deb`/apt package?**
A: A snap bundles its own dependencies and runs somewhat sandboxed/isolated from the rest of the system, updating on its own independent schedule (auto-refresh in the background). A `.deb`/APT package relies on shared system libraries and is only updated when you explicitly run `apt upgrade`. Snaps trade some disk space/startup overhead for consistency and easier cross-distro distribution.

**Q: Do snaps update automatically? Can I control that?**
A: Yes, by default snaps auto-refresh on a schedule set by the snap store (`snap refresh` can also be run manually anytime). Refresh timing/hold windows can be configured (e.g. `snap refresh --hold`) if you need tighter control, though the details go beyond today's basics.

**Q: What's the point of GPG-signing a repository's package index?**
A: It lets APT cryptographically verify that the package list actually came from the claimed maintainer and wasn't tampered with in transit (e.g. by a compromised mirror or a man-in-the-middle) — without it, a malicious actor could substitute compromised packages for legitimate ones.

## Practice / Exercise

**Core (everyone should finish):**
1. Run `apt update`, then use `apt search` to find a package by keyword (e.g. "tree" or "curl"), then `apt show` it to review its details before installing.
2. Install that package, confirm with `dpkg -l | grep`, and inspect every file it placed on disk with `dpkg -L`.
3. Remove it with `apt remove`, confirm via `dpkg -l` that it still shows a config-remaining state, then fully `purge` it and confirm it's gone.
4. Run `apt list --upgradable` and interpret the output — are there pending upgrades? Run `apt upgrade` if instructor approves doing so in the lab environment.
5. Use `apt-cache policy` on a package you've installed and explain what "Installed" vs "Candidate" means in its output.
6. Install a package that pulls in several dependencies (e.g. `nginx`), purge it, then run `apt autoremove` and observe what gets cleaned up.

**Stretch (fast finishers):**
7. Install `hello-world` via `snap`, run it, inspect it with `snap info`, then remove it — compare the experience (install time, file layout if inspectable) against a regular apt-installed package.
8. Research (or ask instructor) what channel `stable` vs `edge` means for a snap you're curious about, using `snap info packagename`.
9. If a safe demo PPA is provided by the instructor, add it with `add-apt-repository`, run `apt update`, and use `apt-cache policy` to confirm the new repo's version is now visible as a candidate — then remove the PPA and update again.
10. Explain in your own words (no need to execute anything) why `dpkg` by itself cannot install a `.deb` file with unmet dependencies, but `apt install ./file.deb` generally can.

## Further Reading

- `man apt`, `man apt-get`, `man apt-cache` — official reference (note: `apt` is the modern user-friendly frontend; `apt-get`/`apt-cache` are the older, more script-stable underlying tools, still common in scripts/automation)
- `man dpkg` — full dpkg flag reference
- Ubuntu Package documentation: https://help.ubuntu.com/community/AptGet/Howto
- Snapcraft docs: https://snapcraft.io/docs
