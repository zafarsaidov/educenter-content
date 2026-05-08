# Branches

A branch in Git is nothing more than a lightweight movable pointer to a commit. Understanding that fact makes every branch operation intuitive — creating a branch costs almost nothing, switching is instant, and deleting one removes only the pointer, not the commits it pointed to.

## What a Branch Really Is

When you make commits, Git builds a chain of snapshot objects. A branch is simply a file inside `.git/refs/heads/` that contains the SHA-1 hash of the latest commit on that branch. Every time you commit, Git updates that file to point to the new commit.

`HEAD` is a special pointer that tells Git which branch (or commit) you are currently on. Normally `HEAD` points to a branch name, and the branch name points to the latest commit.

```text
          HEAD
            |
          main
            |
A --- B --- C
```

When you create a feature branch and make commits on it, the picture becomes:

```text
          HEAD
            |
       feature/login
            |
A --- B --- C --- D --- E
            \
             (main stays at C)
```

`main` and `feature/login` are just two pointers into the same commit graph. Git never duplicates content.

## Creating and Listing Branches

```bash
# Create a new branch (does NOT switch to it)
git branch feature/login-form

# List all local branches (* marks the current branch)
git branch

# List all branches, including remote-tracking ones
git branch -a
```

Sample output of `git branch -a`:

```text
* main
  feature/login-form
  remotes/origin/main
  remotes/origin/feature/login-form
```

## Switching Between Branches

The modern command is `git switch`. It is clearer and less error-prone than the old `git checkout`.

```bash
# Switch to an existing branch
git switch feature/login-form

# Old equivalent (still works)
git checkout feature/login-form
```

When you switch, Git updates the files in your working tree to match the snapshot at the tip of the target branch and moves `HEAD` to point to that branch.

### Creating and Switching in One Step

```bash
# Modern: create and switch
git switch -c feature/signup-form

# Old equivalent
git checkout -b feature/signup-form
```

## Deleting Branches

```bash
# Delete a branch that has already been merged (safe)
git branch -d feature/login-form

# Force-delete a branch even if it has unmerged commits (data may be lost)
git branch -D feature/abandoned-idea
```

Git refuses `-d` if the branch contains commits not reachable from your current branch, protecting you from accidentally throwing away work. Use `-D` only when you are certain those commits are no longer needed.

## Detached HEAD State

Normally `HEAD` points to a branch name, and the branch name points to a commit. Detached HEAD means `HEAD` points directly to a commit hash instead of to a branch.

This happens when you check out a specific commit, a tag, or a remote-tracking branch directly:

```bash
# These all produce detached HEAD
git checkout a3f9c21
git checkout v1.4.0
git checkout origin/main
```

Git warns you immediately:

```text
HEAD detached at a3f9c21
```

In detached HEAD state you can look around and even make experimental commits, but those commits are not reachable from any branch. If you switch away without saving them, they become orphaned and will eventually be garbage-collected.

### Recovering from Detached HEAD

If you made commits in detached HEAD state that you want to keep, create a branch from the current position before switching away:

```bash
git switch -c rescue/my-experiment
```

If you just want to return to a branch without keeping anything:

```bash
git switch main
```

## Branch Naming Conventions

Well-named branches make it obvious what work is in progress and integrate cleanly with issue trackers and CI/CD systems.

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality | `feature/login-form` |
| `bugfix/` | Fix a reported bug | `bugfix/null-pointer-on-logout` |
| `hotfix/` | Urgent production fix | `hotfix/security-patch-cve-2024` |
| `release/` | Release preparation | `release/v1.2.0` |
| `chore/` | Maintenance, refactoring, tooling | `chore/upgrade-node-20` |

Additional rules that keep branch names clean:

- Use lowercase letters, numbers, and hyphens only — no spaces or underscores.
- Keep names short but descriptive. `feature/user-auth` is better than `feature/implement-complete-user-authentication-flow`.
- Include a ticket or issue number when your team uses one: `feature/AUTH-142-login-form`.

## Common Pitfalls

- Forgetting to switch to the new branch before committing — your commits land on the wrong branch. Always run `git branch` or check your shell prompt before you start coding.
- Using `git branch -D` when `-d` would have warned you about unmerged work — double-check with `git log feature/name --not main` first.
- Ending up in detached HEAD after `git checkout <tag>` and then committing — create a branch immediately if you intend to keep that work.
- Long-lived branches that diverge significantly from `main` — they become painful to merge. Merge or rebase against `main` frequently.
- Branch names with uppercase letters or special characters — some filesystems and tools handle them inconsistently. Stick to lowercase and hyphens.

## Summary

- A branch is a lightweight file containing a single commit hash; creating or deleting one is nearly instant.
- `HEAD` normally points to the current branch; it points directly to a commit in detached HEAD state.
- Use `git switch -c <name>` to create and switch to a new branch in one step.
- Use `git branch -d` for safe deletion (refuses if unmerged) and `-D` for force deletion.
- Detached HEAD is harmless for browsing; run `git switch -c <name>` before committing in that state.
- Follow a consistent naming convention (`feature/`, `bugfix/`, `hotfix/`, `release/`) to communicate intent clearly.
