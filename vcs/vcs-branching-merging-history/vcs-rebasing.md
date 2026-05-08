# Rebasing

Rebasing moves the point where a branch diverged from its base, replaying each commit on top of a new starting point. The result is a straight, linear history with no merge commits — as if the feature branch had been created from the latest commit all along.

## What Rebasing Does

Consider a branch `feature/login` that was created from `main` at commit `C`. While you were working, `main` received two more commits (`F` and `G`).

Before rebase:

```text
          main
            |
A --- B --- C --- F --- G
             \
              D --- E
                    |
             feature/login
```

After `git rebase main` (run from `feature/login`):

```text
                         main
                           |
A --- B --- C --- F --- G --- D' --- E'
                              |
                       feature/login
```

Git detaches `D` and `E`, replays them one by one on top of `G`, and produces new commits `D'` and `E'`. They contain the same changes as `D` and `E` but have different parent commits and therefore different SHA-1 hashes. The originals `D` and `E` are no longer referenced by any branch and will eventually be garbage-collected.

Compare this to a merge, which would have produced a merge commit `M` with two parents:

```text
A --- B --- C --- F --- G --- M   (merge)
             \               /
              D ----------- E
```

Rebase gives you a clean line; merge preserves the exact history of when things happened.

## Performing a Rebase

```bash
# Switch to the feature branch
git switch feature/login

# Rebase onto main (move the base to the tip of main)
git rebase main
```

Git processes each commit in order. If everything applies cleanly, the operation completes silently.

### Rebase Conflicts

If a replayed commit conflicts with the new base, Git pauses and asks you to resolve it — exactly like a merge conflict.

```text
CONFLICT (content): Merge conflict in src/auth/LoginService.java
error: could not apply d4e8a2f... Add email validation
hint: Resolve all conflicts manually, then run "git rebase --continue"
```

The workflow:

```bash
# 1. Fix the conflict markers in the file
# 2. Stage the resolved file
git add src/auth/LoginService.java

# 3. Tell Git to continue replaying the remaining commits
git rebase --continue
```

If you want to give up and return to the state before the rebase started:

```bash
git rebase --abort
```

## Interactive Rebase

Interactive rebase lets you edit, reorder, combine, or delete commits before sharing your branch. It is the standard way to clean up a messy local history before opening a merge request.

```bash
# Open an editor listing the last 3 commits
git rebase -i HEAD~3
```

The editor shows something like:

```text
pick a1b2c3d Add login form HTML
pick b2c3d4e Add form validation
pick c3d4e5f Fix typo in error message

# Rebase d5e6f7a..c3d4e5f onto d5e6f7a (3 commands)
#
# Commands:
# p, pick   = use commit
# r, reword = use commit, but edit the commit message
# e, edit   = use commit, but stop for amending
# s, squash = use commit, meld into previous commit
# f, fixup  = like squash, but discard this commit's log message
# d, drop   = remove commit
```

### Actions Explained

| Action | What it does |
|---|---|
| `pick` | Keep the commit exactly as it is |
| `reword` | Keep the commit but open an editor to change its message |
| `squash` | Combine this commit into the one above it; Git merges both messages and asks you to edit the result |
| `fixup` | Same as `squash` but silently discards this commit's message |
| `drop` | Remove the commit entirely — its changes disappear |
| reorder | Move a line up or down in the list to change commit order |

### Example: Squash Three Commits Into One

You have three small commits that logically belong together:

```text
pick a1b2c3d Add login form HTML
squash b2c3d4e Add form validation
fixup c3d4e5f Fix typo in error message
```

Git replays them as a single commit. For `squash` it opens an editor showing both messages so you can write a clean combined message. For `fixup` it silently discards the second message.

Result:

```text
(single commit) Add login form HTML with validation
```

### Example: Drop a Commit

```text
pick a1b2c3d Add login form HTML
drop b2c3d4e Add debug logging   ← this will be removed
pick c3d4e5f Add form validation
```

The debug-logging commit and all its changes will be gone.

### Example: Reword a Commit Message

```text
reword a1b2c3d add form html   ← Git will open an editor for this message
pick b2c3d4e Add form validation
```

Git will pause and open your editor so you can fix the message.

## The Golden Rule of Rebasing

**Never rebase commits that have already been pushed to a shared branch.**

Rebasing rewrites commits, giving them new SHA-1 hashes. If teammates have already based work on the original commits, their histories will diverge from yours. When they try to pull, Git will see two different versions of the same logical commit and create a confusing mess of duplicated commits.

Safe uses of rebase:

- Rebasing a local feature branch onto an updated `main` before pushing for the first time.
- Interactive rebase to clean up local commits before opening a merge request.

Dangerous uses:

- Running `git rebase` on `main`, `develop`, or any branch others are tracking.
- Force-pushing a rebased branch (`git push --force`) to a shared branch without coordinating with the team.

If you must force-push a rebased branch to a remote (for example, after cleaning up your own merge-request branch that no one else has checked out), use the safer form:

```bash
# Refuses to overwrite if the remote has commits you do not have locally
git push --force-with-lease origin feature/login
```

## Merge vs Rebase: When to Use Each

| Situation | Recommended approach |
|---|---|
| Integrating a finished feature into `main` | Merge (preserves the integration point in history) |
| Updating a local feature branch with the latest `main` | Rebase (keeps history linear, easier to read) |
| Cleaning up commits before opening a merge request | Interactive rebase |
| On a shared branch that teammates are using | Merge (never rebase) |
| You want to see exactly when branches were integrated | Merge |
| You want a clean, linear history for `git log` readability | Rebase |

## Common Pitfalls

- Rebasing a branch that a colleague has already checked out — they will have to run `git pull --rebase` or manually reset, and may lose work if they are not careful.
- Forgetting `git rebase --continue` after resolving a conflict and running `git commit` instead — this creates an extra commit mid-rebase and corrupts the sequence.
- Dropping the wrong commit in interactive rebase — always re-read the commit list carefully before saving the editor.
- Running interactive rebase on too many commits (`HEAD~50`) — keep it focused; a large interactive rebase is error-prone.
- Using `--force` instead of `--force-with-lease` — `--force` silently overwrites any commits on the remote, including work pushed by others after your last fetch.

## Summary

- Rebase moves branch commits to a new base, replaying them one by one; the commits get new hashes but carry the same changes.
- Use `git rebase main` to bring a feature branch up to date with `main` cleanly, without a merge commit.
- Resolve rebase conflicts the same way as merge conflicts, then run `git rebase --continue`; use `git rebase --abort` to give up.
- `git rebase -i HEAD~N` opens an interactive editor for the last N commits; use `pick`, `squash`, `fixup`, `reword`, `drop`, and reordering to shape your history.
- The golden rule: never rebase commits already pushed to a shared branch — it rewrites history and forces others to reconcile diverging commits.
- Merge preserves the true history of integrations; rebase produces a cleaner linear log — choose based on your team's conventions and whether the branch is shared.
