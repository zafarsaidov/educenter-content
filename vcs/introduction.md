# Course Introduction

This is a complete Git course designed for developers who want to use Git confidently in real projects. It covers everything from first principles to advanced operations, collaborative workflows, and professional tooling. Whether you are starting from scratch or filling gaps in your existing knowledge, this course gives you a solid, practical understanding of modern version control.

## Why Git Matters

Every software project needs version control. Without it, teams resort to emailing zip files, naming documents `report_final_v3_REAL.docx`, and losing hours trying to figure out what changed when a bug appeared.

Git solves four concrete problems:

- **Collaboration** — multiple people work on the same codebase simultaneously without overwriting each other's changes.
- **History** — every change is recorded with who made it, when, and why.
- **Recovery** — deleted a file or introduced a bug? Roll back to any previous state in one command.
- **Accountability** — the full audit trail tells you exactly when a behaviour changed and who changed it.

Git is the dominant version control system in the industry. GitHub, GitLab, Bitbucket — all major platforms are built on Git. Learning Git is not optional for a professional developer.

## What You Will Learn

The course is organised into four sections, each building on the previous one.

### Section 1 — Git Basics

The foundation. You will understand what version control is and how Git models your project as a sequence of snapshots. You will install and configure Git, create repositories, track changes with `git status` and `git diff`, write quality commits, read history with `git log`, and keep your repository clean with `.gitignore`.

**Lessons:**
1. What is Version Control — VCS types, Git's snapshot model, the three areas (working tree, staging area, repository)
2. Installation and Configuration — installing on any OS, config levels (system, global, local), SSH key setup
3. Init, Clone, and Status — creating and cloning repositories, tracking file states, understanding `git diff`
4. Staging and Commits — `git add`, patch mode, commit messages, commit object anatomy
5. History and .gitignore — `git log`, `git show`, `.gitignore` patterns, `git rm --cached`

### Section 2 — Branching, Merging, and History

Branches are Git's most powerful feature. You will learn what a branch really is, how to create, switch, and delete branches, how to merge with fast-forward and three-way merges, how to resolve conflicts, and how rebasing works — including interactive rebase for cleaning up history before sharing. The section ends with practical tools for undoing work: `git restore`, `git revert`, `git reset`, and `git reflog`.

**Lessons:**
1. Branches — what a branch is, `git switch`, detached HEAD, naming conventions
2. Merging and Conflicts — fast-forward, three-way merge, conflict markers, `git merge --abort`
3. Rebasing — `git rebase`, interactive rebase, squash, the golden rule
4. Undoing Changes — `git restore`, `git revert`, `git reset` (soft, mixed, hard), `git reflog`

### Section 3 — Remotes, Tags, and Advanced Operations

Everything that happens beyond your local machine. You will add and manage remotes, understand the difference between `git fetch` and `git pull`, push branches with upstream tracking, and work with the fork workflow. You will create lightweight and annotated tags, push them to remotes, and understand semantic versioning. The section closes with `git stash` for shelving work and `git cherry-pick` for surgical commit transfers.

**Lessons:**
1. Working with Remotes — fetch vs pull, `git push -u`, fork workflow, shallow clones
2. Tags and Releases — annotated vs lightweight, `--follow-tags`, semantic versioning, hotfix from a tag
3. Stashing and Cherry-picking — `git stash`, `git cherry-pick`, `--no-commit`, cherry-pick vs merge

### Section 4 — Hooks and Professional Workflow

The gap between knowing Git and using it professionally. You will write client-side hooks that lint code and validate commit messages before they reach history, learn the Conventional Commits specification, and understand three branching strategies — Feature Branch Workflow, Gitflow, and Trunk-Based Development — so you can choose the right one for your team.

**Lessons:**
1. Git Hooks — pre-commit, commit-msg, pre-push, the `pre-commit` framework
2. Conventional Commits — `type(scope): description`, semantic release, commitizen
3. Branching Strategies — Feature Branch, Gitflow, Trunk-Based Development, merge strategies

## Who This Course Is For

- **Beginners** who have never used Git — the course starts from zero and builds up.
- **Developers** who know the basics but struggle with branching, rebasing, or conflict resolution.
- **DevOps and backend engineers** who want a solid, systematic understanding rather than scattered knowledge.

**Prerequisite:** basic terminal usage (navigating directories, running commands). No prior Git experience required.

## Course Format

Each lesson has two parts:

- **Theory** — visual presentation explaining concepts with diagrams and examples.
- **Practice** — live terminal demonstration where you follow along on your own machine.

## Summary

- Git is the industry-standard version control system — every professional developer needs it.
- This course covers Git from first principles to professional workflows in four sections and fifteen lessons.
- Each lesson combines theory (visual presentation) with practice (live terminal demo).
- No prior Git experience required — only basic terminal skills.
- By the end of the course you will be confident with branching, merging, rebasing, remotes, tags, hooks, and team workflows.
