# Undoing Changes

Git gives you several distinct tools for undoing work, each suited to a different situation. Choosing the right one depends on where the change lives (working tree, staging area, or history) and whether the commits involved have been shared with others.

## Quick Reference

| Situation | Command | Safe on shared branches? |
|---|---|---|
| Discard unsaved edits in a file | `git restore <file>` | n/a (local only) |
| Unstage a file (keep changes) | `git restore --staged <file>` | n/a (local only) |
| Fix the last commit message or add a file | `git commit --amend` | No — only before pushing |
| Undo a commit without rewriting history | `git revert <hash>` | Yes |
| Move HEAD back, keep changes staged | `git reset --soft HEAD~1` | No — only on local work |
| Move HEAD back, keep changes in working tree | `git reset --mixed HEAD~1` | No — only on local work |
| Move HEAD back and discard all changes | `git reset --hard HEAD~1` | No — destructive |
| Recover from a bad reset | `git reflog` | Yes |

## Discarding Working Tree Changes

`git restore <file>` throws away all unsaved edits in the specified file and replaces it with the version from the last commit (or from the staging area if changes are staged).

```bash
# Discard all uncommitted changes to a single file
git restore src/auth/LoginService.java

# Discard all uncommitted changes in the entire working tree
git restore .
```

This operation is **not reversible**. Once you run it, the edits are gone — they are not in the staging area, not in the stash, and not in the reflog. Only use it when you are certain you do not need those changes.

## Unstaging a File

`git restore --staged <file>` moves a file back from the staging area to the working tree. The changes in the file are kept; only the staging is undone.

```bash
# Stage two files by mistake
git add LoginService.java Config.java

# Unstage one of them (keep the edits in the file)
git restore --staged Config.java

# Verify: Config.java appears under "Changes not staged for commit"
git status
```

This is safe and reversible — your edits are still there.

## Amending the Last Commit

`git commit --amend` rewrites the most recent commit. You can change its message, add forgotten files, or fix a typo — all without creating a new commit.

```bash
# Change only the commit message
git commit --amend -m "feat(auth): add email validation to login form"

# Add a forgotten file to the last commit
git add src/auth/LoginService.java
git commit --amend --no-edit   # keep the existing message
```

Amend works by creating a brand-new commit with a different SHA-1 and abandoning the old one. If you have already pushed the original commit to a remote, everyone else has the old SHA-1 in their history. Amending and force-pushing will cause divergence for them.

**Rule: only amend before pushing.**

## Reverting a Commit

`git revert <hash>` creates a new commit whose changes are the exact inverse of the target commit. The original commit stays in the history — nothing is rewritten.

```bash
# Find the hash of the commit you want to undo
git log --oneline

# Revert it (Git opens an editor for the revert commit message)
git revert a3f9c21

# Revert without opening an editor (accept the default message)
git revert --no-edit a3f9c21
```

Before revert:

```text
A --- B --- C --- D
                  |
                 main
```

After `git revert C`:

```text
A --- B --- C --- D --- C'
                        |
                       main
```

`C'` undoes everything `C` introduced. `C` itself is still visible in the log.

Because `revert` adds a commit rather than removing one, it is the correct tool for undoing work on shared branches like `main` or `develop`.

## Resetting History

`git reset` moves `HEAD` (and the current branch pointer) back to a previous commit. Unlike `revert`, it removes commits from the branch's visible history. There are three modes, differing in what happens to the changes from the removed commits.

### `--soft`: Keep Changes Staged

```bash
git reset --soft HEAD~1
```

The last commit disappears from history, but its changes are moved to the staging area, ready to commit again. Use this when you want to redo the commit with a different message or combined with other changes.

```text
Before: A --- B --- C   (HEAD at C, staging clean)
After:  A --- B         (HEAD at B, changes from C are staged)
```

### `--mixed` (Default): Keep Changes in Working Tree

```bash
git reset HEAD~1
# equivalent:
git reset --mixed HEAD~1
```

The last commit disappears, and its changes land in the working tree as unstaged edits. Use this when you want to re-stage selectively or make further edits before recommitting.

### `--hard`: Discard All Changes

```bash
git reset --hard HEAD~1
```

The last commit disappears and its changes are gone from both the staging area and the working tree. **This is destructive and cannot be undone through normal Git operations.**

Use `--hard` only for commits you are absolutely certain you do not need — for example, when reverting an experiment that went completely wrong on a branch no one else has seen.

### Resetting to a Specific Commit

```bash
# Go back to the state of a specific commit, discarding everything after it
git reset --hard a3f9c21
```

## The Reflog: Your Safety Net

The reflog records every position `HEAD` has been at, including commits created during rebases, amends, and resets. Even after a `--hard` reset, the original commit is usually still recoverable for at least 30 days (Git's default reflog expiry).

```bash
# Show the reflog for HEAD
git reflog

# Sample output:
# a3f9c21 HEAD@{0}: reset: moving to HEAD~1
# b4d8e32 HEAD@{1}: commit: Add password strength meter
# c5e9f43 HEAD@{2}: commit: Add login form HTML
```

Each entry has a hash and a `HEAD@{N}` reference. To recover commits you accidentally reset away:

```bash
# Option 1: reset to the lost commit
git reset --hard b4d8e32

# Option 2: create a new branch pointing at the lost commit
git switch -c recovery/lost-work b4d8e32
```

The reflog is local — it is not pushed to the remote. It is your personal undo history.

## Decision Guide

```text
Where is the problem?
│
├─ In the working tree (not yet staged)?
│     └─ git restore <file>
│
├─ In the staging area?
│     └─ git restore --staged <file>
│
├─ In the last commit, not yet pushed?
│     └─ git commit --amend
│
├─ In a commit that has been pushed to a shared branch?
│     └─ git revert <hash>
│
└─ In local commits (not pushed), want to restructure history?
      ├─ Keep changes staged:        git reset --soft HEAD~N
      ├─ Keep changes in work tree:  git reset --mixed HEAD~N
      └─ Throw changes away:         git reset --hard HEAD~N
```

## Common Pitfalls

- Running `git restore .` without realizing there are hours of unsaved work — use `git diff` first to review exactly what will be lost.
- Amending a commit that was already pushed and then running `git push --force` on a shared branch — teammates will have diverging history.
- Using `git reset --hard` when `git revert` was the right call — on a shared branch, `reset --hard` plus a force-push causes conflicts for everyone else.
- Forgetting that `git revert` may itself produce a conflict if subsequent commits touched the same code — resolve it the same way as a merge conflict.
- Not knowing about `git reflog` and believing a `--hard` reset is permanent — always check the reflog before concluding data is gone.
- Reverting a merge commit without the `-m` flag — Git does not know which parent to treat as the mainline; you must specify it with `-m 1`.

## Summary

- `git restore <file>` discards working tree edits; `git restore --staged <file>` unstages without losing edits.
- `git commit --amend` rewrites the last commit — safe only before pushing.
- `git revert <hash>` creates a new undo commit and is the only safe option on shared branches.
- `git reset --soft` keeps changes staged, `--mixed` keeps them in the working tree, `--hard` discards them entirely — use only on unpushed local commits.
- `git reflog` records every `HEAD` position and is your recovery tool after any destructive operation.
- The decision rule: use `restore` for file-level undo, `revert` on public branches, `reset` only on local unpushed work.
