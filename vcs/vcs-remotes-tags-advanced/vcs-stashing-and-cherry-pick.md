# Stashing and Cherry-picking

Sometimes you need to switch context instantly — drop what you are doing and move to another branch — without making a half-finished commit. Git provides two tools for situations like this: `git stash` for temporarily shelving work, and `git cherry-pick` for surgically applying a specific commit from one branch to another.

## The Problem Stash Solves

Imagine you are halfway through a feature when a critical production bug is reported. You need to switch to `main`, create a hotfix branch, and fix the bug immediately. But Git will refuse to switch branches if you have uncommitted changes that conflict with the target branch.

You have two bad options and one good one:

- **Bad**: commit the half-finished work — now your history contains a broken commit.
- **Bad**: discard the changes — you lose your work.
- **Good**: stash the changes, fix the bug, then restore your work.

## Basic Stash Usage

### Saving Changes to the Stash

```bash
git stash
```

This saves all **tracked** modified files and any **staged** changes to a stack called the stash, then restores your working tree to the last commit. The directory looks clean — as if you had not made any changes.

### Stashing with a Description

A bare `git stash` creates an entry named after the commit it was stashed from, which is not very readable. Add a message:

```bash
git stash push -m "WIP: add user avatar upload endpoint"
```

Now the stash entry has a human-readable label.

### Including Untracked Files

By default, `git stash` ignores untracked files (new files you have not yet run `git add` on). To include them:

```bash
git stash -u
```

Or the long form:

```bash
git stash push --include-untracked -m "WIP: new config parser, untracked files included"
```

To also include files matching `.gitignore` patterns (rarely needed):

```bash
git stash push --all
```

## Managing the Stash

### Viewing All Stashes

```bash
git stash list
```

Sample output:

```text
stash@{0}: On feature/avatar: WIP: add user avatar upload endpoint
stash@{1}: On main: Quick spike for export feature
stash@{2}: WIP on feature/login: e4f5a6b Add login form skeleton
```

`stash@{0}` is always the most recently created stash. The number increments as you go back in time.

### Applying the Most Recent Stash

```bash
git stash pop
```

`pop` applies `stash@{0}` to your working tree and removes it from the stash list. If there are conflicts, Git will report them — resolve them the same way you resolve merge conflicts, then remove the stash entry manually:

```bash
git stash drop stash@{0}
```

### Applying a Specific Stash

```bash
git stash apply stash@{2}
```

`apply` restores the stash but keeps it in the list — useful when you want to apply the same stash to multiple branches for comparison.

### Deleting Stashes

Delete a single entry:

```bash
git stash drop stash@{0}
```

Delete all stashes at once:

```bash
git stash clear
```

Be careful with `git stash clear` — there is no undo.

## Cherry-picking

Cherry-picking means taking a single commit from one branch and applying its changes as a new commit on your current branch. The original commit is not moved — a new commit with a new hash is created.

### When to Use Cherry-pick

The classic scenario is **backporting a bug fix**. Your team has already shipped `v2.0`, but a critical bug exists in both `main` (v3 development) and the `release/v2` branch. You fix it on `main` and then cherry-pick the fix onto `release/v2`.

```text
main:        A --- B --- C(fix) --- D
                          ↓ cherry-pick
release/v2:  X --- Y --- C'(fix)
```

`C'` has the same changes as `C` but a different commit hash.

### Cherry-picking a Single Commit

Find the hash of the commit you want:

```bash
git log --oneline main
```

```text
d8e2f1a Fix null pointer in auth module
3c4b5a6 Add export endpoint
...
```

Switch to the target branch and cherry-pick:

```bash
git switch release/v2
git cherry-pick d8e2f1a
```

Git applies the diff from that commit as a new commit on `release/v2`.

### Cherry-picking a Range of Commits

```bash
git cherry-pick A..B
```

This cherry-picks all commits **after** `A` up to and including `B` (A is **exclusive**, B is **inclusive**). To include A:

```bash
git cherry-pick A^..B
```

### Staging Without Committing

```bash
git cherry-pick --no-commit d8e2f1a
```

Instead of creating a commit immediately, Git stages the changes. This is useful when you want to combine multiple cherry-picks into a single commit, or inspect and modify the changes before committing:

```bash
git cherry-pick --no-commit d8e2f1a e9f0a1b
git commit -m "Backport auth fixes to release/v2"
```

### Resolving Cherry-pick Conflicts

If the target branch has diverged enough, the cherry-pick may produce conflicts:

```text
error: could not apply d8e2f1a... Fix null pointer in auth module
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue"
```

Resolve the conflict in the file, then:

```bash
git add src/auth/handler.go
git cherry-pick --continue
```

To abort and return to where you were before:

```bash
git cherry-pick --abort
```

## Patch Files

Patch files are a lightweight, portable way to share changes — useful when you cannot directly access a remote or need to send a fix by email.

### Creating a Patch from a Commit

```bash
git format-patch -1 d8e2f1a
```

This creates a file named something like `0001-Fix-null-pointer-in-auth-module.patch` in the current directory. The `-1` means "one commit". To create patches for the last three commits:

```bash
git format-patch -3
```

### Applying a Patch File

```bash
git apply 0001-Fix-null-pointer-in-auth-module.patch
```

This applies the diff to your working tree but does not create a commit. To apply and commit (preserving the original author and message):

```bash
git am 0001-Fix-null-pointer-in-auth-module.patch
```

`git am` (apply mailbox) is designed for applying `format-patch` output and produces a proper commit with the original metadata intact.

## Cherry-pick vs Merge: When to Use Which

| Situation                                          | Use           |
|----------------------------------------------------|---------------|
| Backport a single bug fix to a release branch      | cherry-pick   |
| Apply a specific commit from a spike/experiment    | cherry-pick   |
| Integrate an entire feature branch into `main`     | merge         |
| Catch up a long-running branch with `main`         | rebase/merge  |
| Apply two or three related fixes to a support branch | cherry-pick |

Cherry-pick is a scalpel — precise but narrow. When you need to integrate sustained, ongoing work between branches, merge or rebase is the right tool. Overusing cherry-pick can duplicate commits and make history harder to understand.

## Common Pitfalls

- Running `git stash` and forgetting about stashes for weeks — the stash is not visible in normal `git log` output; use `git stash list` regularly and clean up entries you no longer need.
- Using `git stash clear` without checking `git stash list` first — all stashed work is permanently deleted.
- Cherry-picking commits that were already merged into the target branch — this creates duplicate commits with different hashes, which can cause confusion and double-application of changes.
- Cherry-picking a merge commit without the `-m` flag — Git does not know which parent to use; you must specify it: `git cherry-pick -m 1 <merge-commit-hash>`.
- Relying on cherry-pick instead of a proper shared branch — if the same fix needs to live in many places, consider whether the branch structure itself is the problem.
- Forgetting `git cherry-pick --abort` after a conflict you cannot resolve — leaving a cherry-pick in progress blocks other Git operations.

## Summary

- `git stash` shelves tracked changes and restores a clean working tree; `git stash -u` also includes untracked files.
- `git stash push -m "description"` creates a readable stash entry; `git stash list` shows all entries.
- `git stash pop` applies and removes the most recent stash; `git stash apply stash@{N}` applies a specific one without removing it.
- `git stash drop stash@{N}` removes a single entry; `git stash clear` removes all entries permanently.
- `git cherry-pick <hash>` applies a commit's changes as a new commit on the current branch.
- `git cherry-pick A..B` cherry-picks a range of commits; `--no-commit` stages changes without committing.
- Cherry-pick conflicts are resolved the same way as merge conflicts, then continued with `git cherry-pick --continue`.
- `git format-patch -1 <hash>` exports a commit as a portable `.patch` file; `git am <file.patch>` applies it with full metadata.
- Use cherry-pick for isolated, targeted backports; use merge or rebase for integrating full branches.
