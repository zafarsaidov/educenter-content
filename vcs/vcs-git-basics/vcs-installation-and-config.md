# Installation and Configuration

Before you can use Git you need to install it and tell it who you are. This lesson walks through installation on every major operating system, explains Git's three-level configuration system, covers the essential settings every developer should apply, and shows you how to set up an SSH key for GitLab so you never have to type a password when pushing or pulling.

## Installing Git

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install git
```

### macOS

The easiest way is via Homebrew. If you do not have Homebrew, install it from [brew.sh](https://brew.sh) first.

```bash
brew install git
```

### Windows

**Option 1 — winget (Windows Package Manager, built into Windows 11):**

```bash
winget install --id Git.Git -e --source winget
```

**Option 2 — official installer:**

Download from [git-scm.com](https://git-scm.com/download/win) and run the installer. The defaults are sensible; the only setting worth changing during install is selecting your preferred default editor.

### Verify the installation

```bash
git --version
```

```text
git version 2.44.0
```

Any version 2.x is fine for everything in this course.

## Git Config Levels

Git stores configuration in three separate files, each with a different scope. A setting in a narrower scope overrides the same setting in a broader one.

| Level | Flag | File location | Applies to |
|-------|------|---------------|------------|
| System | `--system` | `/etc/gitconfig` | All users on the machine |
| Global | `--global` | `~/.gitconfig` | Current user, all repos |
| Local | `--local` | `.git/config` inside the repo | Current repository only |

**Override order: local > global > system**

This means you can set your work email globally and override it with a personal email in a specific repository, without touching the global config.

```bash
# View the file Git is writing to for each level
git config --system --list   # /etc/gitconfig
git config --global --list   # ~/.gitconfig
git config --local  --list   # .git/config (must be inside a repo)
```

## Essential Configuration

Run these commands once after installing Git. They set your identity, preferred editor, and default branch name globally.

### Identity

Git attaches your name and email to every commit you make. Without these, Git will complain or use system defaults that will look wrong in GitLab.

```bash
git config --global user.name  "Firstname Lastname"
git config --global user.email "you@example.com"
```

### Default editor

Git opens a text editor when you run `git commit` without `-m`, or when you rebase interactively. Set it to something you are comfortable with.

```bash
# Vim (default on most systems)
git config --global core.editor "vim"

# Nano (simpler alternative)
git config --global core.editor "nano"

# VS Code (--wait tells Git to wait until you close the file tab)
git config --global core.editor "code --wait"
```

### Default branch name

Newer Git versions default to `main` instead of `master`. Set this explicitly so new repositories always start with `main`.

```bash
git config --global init.defaultBranch main
```

## Viewing Your Configuration

```bash
# Show all effective settings (merged from all levels)
git config --list

# Show only global settings
git config --global --list

# Query a single value
git config user.email
git config user.name
```

If a key appears twice in `git config --list`, the last occurrence wins — that is the active value.

## Setting Up SSH for GitLab

Connecting to GitLab over SSH means Git authenticates with a key pair instead of a username and password. You only set this up once per machine.

### Step 1 — Generate an SSH key pair

The `ed25519` algorithm is modern, fast, and recommended by GitLab.

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

You will be prompted for a file path (press Enter to accept the default `~/.ssh/id_ed25519`) and an optional passphrase (recommended — it protects the key if your laptop is stolen).

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/you/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/you/.ssh/id_ed25519
Your public key has been saved in /home/you/.ssh/id_ed25519.pub
```

### Step 2 — Copy the public key

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the entire output. It starts with `ssh-ed25519` and ends with your comment (email).

### Step 3 — Add the key to GitLab

1. Open GitLab → click your avatar → **Preferences** → **SSH Keys**.
2. Paste the public key into the **Key** field.
3. Give it a recognisable title (e.g., `Work laptop - 2024`).
4. Click **Add key**.

### Step 4 — Test the connection

```bash
ssh -T git@gitlab.com
```

Expected output:

```text
Welcome to GitLab, @yourusername!
```

If you see `Permission denied (publickey)`, double-check that you pasted the `.pub` file (public key), not the private key.

### Using SSH vs HTTPS

When you clone a repository, GitLab gives you two URL formats:

```bash
# HTTPS — prompts for credentials (or uses a stored token)
git clone https://gitlab.com/yourgroup/yourrepo.git

# SSH — uses your key pair, no password prompt
git clone git@gitlab.com:yourgroup/yourrepo.git
```

For day-to-day development, SSH is more convenient. HTTPS is useful in CI environments where injecting a token is easier than managing SSH keys.

## Common Pitfalls

- Forgetting to set `user.name` and `user.email` — commits will show a wrong or empty author, which cannot be changed easily once pushed.
- Using `--system` when you meant `--global` — system config requires root/admin privileges and affects all users on the machine.
- Adding the private key (`id_ed25519`) to GitLab instead of the public key (`id_ed25519.pub`) — the private key must never leave your machine.
- Not passing `--wait` to `code` — without it, Git thinks you closed the editor immediately and creates an empty commit message.
- Running `git config --list` and seeing duplicate keys — Git merges all levels; the last value shown is the one that applies.

## Xulosa

- Install Git with your OS package manager and verify with `git --version`.
- Git has three config levels: `--system` (all users), `--global` (current user), `--local` (current repo). Local overrides global overrides system.
- Always set `user.name`, `user.email`, `core.editor`, and `init.defaultBranch` globally after a fresh install.
- Use `git config --list` to see merged settings and `git config <key>` to query a single value.
- Generate an `ed25519` SSH key, add the `.pub` file to GitLab, and test with `ssh -T git@gitlab.com`.
- SSH URLs (`git@gitlab.com:...`) avoid password prompts; HTTPS URLs are better for CI/token-based auth.
