# Staging and Commits

The staging area is what makes Git commits precise and intentional rather than "save everything I touched." This lesson explains how to stage files selectively, how to use patch mode to stage individual changes within a file, how to write commit messages that will still make sense six months from now, and what a commit actually contains under the hood.

## Staging Files

### Stage a specific file

```bash
git add README.md
git add src/app.py
```

### Stage everything in the current directory

```bash
git add .
```

This stages all new and modified files under the current directory. If you run it from the repo root it stages everything. Be careful — you may accidentally stage files you did not intend to commit (like local config files). Always follow `git add .` with `git status` or `git diff --staged` to verify.

### Stage a directory

```bash
git add src/
```

All changes inside `src/` are staged.

## git add -p — Patch Mode (Interactive Staging)

Patch mode lets you review your changes hunk by hunk and decide for each one whether to include it in the next commit. This is how you build a clean, focused commit even when you have changed many things at once.

```bash
git add -p
```

Git splits your changes into hunks and presents each one:

```text
diff --git a/app.py b/app.py
index 3a1f2b0..9d4e6c1 100644
--- a/app.py
+++ b/app.py
@@ -12,6 +12,7 @@ def process():
     data = load()
+    validate(data)
     return transform(data)

Stage this hunk [y,n,q,a,d,s,?]?
```

### Patch mode options

| Key | Meaning |
|-----|---------|
| `y` | Yes — stage this hunk |
| `n` | No — skip this hunk (leave it unstaged) |
| `s` | Split — break this hunk into smaller hunks |
| `e` | Edit — open the hunk in an editor for manual adjustments |
| `q` | Quit — stop staging, leave remaining hunks untouched |
| `a` | All — stage this hunk and all remaining hunks in this file |
| `d` | Skip — skip this hunk and all remaining hunks in this file |
| `?` | Help — show all options |

### Why patch mode matters

Suppose you fixed a bug and also refactored a function in the same file. With `git add -p` you can commit the bug fix alone (with a message like `Fix null pointer in process()`) and then commit the refactor separately (with a message like `Refactor transform() for clarity`). The result is a clean, readable history that is easy to review and bisect.

## Why the Staging Area Matters

The staging area gives you a deliberate checkpoint between "I made changes" and "I recorded a change." It lets you:

- **Include only what belongs together** — a commit that fixes one bug is easier to review, revert, and understand than a commit that fixes a bug, adds a feature, and reformats code.
- **Review before committing** — `git diff --staged` shows you exactly what will be in the commit. Read it before you commit.
- **Stage partial file changes** — with `git add -p`, a single file's changes can be split across multiple commits.

## Creating Commits

### Commit with an inline message

```bash
git commit -m "Add input validation to process()"
```

### Commit and open the editor for a full message

```bash
git commit
```

Git opens the editor you configured in `core.editor`. Write your message, save, and close the file. Git creates the commit.

### Commit an empty working tree (trigger or marker commits)

```bash
git commit --allow-empty -m "chore: trigger CI pipeline"
```

Useful when you need to re-trigger a CI job without making a code change.

## Writing Good Commit Messages

A good commit message is a gift to your future self and your teammates. Follow these rules:

### The seven rules of a great commit message

1. **Separate subject from body with a blank line**
2. **Limit the subject line to 72 characters**
3. **Use the imperative mood** — "Fix bug" not "Fixed bug" or "Fixes bug"
4. **Do not end the subject line with a period**
5. **Use the body to explain what and why, not how** — the code shows how
6. **Wrap the body at 72 characters**
7. **Reference issues or merge requests when relevant**

### Example of a well-structured commit message

```text
Add rate limiting to the login endpoint

Without rate limiting, the login endpoint accepts unlimited requests
per second, making brute-force attacks trivial. This commit adds a
sliding-window rate limiter that allows 5 attempts per IP per minute.

Attempts that exceed the limit receive a 429 response with a
Retry-After header.

Closes #142
```

### Subject line formats

**Bare imperative (simple projects):**
```text
Fix null pointer in user loader
Add health check endpoint
Remove deprecated API methods
```

**Conventional Commits (team projects — covered in Section 4):**
```text
fix: null pointer in user loader
feat: add health check endpoint
chore: remove deprecated API methods
```

## Anatomy of a Commit

Every Git commit is a data object stored in `.git/objects/`. You can inspect any commit with:

```bash
git cat-file -p HEAD
```

```text
tree   4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent a3f9c12e7b2d58c3a1e4f6d7890abcde12345678
author  Jane Doe <jane@example.com> 1711468800 +0000
committer Jane Doe <jane@example.com> 1711468800 +0000

Add rate limiting to the login endpoint
```

| Field | Meaning |
|-------|---------|
| `tree` | SHA-1 hash of the top-level directory snapshot (a tree object listing all files) |
| `parent` | SHA-1 hash of the previous commit. Merge commits have two or more parents. The very first commit has no parent. |
| `author` | Who originally wrote the change (name, email, Unix timestamp, timezone offset) |
| `committer` | Who applied the commit. Usually the same as author, but differs after a rebase or `git am`. |
| message | The commit message you wrote |

The commit's own SHA-1 hash is computed from all of this content. Changing even one character — the message, the timestamp, a parent hash — produces a completely different hash. This is why Git history is tamper-evident.

## Common Pitfalls

- Using `git add .` without reviewing what gets staged — always run `git status` or `git diff --staged` after staging.
- Writing subject lines in past tense ("Fixed the bug") — use imperative mood ("Fix the bug") to match the style of Git's own generated messages.
- Putting everything in one massive commit "because it all belongs to the same feature" — smaller commits are easier to review, easier to revert, and easier to understand in `git log`.
- Forgetting that `git commit -m` only sets the subject line — for a multi-line message, open the editor with plain `git commit`.
- Conflating author and committer — `author` is who wrote the change, `committer` is who recorded it. They differ after rebasing others' work.

## Summary

- `git add <file>` stages a specific file; `git add .` stages everything in the current directory.
- `git add -p` enters patch mode: review and selectively stage individual hunks within files using `y`, `n`, `s`, `q`, and other keys.
- The staging area exists to let you build clean, focused commits — use it intentionally rather than staging everything at once.
- `git commit -m "message"` creates a commit with an inline subject; plain `git commit` opens your editor for a multi-line message.
- Good commit messages use imperative mood, keep the subject under 72 characters, and explain *why* in the body, not *how*.
- A commit object contains: a tree hash, parent hash(es), author, committer, timestamps, and the message — all hashed together into a unique SHA-1.
