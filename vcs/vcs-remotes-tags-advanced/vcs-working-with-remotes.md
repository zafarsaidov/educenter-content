# Working with Remotes

A remote is a named reference to another Git repository — most often hosted on a server like GitLab or GitHub. Understanding how your local repository communicates with remotes is the foundation of every collaborative Git workflow.

## What Is a Remote?

A remote is simply an alias for a URL. Instead of typing `https://gitlab.com/yourname/project.git` every time, you give it a short name and use that name in commands.

The name `origin` is a convention, not a rule. When you run `git clone`, Git automatically creates a remote called `origin` pointing to the URL you cloned from. You can name your remotes anything you like.

## Adding and Managing Remotes

### Adding a Remote

```bash
git remote add origin https://gitlab.com/yourname/project.git
```

This creates a remote named `origin` pointing to that URL. Nothing is downloaded or uploaded yet.

### Listing Remotes

```bash
git remote -v
```

Sample output:

```text
origin  https://gitlab.com/yourname/project.git (fetch)
origin  https://gitlab.com/yourname/project.git (push)
```

Each remote shows two URLs — one used for fetching (downloading) and one for pushing (uploading). They are usually identical.

### Renaming a Remote

```bash
git remote rename origin gitlab
```

After this, all commands that previously used `origin` must use `gitlab`.

### Removing a Remote

```bash
git remote remove upstream
```

This only removes the local reference — it does not delete anything on the server.

## Fetching vs Pulling

These two commands are often confused. Understanding the difference prevents surprises.

### git fetch

```bash
git fetch origin
```

`git fetch` downloads all new commits, branches, and tags from the remote and stores them in your local **remote-tracking branches** (e.g., `origin/main`). It does **not** touch your working tree or your local branches. Your files on disk remain completely unchanged.

It is safe to run `git fetch` at any time, even in the middle of other work.

### git pull

```bash
git pull
```

`git pull` is a shortcut for two steps:

1. `git fetch origin`
2. `git merge origin/main` into your current branch

This means it can create a merge commit and may produce conflicts if the remote and local branches have diverged.

### git pull --rebase

```bash
git pull --rebase
```

Instead of merging after fetching, this replays your local commits on top of the fetched commits. The result is a linear history — no merge commits. Many teams configure this as the default:

```bash
git config --global pull.rebase true
```

## Pushing Branches

### Basic Push

```bash
git push origin feature/login
```

This pushes the local branch `feature/login` to the remote named `origin`. If the branch does not exist on the remote yet, Git creates it.

### Push and Set Upstream Tracking

```bash
git push -u origin feature/login
```

The `-u` flag (short for `--set-upstream`) links your local branch to the remote branch. After running this once, all future pushes and pulls on this branch can omit the remote and branch name:

```bash
git push      # works because tracking is configured
git pull      # same
```

Without `-u`, Git will remind you to specify the remote and branch each time.

## Tracking Branches

When Git knows which remote branch a local branch is tracking, it can show useful information.

```bash
git branch -vv
```

Sample output:

```text
  main           a1b2c3d [origin/main] Add README
* feature/login  e4f5a6b [origin/feature/login: ahead 2] Add login form
```

- `ahead 2` means you have 2 local commits not yet pushed to the remote.
- `behind 3` would mean the remote has 3 commits you have not yet fetched.
- `gone` means the remote branch has been deleted.

### Remote-Tracking Branches

`origin/main` is a **remote-tracking branch**. It is a read-only snapshot of the last known state of `main` on `origin`. It updates only when you run `git fetch` or `git pull`.

```text
main          ← your local branch (you can commit to it)
origin/main   ← what Git last saw on the remote (read-only locally)
```

If a colleague pushed three commits since your last fetch, your `origin/main` is stale until you run `git fetch`.

## Deleting a Remote Branch

After merging a feature branch, delete it on the remote to keep the repository tidy:

```bash
git push origin --delete feature/login
```

This does not delete your local `feature/login` branch. To delete the local branch as well:

```bash
git branch -d feature/login
```

## The Fork Workflow

Forking is common in open-source and large-organisation projects where contributors do not have direct write access to the main repository.

A fork creates your own copy of a repository under your account. You have two remotes:

| Remote     | URL                                          | Purpose                     |
|------------|----------------------------------------------|-----------------------------|
| `origin`   | `https://gitlab.com/yourname/project.git`    | Your personal fork          |
| `upstream` | `https://gitlab.com/originalowner/project.git` | The authoritative source |

### Setting Up the Upstream Remote

```bash
git remote add upstream https://gitlab.com/originalowner/project.git
```

### Keeping Your Fork in Sync

```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

Or in one step with rebase for a cleaner history:

```bash
git fetch upstream
git rebase upstream/main
git push origin main --force-with-lease
```

`--force-with-lease` is a safer alternative to `--force` — it refuses to overwrite the remote if someone else has pushed since your last fetch.

## Shallow Clones for CI

In CI pipelines you often only need the latest state of the code, not the full history. A shallow clone fetches only the most recent snapshot:

```bash
git clone --depth 1 https://gitlab.com/yourname/project.git
```

This can reduce clone time from minutes to seconds on large repositories. The downside is that operations requiring history — like `git log`, `git bisect`, or `git blame` — are limited or unavailable.

To convert a shallow clone into a full clone later:

```bash
git fetch --unshallow
```

## Common Pitfalls

- Forgetting to run `git fetch` before starting work — your local `origin/main` may be stale and you will base work on an outdated commit.
- Using `git pull` (with merge) on a shared branch when your team expects linear history — use `git pull --rebase` instead.
- Pushing without `-u` and then being confused why `git push` alone says "no upstream configured".
- Running `git push --force` instead of `git push --force-with-lease` — force without the lease can silently overwrite a teammate's commits.
- Deleting only the remote branch and leaving the local one — clean up both to avoid confusion.
- Shallow clones breaking `git describe` or semantic-version tooling that depends on tags and history depth.

## Summary

- A remote is a named alias for a repository URL; `origin` is just a convention set by `git clone`.
- `git remote add`, `git remote rename`, and `git remote remove` manage your remote references.
- `git fetch` downloads remote changes safely without touching your working tree; `git pull` fetches and then merges (or rebases).
- `git push -u origin <branch>` pushes and sets upstream tracking so future pushes require no arguments.
- `origin/main` is a read-only remote-tracking branch that reflects the last known state of the remote.
- Delete a remote branch with `git push origin --delete <branch>`.
- In the fork workflow you maintain two remotes: `origin` (your fork) and `upstream` (the source).
- Use `git clone --depth 1` for fast CI clones where full history is not needed.
