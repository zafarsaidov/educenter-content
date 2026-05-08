# What is Version Control

Version control is the practice of tracking and managing changes to files over time so that you can recall specific versions later, collaborate with others without overwriting each other's work, and recover from mistakes. Git is today's dominant version control system, and understanding its core model will make every command you learn feel logical rather than magical.

## Why Version Control Matters

Without version control, teams email zip files, name documents `report_final_v3_REAL.docx`, and lose hours reconstructing what changed when a bug appeared. Version control solves four concrete problems:

- **Collaboration** — multiple people can work on the same codebase simultaneously without stepping on each other's changes.
- **History** — every change is recorded with who made it, when, and why. You can answer "what changed last Tuesday?" in seconds.
- **Backup and recovery** — the full history of a project is stored in the repository. If you delete a file or introduce a bug, you can roll back.
- **Rollback** — deploying a broken release? Revert to the last known-good commit in one command.

## Types of Version Control Systems

### Local VCS

The earliest form. A simple database on your machine stores patches (diffs) between file versions. RCS (Revision Control System) is the classic example. It works fine for a single developer but makes collaboration impossible — the history lives only on one machine.

```text
Developer's Machine
└── RCS database (local patches only)
```

### Centralised VCS (CVCS)

A single server holds the full versioned history. Every developer checks out a snapshot from that server and commits changes back to it. Subversion (SVN) and CVS are the most widely used examples.

```text
Central Server (SVN)
    ↑  ↓  ↑  ↓
Dev A  Dev B  Dev C
```

Advantages: simple administration, fine-grained access control.
Disadvantages: the server is a single point of failure. If it goes down, nobody can commit or access history. If the disk is not backed up and it dies, the project history is gone.

### Distributed VCS (DVCS)

Every developer clones the full repository — every file, every commit, the entire history. Git and Mercurial are distributed VCS. There is no single server that must be online; any clone can act as a backup or a new canonical source.

```text
Remote (GitLab)
   ↑  ↓       ↑  ↓
Dev A clone   Dev B clone
(full history) (full history)
```

This design makes Git fast (most operations are local), resilient (any clone can restore the remote), and flexible (multiple workflows: centralised, forking, peer-to-peer).

## The Git Model: Snapshots, Not Deltas

Most older VCS tools think of history as a list of per-file diffs (deltas). They store the original file plus a chain of changes to reconstruct any version.

Git thinks differently. **On every commit, Git stores a snapshot of the entire project** — a pointer to the full state of every tracked file at that moment. Files that have not changed are not stored again; Git stores a reference to the identical blob it already has. But conceptually, each commit is a complete picture of the project, not a diff.

This makes operations like branching, merging, and checking out old versions extremely fast — Git does not need to replay a chain of diffs.

```text
Delta-based (SVN):
File A: v1 → Δ1 → Δ2 → Δ3

Snapshot-based (Git):
Commit 1: [File A v1] [File B v1]
Commit 2: [File A v2] [File B v1 — same ref]
Commit 3: [File A v2 — same ref] [File B v2]
```

## Key Concepts

### Repository

A **repository** (repo) is the `.git/` directory at the root of your project. It contains the entire history: every commit, every version of every file, all branch and tag references. When you run `git clone`, you receive a full copy of this directory.

### Working Tree

The **working tree** (also called the working directory) is the set of files you actually see and edit in your project folder. These are the checked-out files that correspond to a particular commit. Changes you make here are not yet tracked by Git.

### Staging Area (Index)

The **staging area** (also called the index) is a preparation zone that sits between your working tree and the repository. When you run `git add`, you copy changes from the working tree into the staging area. Only what is in the staging area will be included in the next commit. This lets you craft precise, focused commits even when you have changed many files.

### Commit

A **commit** is a permanent snapshot stored in the repository. It records the state of the staging area at the moment you ran `git commit`, plus metadata: author, timestamp, and a commit message explaining the change. Each commit is identified by a unique SHA-1 hash (e.g., `a3f9c12`).

### History (DAG)

Git history is a **directed acyclic graph (DAG)** of commits. Each commit points to its parent commit (or two parents in the case of a merge). This graph is the authoritative record of everything that happened to the project.

## The Three Areas: A Visual Overview

```text
┌─────────────────┐    git add     ┌─────────────────┐    git commit    ┌─────────────────┐
│                 │ ─────────────► │                 │ ───────────────► │                 │
│  Working Tree   │                │  Staging Area   │                  │   Repository    │
│  (your files)   │ ◄───────────── │    (index)      │                  │   (.git/)       │
│                 │  git restore   │                 │                  │                 │
└─────────────────┘                └─────────────────┘                  └─────────────────┘
                                                          git restore --staged
                                   ◄──────────────────────────────────────────────────────
                                                          git checkout / git switch
```

- **Working Tree → Staging Area**: `git add <file>` — you choose what to include in the next commit.
- **Staging Area → Repository**: `git commit` — you record the snapshot permanently.
- **Repository → Working Tree**: `git checkout`, `git switch`, or `git restore` — you update your files to match a past commit or branch.

## Common Pitfalls

- Confusing the working tree with the repository — editing files does not save anything in Git until you `add` and `commit`.
- Thinking Git stores diffs — Git stores snapshots; diffs are computed on the fly when you run `git diff` or `git show`.
- Skipping the staging area by always using `git add .` — the staging area exists so you can build clean, focused commits; use it intentionally.
- Treating a centralised server as if it were Git's model — in Git, the remote is just another clone; the real history lives in every `.git/` directory.

## Summary

- Version control tracks file changes over time, enabling collaboration, history, backup, and rollback.
- Local VCS (RCS) works for one person; centralised VCS (SVN) needs a server; distributed VCS (Git) gives every developer a full copy.
- Git stores **snapshots** of the whole project on each commit, not file-by-file deltas — this makes it fast and flexible.
- The three areas are: **working tree** (your editable files), **staging area** (your next commit under construction), and **repository** (permanent history in `.git/`).
- Git history is a directed acyclic graph (DAG) of commits, each identified by a SHA-1 hash.
