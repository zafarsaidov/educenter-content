# Lesson 14: Git Fundamentals and Workflow

**Module:** Git Version Control
**Duration:** 120-150 min
**Prerequisites:** Linux Basics module, Web Servers module (terminal fluency assumed; no prior Git knowledge required)

## Learning Objectives

By end of lesson student can:
- Explain what a VCS is, why distributed version control matters, and the roles of the working tree, staging area, and repository
- Configure Git's identity and defaults, and initialize or clone a repository
- Stage and commit changes with intent, and read commit history
- Work with remotes: push, pull, fetch
- Write an effective `.gitignore`, and inspect changes with `diff`/`show`

## Topics

- Git concepts: VCS, distributed model, repository, staging area, working tree
- Setup: `git config` (`user.name`, `email`, `editor`, alias); `git init`, `git clone`
- Basic workflow: `git add` (`-A`, `-p`), `git commit` (`-m`), `git status`, `git log` (`--oneline`, `--graph`)
- Working with remotes: `git remote` (`-v`, `add`, `remove`), `git push`, `git pull`, `git fetch`
- `.gitignore`: patterns, `git rm --cached`, global gitignore; `git diff`, `git show`

## Concepts

### What is version control, and why distributed?

A **version control system (VCS)** tracks changes to files over time, letting you see history, revert mistakes, and work on changes without overwriting others' work. Git is a **distributed** VCS: every clone of a repository is a full copy of the entire history, not just a pointer to a central server. This means you can commit, branch, and view history entirely offline — a network connection is only needed to synchronize with others (`push`/`pull`/`fetch`), not for every operation, unlike older centralized systems (e.g. SVN) where nearly everything required contacting a central server.

### The three areas: working tree, staging area, repository

Git models your project through three distinct areas:

| Area | What it holds |
|---|---|
| **Working tree** | The actual files on disk, as you edit them |
| **Staging area** (a.k.a. "index") | A holding area for changes you've explicitly marked to include in the *next* commit |
| **Repository** (`.git/` directory) | The full committed history — every snapshot ever recorded |

A change moves through these in order: you edit a file (working tree), `git add` it (staging area), then `git commit` it (repository, permanently recorded in history). This two-step "stage then commit" model — unusual compared to some other VCSs — lets you build a commit out of only *some* of your current changes, rather than being forced to commit everything you've touched at once.

### Setup: identity and config

Before committing, Git needs to know who you are — every commit is permanently stamped with an author name and email. `git config` sets this, along with other preferences like your default editor (for writing commit messages) and shortcuts (**aliases**) for commands you type often. Config can be set at three scopes: `--system` (whole machine), `--global` (your user account, most common for identity), `--local` (just one repository, the default when no flag is given).

### Starting a repository: `init` vs `clone`

`git init` turns an existing (or new, empty) directory into a Git repository from scratch — creates the `.git/` directory and starts history at zero commits. `git clone <url>` instead copies an *existing* repository (its full history, all branches) from somewhere else (a server, another local path) into a new directory on your machine, and automatically sets up that source as a remote named `origin`. Use `init` when starting something brand new; use `clone` when joining a project that already exists.

### The staging area in practice: `git add`

`git add <file>` moves a file's current changes into the staging area. `git add -A` stages everything changed, added, or deleted across the whole repository. `git add -p` (patch mode) is more surgical — it walks through each *chunk* of changes in a file interactively, letting you choose to stage only some of the changes within a single file, not the whole file at once. This matters when you've made several unrelated edits in one file but want them recorded as separate, focused commits.

### Committing and reading history

`git commit -m "message"` records everything currently staged as a new permanent snapshot, with the given message. `git status` shows the current state: which files are staged, unstaged, or untracked (not yet known to Git at all). `git log` lists commit history, newest first; `--oneline` compresses each commit to a single summary line, and `--graph` draws the branch/merge structure visually (most useful once branching is introduced in the next lesson, but works fine on a single-branch history too).

### Remotes: syncing with others

A **remote** is a named reference to another copy of the repository, usually hosted on a server (GitHub, GitLab, etc.). `git remote -v` lists configured remotes and their URLs; `origin` is the conventional name for the primary remote (auto-created by `clone`, or added manually with `git remote add origin <url>`).

| Command | Effect |
|---|---|
| `git fetch` | Downloads new commits/branches from the remote, but does **not** change your working tree or current branch |
| `git pull` | `fetch` followed immediately by merging the remote's changes into your current branch |
| `git push` | Uploads your local commits to the remote |

`fetch` is the "safe" one — it lets you look at what changed remotely before deciding whether/how to integrate it. `pull` is more convenient but immediately merges, which can surprise you if the remote has changes you weren't expecting.

### `.gitignore`

Not every file in a project directory belongs in version control — build artifacts, dependency folders, local secrets, editor swap files. `.gitignore` lists patterns for files/directories Git should never track or show as "untracked" in `git status`. If a file was already committed *before* being added to `.gitignore`, adding the pattern alone doesn't remove it from tracking — `git rm --cached <file>` untracks it (keeping the actual file on disk) so `.gitignore` can then take effect going forward. A **global gitignore** (configured via `git config --global core.excludesfile`) applies patterns across every repository on your machine — useful for editor/OS-specific junk files that have nothing to do with any particular project.

### Inspecting changes: `diff` and `show`

`git diff` shows the exact line-by-line difference between two states — by default, working tree vs staging area (unstaged changes); `git diff --staged` shows staging area vs the last commit (what would actually be committed next). `git show <commit>` displays the full details and diff of one specific historical commit.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `git config` | Set configuration | `git config --global user.name "Your Name"` |
| `git init` | Create a new repository | `git init` |
| `git clone` | Copy an existing repository | `git clone https://github.com/user/repo.git` |
| `git status` | Show working tree/staging state | `git status` |
| `git add` | Stage changes | `git add -A` or `git add -p file.txt` |
| `git commit` | Record staged changes | `git commit -m "message"` |
| `git log` | Show commit history | `git log --oneline --graph` |
| `git remote` | Manage remotes | `git remote -v` |
| `git push` | Upload commits to remote | `git push origin main` |
| `git pull` | Fetch + merge from remote | `git pull origin main` |
| `git fetch` | Download from remote, no merge | `git fetch origin` |
| `git diff` | Show unstaged/staged changes | `git diff --staged` |
| `git show` | Show one commit's details | `git show abc1234` |
| `git rm --cached` | Untrack a file, keep it on disk | `git rm --cached secrets.env` |

## Examples / Walkthrough

```bash
# --- setup: identity and preferences ---
git config --global user.name "Alex Student"
git config --global user.email "alex@example.com"
git config --global core.editor "vim"           # or "nano" — whichever you're comfortable in (Lesson 3)
git config --global alias.st status              # now "git st" works as a shortcut for "git status"
git config --global alias.co checkout

git config --list                                 # confirm everything is set as expected

# --- starting a repository ---
mkdir ~/git-practice && cd ~/git-practice
git init                                           # creates .git/ — an empty repository, no commits yet
git status                                         # "On branch main, no commits yet"

# alternative: cloning an existing repository
# git clone https://github.com/example-org/example-repo.git
# cd example-repo

# --- basic workflow: edit, stage, commit ---
echo "# My Practice Project" > README.md
git status                                         # README.md shown as "untracked"

git add README.md                                  # stage it
git status                                          # now shown as "staged" / "to be committed"

git commit -m "Add initial README"
git log --oneline                                   # one line: short hash + message

# make a few more changes to practice add -A vs add -p
echo "console.log('hello');" > app.js
mkdir src && echo "print('hi')" > src/main.py
git add -A                                           # stage everything: new files, edits, deletions
git status
git commit -m "Add app.js and src/main.py"

git log --oneline --graph                            # visualize history (linear so far, no branches yet)

# --- .gitignore ---
echo "node_modules/" > .gitignore
echo "*.log" >> .gitignore
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Add .gitignore"

touch debug.log                                      # matches *.log
git status                                            # debug.log does NOT appear — .gitignore is working

# untracking a file that was committed before being ignored (common real-world scenario)
echo "SECRET_KEY=abc123" > config.env
git add config.env
git commit -m "Oops, accidentally committed a secret"
echo "config.env" >> .gitignore
git rm --cached config.env                            # untrack it, but keep the file on disk
git commit -m "Stop tracking config.env"

# global gitignore, for OS/editor junk across every repo on this machine
echo ".DS_Store" >> ~/.gitignore_global
echo "*.swp" >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global

# --- diff and show ---
echo "console.log('hello world');" > app.js          # modify an existing tracked file
git diff                                              # unstaged diff: old line vs new line

git add app.js
git diff --staged                                     # same diff, now shown as staged (about to be committed)
git commit -m "Update greeting in app.js"

git show HEAD                                         # full details of the most recent commit
git show HEAD~1                                        # the commit before that

# --- remotes: push/pull/fetch ---
git remote -v                                          # (empty, if this is a local-only repo so far)

# adding a remote (using a real GitHub/GitLab repo URL you have push access to)
git remote add origin https://github.com/your-username/git-practice.git
git remote -v                                          # confirm origin is listed, fetch + push URLs

git push -u origin main                                 # first push: -u sets the upstream tracking branch
# after -u is set once, plain "git push" and "git pull" know which remote/branch to use

# simulate a teammate's change by editing directly on the remote host's web UI, then:
git fetch origin                                        # download the new commit, don't merge yet
git log origin/main --oneline                            # inspect what's new before merging
git pull origin main                                      # now actually merge it into local main
```

## Common Pitfalls

- **Forgetting to `git add` before `git commit`** — `git commit` only records what's currently *staged*, not every change in the working tree. A common surprise is committing and then running `git status` to find changes still sitting unstaged, absent from the commit just made.
- **Using `git add -A` reflexively without checking `git status` first** — it stages absolutely everything, including files you may not have meant to include (a stray debug file, a half-finished edit in an unrelated area). Get in the habit of reading `git status` before staging broadly.
- **Committing a secret, then thinking `.gitignore` retroactively removes it** — `.gitignore` only prevents *future* tracking. A file already committed remains in history even after being added to `.gitignore` and `git rm --cached`; the old commit still contains it. (Actually purging a secret from history is a more advanced operation, out of scope for this lesson — the practical lesson here is to `.gitignore` sensitive file patterns *before* ever committing them.)
- **Confusing `git fetch` and `git pull`** — `pull` immediately merges remote changes into your working branch, which can create a merge you weren't expecting if you hadn't looked at what changed. `fetch` first, then `log`/`diff` to review, is the safer habit when working on a shared branch.
- **Writing vague commit messages** ("fix stuff", "wip", "updates") — `git log` becomes much less useful for understanding *why* a change was made later. A message should describe the change's purpose, not just restate that something changed.
- **Not setting `user.name`/`user.email` before committing** — Git will still let you commit using a fallback identity guessed from the OS account, which usually isn't what you want permanently attached to your project history. Set it explicitly with `--global` before your first commit.

## FAQ

**Q: What's actually inside the `.git/` directory?**
A: The entire repository's data: every commit, every version of every tracked file, branch references, and configuration — all of Git's internal database. Deleting `.git/` (careful — this is destructive) removes all version history while leaving your current working files on disk untouched.

**Q: Why does Git have a separate staging area instead of just committing the working tree directly?**
A: It lets you build a commit deliberately from a subset of your current changes, rather than being forced to commit everything you've touched. If you've fixed a bug and also made an unrelated tweak in the same session, staging lets you commit them as two focused, separately-reviewable commits instead of one tangled one.

**Q: Do I need to `git add` a file every time I change it, even after adding it once before?**
A: Yes — staging is not a one-time "start tracking" action, it reflects the *current* state of changes. Editing a file again after committing puts it back in the "changed, not staged" state; you `add` it again to stage the new changes for the next commit.

**Q: What's the difference between `origin` and `main`?**
A: They're different kinds of names entirely. `origin` is the conventional name for a *remote* (a location — a URL pointing at another copy of the repo). `main` is the conventional name for the primary *branch* (a line of commit history). `git push origin main` reads as "push the `main` branch to the `origin` remote."

**Q: Is it safe to run `git pull` on a branch other people are also pushing to?**
A: Usually yes for straightforward cases, but if your local branch has commits the remote doesn't have yet, `pull`'s merge step can create a merge commit or, if histories conflict, require manual conflict resolution. `git fetch` plus reviewing `git log origin/main` first avoids being surprised by this.

## Practice / Exercise

**Core:**
1. Set your Git identity (`user.name`, `user.email`) globally, and set your preferred editor.
2. Create a new directory, `git init` it, and make 3 separate commits, each adding or changing something small, writing a clear message each time.
3. Run `git log --oneline` and `git log --graph` after your 3 commits and read the output.
4. Create a `.gitignore` with at least 3 patterns (e.g. `*.log`, `node_modules/`, `.env`), create matching files, and confirm with `git status` that they're correctly ignored.
5. Modify a tracked file, run `git diff` to see the unstaged change, `git add` it, then `git diff --staged` to see the same change from the other side.
6. Create a repository on GitHub or GitLab (via their web UI), add it as `origin` to your local repo, and push your commits with `git push -u origin main`.
7. Edit a file directly on the remote's web UI, then locally run `git fetch origin` followed by `git log origin/main --oneline` before finally `git pull` to bring the change down.

**Stretch:**
1. Practice `git add -p` on a file with two unrelated changes in it, staging only one chunk, committing it, then staging and committing the second chunk separately.
2. Commit a file, then afterward add a matching pattern to `.gitignore` and use `git rm --cached` to stop tracking it — explain in your own words why the file still appears in the earlier commit's history.
3. Set up a global gitignore file for OS/editor junk files and confirm (with a fresh `git init` in a new directory) that it applies automatically without being listed in that repo's own `.gitignore`.
4. Use `git show` on a commit from your history and identify each part of its output (author, date, message, diff).

## Further Reading

- [Pro Git book (free, official)](https://git-scm.com/book/en/v2)
- `man git`, `git help <command>` (e.g. `git help commit`)
- [git-scm.com documentation](https://git-scm.com/doc)
