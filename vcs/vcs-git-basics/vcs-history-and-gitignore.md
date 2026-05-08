# History and .gitignore

Two essential skills complete the Git basics: reading your project's history to understand what happened and when, and teaching Git which files to ignore so your repository stays clean of build artifacts, secrets, and editor noise.

## Viewing History with git log

`git log` is the primary command for reading commit history. It supports many formatting and filtering options.

### Full log

```bash
git log
```

```text
commit a3f9c12e7b2d58c3a1e4f6d7890abcde12345678
Author: Jane Doe <jane@example.com>
Date:   Mon Mar 10 14:22:05 2025 +0000

    Add rate limiting to the login endpoint

    Without rate limiting, the endpoint accepts unlimited requests per
    second. This commit adds a sliding-window rate limiter (5/min per IP).

commit 88b2c41e1a3d0f9c5b7e2a8d6f4c3b1a9e7d5c2
Author: Jane Doe <jane@example.com>
Date:   Fri Mar  7 09:15:30 2025 +0000

    Fix null pointer in user loader
```

Press `q` to quit the pager.

### Compact one-line format

```bash
git log --oneline
```

```text
a3f9c12 Add rate limiting to the login endpoint
88b2c41 Fix null pointer in user loader
3d4e5f6 Refactor database connection pooling
1a2b3c4 Add health check endpoint
7f8e9d0 Initial commit
```

Each line shows the abbreviated commit hash and the subject line. This is the format you will use most often for a quick overview.

### ASCII branch graph

```bash
git log --oneline --graph --all
```

```text
* a3f9c12 (HEAD -> main, origin/main) Add rate limiting to the login endpoint
* 88b2c41 Fix null pointer in user loader
| * c7d8e9f (feature/user-profile) Add avatar upload to profile page
| * b5a6c3d Add profile edit form
|/
* 3d4e5f6 Refactor database connection pooling
* 1a2b3c4 Add health check endpoint
* 7f8e9d0 Initial commit
```

`--graph` draws the branch and merge structure. `--all` includes all branches and remotes, not just the current one. This is the single most useful `git log` invocation for understanding a repository's shape.

## Filtering git log

### By author

```bash
git log --author="Jane"
```

Matches any author whose name or email contains "Jane". Accepts a regex.

### By date

```bash
git log --since="2 weeks ago"
git log --until="2024-01-01"
git log --since="2024-06-01" --until="2024-07-01"
```

### By commit message keyword

```bash
git log --grep="login"
```

Shows only commits whose message contains "login". Case-sensitive by default; add `-i` for case-insensitive.

### Combining filters

```bash
git log --author="Jane" --since="1 month ago" --oneline
```

Multiple filters are combined with AND logic by default.

### Commits that touched a specific file

```bash
git log --oneline -- path/to/file.py
```

The `--` separates the git options from the file path. Useful when you want to know the history of a single file.

## Inspecting a Single Commit with git show

`git show` displays the metadata and diff of a specific commit.

### Show the most recent commit

```bash
git show
# or equivalently:
git show HEAD
```

```text
commit a3f9c12e7b2d58c3a1e4f6d7890abcde12345678
Author: Jane Doe <jane@example.com>
Date:   Mon Mar 10 14:22:05 2025 +0000

    Add rate limiting to the login endpoint

diff --git a/auth/views.py b/auth/views.py
index 3a1f2b0..9d4e6c1 100644
--- a/auth/views.py
+++ b/auth/views.py
@@ -28,6 +28,10 @@ def login(request):
+    if rate_limiter.is_exceeded(request.ip):
+        return Response(status=429, headers={"Retry-After": "60"})
+
```

### Show a commit by hash

```bash
git show 88b2c41
```

You only need enough characters of the hash to make it unambiguous — typically 7.

### Show a commit relative to HEAD

```bash
git show HEAD~1   # one commit before HEAD
git show HEAD~2   # two commits before HEAD
git show HEAD^    # parent of HEAD (same as HEAD~1 for non-merge commits)
```

### Show only the commit metadata, no diff

```bash
git show --no-patch HEAD
git show --stat HEAD     # show a summary of files changed
```

## Ignoring Files with .gitignore

`.gitignore` tells Git to completely ignore certain files and directories. Ignored files will never appear in `git status` and can never be accidentally staged or committed.

### Creating a .gitignore file

```bash
touch .gitignore
```

`.gitignore` is a plain text file. Each line is a pattern.

### Pattern syntax

```text
# Exact filename — ignore this specific file anywhere in the repo
secret.key

# Extension — ignore all .log files anywhere in the repo
*.log

# Directory — ignore the entire node_modules directory (trailing slash is optional but clear)
node_modules/

# Path relative to .gitignore location — only ignore build/ at the repo root
/build/

# Negation — ignore all .log files EXCEPT important.log
*.log
!important.log

# Wildcard — ignore any file named config.local.* (e.g. config.local.js, config.local.yaml)
config.local.*

# Double wildcard — ignore all .pyc files in any subdirectory
**/__pycache__/
**/*.pyc
```

### .gitignore placement

A `.gitignore` file applies to the directory it is in and all its subdirectories.

```text
repo/
├── .gitignore          # Applies to the whole repo
├── src/
│   ├── .gitignore      # Applies only to src/ and below
│   └── app.py
└── docs/
    └── notes.md
```

If you have patterns specific to a subdirectory (e.g., generated HTML only inside `docs/`), put a `.gitignore` in that subdirectory. This keeps ignore rules co-located with the content they affect.

### Typical .gitignore for a Python project

```text
# Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
.Python
*.egg-info/
dist/
build/

# Virtual environments
.venv/
venv/
env/

# Environment variables
.env
.env.local

# Editor
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

### Global .gitignore — OS and editor files

Some files (`.DS_Store` on macOS, `.idea/` from JetBrains IDEs) should be ignored in every repository on your machine, not just one project. Put these in a global ignore file.

```bash
# Create the file
touch ~/.gitignore_global

# Tell Git to use it
git config --global core.excludesFile ~/.gitignore_global
```

Example `~/.gitignore_global`:

```text
# macOS
.DS_Store
.AppleDouble
.LSOverride

# JetBrains IDEs
.idea/

# VS Code
.vscode/

# Vim
*.swp
*.swo
*~

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini
```

Using a global file keeps project-level `.gitignore` files free of personal editor preferences — different team members use different editors.

## Stopping Tracking of Already-Committed Files

`.gitignore` only prevents **untracked** files from being tracked. If you committed a file by accident (e.g., `.env`), adding it to `.gitignore` will not stop Git from tracking it. You must remove it from the index first:

```bash
# Remove from Git's tracking without deleting the file from disk
git rm --cached .env

# For a directory
git rm -r --cached node_modules/

# Then commit the removal
git add .gitignore
git commit -m "Stop tracking .env and add to .gitignore"
```

After this, `.env` will appear in `.gitignore`, Git will no longer track it, but the file still exists on your disk.

## Common Pitfalls

- Adding `.env` or `*.key` to `.gitignore` after already committing them — the files remain in history; use `git rm --cached` to stop tracking them and then rotate the secrets, since they are still visible in old commits.
- Using `git log` without `--all` and missing branches — plain `git log` only shows the current branch's ancestry; `--all` shows everything.
- Writing a `.gitignore` pattern without a leading `/` and being surprised it matches everywhere — patterns without `/` match in any subdirectory; prefix with `/` to anchor to the `.gitignore` file's location.
- Putting editor-specific patterns (`.idea/`, `.vscode/`) in the project `.gitignore` — these belong in `~/.gitignore_global` so they do not impose your editor choice on teammates.
- Confusing `HEAD~2` and `HEAD^2` — `HEAD~2` means "two commits back in the first-parent chain"; `HEAD^2` means "the second parent of HEAD" (relevant for merge commits only).

## Summary

- `git log` shows full commit history; `--oneline` condenses it to one line per commit; `--oneline --graph --all` shows the full branch topology.
- Filter with `--author`, `--since`, `--until`, and `--grep` to find specific commits quickly.
- `git show <hash>` displays a commit's metadata and diff; use `HEAD~N` to refer to commits relative to the current position.
- `.gitignore` uses patterns to exclude files: exact names, `*.extension`, `directory/`, negation with `!`, and double wildcards `**`.
- Place `.gitignore` in the directory whose scope it should cover; put personal editor/OS patterns in `~/.gitignore_global`.
- Use `git rm --cached <file>` to stop tracking a file that was already committed, then add it to `.gitignore` and commit the change.
