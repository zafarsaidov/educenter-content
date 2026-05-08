# Merging and Conflicts

Merging is how you bring the work from one branch into another. Git handles two fundamentally different situations — fast-forward merges and three-way merges — and understanding the difference helps you predict exactly what will happen and why conflicts arise.

## Fast-Forward Merge

A fast-forward merge happens when the target branch (for example `main`) has received no new commits since the feature branch was created. There is nothing to reconcile — Git simply moves the `main` pointer forward to the tip of the feature branch.

Before merge:

```text
main
  |
  A --- B --- C
               \
                D --- E
                      |
               feature/login
```

After `git merge feature/login` (fast-forward):

```text
                    main
                      |
  A --- B --- C --- D --- E
                          |
                   feature/login
```

No new commit was created. `main` just moved forward.

## Three-Way Merge

A three-way merge is necessary when both branches have new commits since they diverged. Git identifies three commits: the common ancestor, the tip of the current branch, and the tip of the branch being merged. It combines the changes and creates a new **merge commit** with two parents.

Before merge:

```text
      main
        |
A --- B --- C --- F --- G
        \
         D --- E
               |
         feature/login
```

After `git merge feature/login`:

```text
                       main
                         |
A --- B --- C --- F --- G --- M
        \                   /
         D -------- E ------
                    |
             feature/login
```

`M` is the merge commit. It has two parents (`G` and `E`) and its message defaults to `Merge branch 'feature/login'`.

## Performing a Merge

```bash
# Switch to the branch you want to merge INTO
git switch main

# Merge the feature branch
git merge feature/login

# Force a merge commit even when fast-forward is possible
git merge --no-ff feature/login
```

### Why Use `--no-ff`?

With `--no-ff` (no fast-forward), Git always creates a merge commit even if the history would allow a fast-forward. This keeps a visible record in the log that a feature branch existed and was integrated at a specific point — useful for auditing and reverting an entire feature in one step.

```bash
# See the difference in log output
git log --oneline --graph
```

## Merge Conflicts

A conflict occurs when both branches have changed the same lines of the same file, and Git cannot decide automatically which version to keep.

### What Triggers a Conflict

- Both branches edited the same line(s) differently.
- One branch deleted a file that the other branch modified.
- Both branches added a file with the same name but different content.

### Conflict Markers

When Git detects a conflict, it pauses the merge and annotates the affected file with markers:

```text
<<<<<<< HEAD
    return user.getName();
=======
    return user.getFullName();
>>>>>>> feature/login
```

- Everything between `<<<<<<< HEAD` and `=======` is the version from your current branch.
- Everything between `=======` and `>>>>>>> feature/login` is the version from the branch being merged.
- You must decide what the final content should be, remove all three marker lines, and save.

### Resolving Conflicts Manually

```bash
# 1. See which files have conflicts
git status

# 2. Open the file, find the markers, edit to the desired result
#    For example, keep both lines:
#    return user.getName() + " " + user.getFullName();

# 3. Mark the file as resolved
git add src/UserService.java

# 4. Complete the merge with a commit
git commit
```

Git pre-fills the commit message with merge details; you can keep it or customize it.

### Using a Merge Tool

If you prefer a side-by-side visual interface:

```bash
# Opens the configured diff tool for each conflicted file
git mergetool
```

Common merge tools and how to configure them:

```bash
# VS Code
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# vimdiff (built into most systems)
git config --global merge.tool vimdiff

# meld (Linux GUI)
git config --global merge.tool meld
```

After resolving with the tool, the `.orig` backup files (e.g., `UserService.java.orig`) are left in the directory. You can delete them or configure Git to clean them up automatically:

```bash
git config --global mergetool.keepBackup false
```

## Aborting a Merge

If you started a merge and want to get back to where you were before:

```bash
git merge --abort
```

This restores your working tree and index to the state before `git merge` was run. It only works while the merge is still in progress (i.e., before you run `git commit`).

## Cleaning Up After a Merge

Once a feature branch has been merged and is no longer needed, delete it to keep the branch list tidy:

```bash
# Delete local branch
git branch -d feature/login

# Delete the corresponding remote branch
git push origin --delete feature/login
```

## Common Pitfalls

- Committing conflict markers by accident — always search the file for `<<<<<<<` before running `git add`.
- Resolving a conflict by keeping only one side without reading both — take the time to understand both changes and produce a correct combined result.
- Ignoring `.orig` backup files left by `git mergetool` — they clutter the repo and can end up committed if you run `git add .` carelessly.
- Forgetting `--no-ff` on feature branches when the team wants a clear history of integrations — agree on the convention with your team upfront.
- Resolving a conflict and then forgetting to run `git add` before `git commit` — Git will warn you, but it is easy to miss in the output.

## Summary

- A fast-forward merge moves a pointer forward with no new commit; it happens when the target branch has no new commits since the feature diverged.
- A three-way merge creates a merge commit with two parents; it happens when both branches have diverged.
- Use `git merge --no-ff` to force a merge commit and preserve a clear record of the branch integration.
- Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) show both versions; edit the file to the desired result, then `git add` it and `git commit`.
- `git mergetool` opens a visual three-way diff; configure it once with `git config --global merge.tool`.
- `git merge --abort` safely cancels an in-progress merge and restores the previous state.
- Delete merged branches with `git branch -d` locally and `git push origin --delete` on the remote.
