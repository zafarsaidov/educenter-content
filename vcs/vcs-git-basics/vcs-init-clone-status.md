# Init, Clone, and Status

Every Git workflow begins with either creating a new repository or obtaining a copy of an existing one. This lesson covers `git init` and `git clone` for setting up repositories, then dives into `git status` and `git diff` — the two commands you will run dozens of times a day to understand the current state of your work.

## git init — Creating a New Repository

`git init` turns any ordinary directory into a Git repository by creating a hidden `.git/` subdirectory inside it.

```bash
mkdir my-project
cd my-project
git init
```

```text
Initialized empty Git repository in /home/you/my-project/.git/
```

You can also initialise a directory that already contains files:

```bash
cd existing-project
git init
```

### What lives inside .git/

```text
.git/
├── HEAD          # Points to the currently checked-out branch (e.g. ref: refs/heads/main)
├── config        # Local repository configuration (git config --local writes here)
├── description   # Used by GitWeb — not relevant for normal use
├── hooks/        # Scripts that run on Git events (covered in Section 4)
├── info/
│   └── exclude   # Per-repo ignore rules that are not committed (like a private .gitignore)
├── objects/      # All content Git has ever stored: blobs, trees, commits
│   ├── pack/     # Packed objects for efficiency
│   └── info/
└── refs/
    ├── heads/    # One file per local branch, containing the tip commit hash
    └── tags/     # One file per tag
```

You should never edit files inside `.git/` by hand. Understanding what is there helps you reason about what Git commands actually do.

## git clone — Downloading an Existing Repository

`git clone` copies a remote repository to your local machine. It downloads the full history, not just the latest snapshot.

```bash
git clone https://gitlab.com/yourgroup/yourrepo.git
```

This creates a directory named `yourrepo/` in your current location. Git automatically configures a remote named `origin` pointing back to the URL you cloned from.

### Cloning to a custom directory name

```bash
git clone https://gitlab.com/yourgroup/yourrepo.git my-local-name
```

### Shallow clone for CI pipelines

A **shallow clone** downloads only the most recent N commits instead of the full history. This makes CI pipelines significantly faster when you only need to build and test the latest code.

```bash
git clone --depth 1 https://gitlab.com/yourgroup/yourrepo.git
```

The `--depth 1` flag fetches only the latest commit. You cannot traverse history beyond that depth unless you deepen the clone later with `git fetch --unshallow`.

### What clone sets up for you

```bash
git clone git@gitlab.com:yourgroup/yourrepo.git
cd yourrepo
git remote -v
```

```text
origin  git@gitlab.com:yourgroup/yourrepo.git (fetch)
origin  git@gitlab.com:yourgroup/yourrepo.git (push)
```

The local `main` branch is automatically set to track `origin/main`.

## git status — Understanding the Three Areas

`git status` shows you exactly where each changed file sits across the three areas: working tree, staging area, and repository.

### Clean working tree

```bash
git status
```

```text
On branch main
nothing to commit, working tree clean
```

This means the working tree matches the latest commit exactly.

### After creating or modifying files

```bash
echo "Hello" > hello.txt
git status
```

```text
On branch main

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        hello.txt

nothing added to commit but untracked files present (use "git add" to track)
```

Now stage the file and check again:

```bash
git add hello.txt
git status
```

```text
On branch main

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   hello.txt
```

Now modify the file after staging:

```bash
echo "World" >> hello.txt
git status
```

```text
On branch main

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   hello.txt

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   hello.txt
```

This illustrates a key point: `hello.txt` appears in two sections simultaneously. The version in the staging area (`new file: hello.txt`) is the first `echo "Hello"` version. The working tree version has the additional line `"World"` but is not yet staged.

## git diff — Seeing the Actual Changes

`git status` tells you *which* files changed. `git diff` shows you *what* changed.

### Unstaged changes (working tree vs staging area)

```bash
git diff
```

```text
diff --git a/hello.txt b/hello.txt
index e965047..8ab686e 100644
--- a/hello.txt
+++ b/hello.txt
@@ -1 +1,2 @@
 Hello
+World
```

Lines prefixed with `+` were added; lines prefixed with `-` were removed.

### Staged changes (staging area vs last commit)

```bash
git diff --staged
```

```text
diff --git a/hello.txt b/hello.txt
new file mode 100644
index 0000000..e965047
--- /dev/null
+++ b/hello.txt
@@ -0,0 +1 @@
+Hello
```

`git diff --staged` is what you would be committing if you ran `git commit` right now. It is a good habit to review this before every commit.

## Practical Workflow Summary

```bash
# 1. Start a project
git init my-project
cd my-project

# 2. Create some files
echo "# My Project" > README.md

# 3. Check what Git sees
git status

# 4. Inspect the actual changes
git diff

# 5. Stage what you want in the commit
git add README.md

# 6. Verify what is staged
git status
git diff --staged

# 7. Commit
git commit -m "Add README"
```

## Common Pitfalls

- Editing files inside `.git/` — this will corrupt your repository; never touch that directory manually.
- Confusing `git diff` and `git diff --staged` — plain `git diff` shows **unstaged** changes only; use `--staged` to see what is about to be committed.
- Running `git clone --depth 1` and then trying to access old commits — a shallow clone truncates history; use `git fetch --unshallow` if you need the full history later.
- Expecting `git status` to show binary files differently — it does not diff binary content, it just marks the file as modified; use `git diff --stat` to see a summary.
- Forgetting that `git init` in the wrong directory creates a hidden `.git/` that is hard to notice — always check which directory you are in before running `git init`.

## Summary

- `git init` creates a `.git/` directory, turning any folder into a Git repository. The key files inside are `HEAD`, `config`, `objects/`, and `refs/`.
- `git clone <url>` downloads the full history and sets up `origin` automatically. Use `--depth 1` for fast shallow clones in CI.
- `git status` shows the state of all files across the three areas: untracked, unstaged changes, and staged changes ready to commit.
- `git diff` shows unstaged changes (working tree vs staging area); `git diff --staged` shows staged changes (staging area vs last commit).
- A file can appear in both "Changes to be committed" and "Changes not staged" simultaneously — the staged and working-tree versions can differ.
