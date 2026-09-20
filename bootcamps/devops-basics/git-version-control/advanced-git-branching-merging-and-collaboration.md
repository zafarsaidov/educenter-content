# Lesson 15: Advanced Git – Branching, Merging, and Collaboration

**Module:** Git Version Control
**Duration:** 120-150 min
**Prerequisites:** Lesson 14 (Git Fundamentals and Workflow)

## Learning Objectives

By end of lesson student can:
- Create, switch between, and delete branches
- Explain fast-forward vs 3-way merges, and resolve a merge conflict
- Rebase a branch, including using interactive rebase to clean up commit history
- Explain when to choose rebase vs merge
- Use `git stash` to shelve in-progress work, and `git cherry-pick` to apply a specific commit elsewhere
- Explain the fork/PR (pull request) collaboration workflow, and create annotated tags for releases

## Topics

- Branches: `git branch` (`-a`, `-d`, `-D`), `git checkout`/`switch`, `git log --all --graph`
- Merging: fast-forward vs 3-way merge; `git merge`; conflict resolution (`git mergetool`)
- Rebasing: `git rebase`, interactive rebase (`-i`: squash, reword, fixup); rebase vs merge
- Stash and cherry-pick: `git stash` (`push`, `pop`, `list`, `drop`); `git cherry-pick`
- Collaboration workflow: fork, PR/MR concept; `git tag` (`-a` annotated, releases)

## Concepts

### Branches

A **branch** is just a movable pointer to a specific commit — nothing more. When you commit on a branch, the branch pointer moves forward to the new commit; the previous commit remains part of history, reachable by walking backward. This lightweight design is what makes branching in Git fast and cheap compared to some older VCSs, where a branch could mean physically copying the entire codebase.

`git branch` lists local branches (`-a` includes remote-tracking branches too), creates one (`git branch new-feature`), or deletes one (`-d` for a safe delete that refuses if the branch has unmerged commits, `-D` to force-delete regardless). `git checkout <branch>` or the newer `git switch <branch>` moves you onto a different branch, updating your working tree to match that branch's latest commit. `git log --all --graph` visualizes every branch's history and where they diverged/converged — much more useful once more than one branch exists.

### Merging: fast-forward vs 3-way

`git merge <branch>` brings another branch's commits into your current branch. Git picks one of two strategies automatically, depending on history shape:

- **Fast-forward merge** — happens when the current branch hasn't diverged at all (no new commits of its own since the other branch was created). Git just moves the current branch's pointer forward to match — no new "merge commit" is created, history stays perfectly linear.
- **3-way merge** — happens when both branches have new commits since they diverged. Git looks at three points (the common ancestor, and the tip of each branch), combines the changes from both, and creates a new **merge commit** with two parents, recording that the two histories came back together.

A **merge conflict** happens during a 3-way merge when both branches changed the *same* lines of the *same* file in different ways — Git can't automatically decide which version is correct, so it pauses and marks the conflicting sections in the file for you to resolve manually (or with a visual `git mergetool`), before completing the merge with another commit.

### Rebasing

`git rebase <branch>` takes the commits unique to your current branch and replays them, one by one, on top of the tip of `<branch>` instead — rewriting your branch's history as if you'd started your work from that later point. Unlike merge, this produces a **linear** history with no merge commit, at the cost of literally rewriting commit hashes (each replayed commit is a genuinely new commit object, even if its content is identical).

**Interactive rebase** (`git rebase -i <base>`) opens an editable list of commits, letting you rewrite recent history before it's shared:

| Action | Effect |
|---|---|
| `pick` | Keep the commit as-is (default) |
| `reword` | Keep the commit's changes, but edit its message |
| `squash` | Combine this commit into the previous one, merging their messages |
| `fixup` | Combine this commit into the previous one, discarding this commit's message entirely |
| `drop` | Remove the commit entirely |

This is commonly used before opening a pull request, to turn a messy sequence of "wip", "fix typo", "actually fix it" commits into one or two clean, meaningful commits.

### Rebase vs merge: when to use which

The golden rule: **never rebase commits that have already been pushed and might be based on by someone else** — since rebase rewrites history (new commit hashes), anyone who already pulled the old commits now has a diverged, conflicting view of history. Rebase is safe and useful for cleaning up your *own local, not-yet-shared* work before pushing it. Merge is the safe default for combining branches that are already shared/pushed, since it never rewrites existing commits — it only adds a new merge commit on top.

### Stash: shelving in-progress work

`git stash` temporarily saves your uncommitted working tree changes (both staged and unstaged) and reverts your working tree to match the last commit — useful when you need to switch branches or pull urgent changes but aren't ready to commit what you're working on. `git stash pop` re-applies the most recent stash and removes it from the stash list; `git stash apply` re-applies without removing it (useful if you want to apply the same stash to more than one branch). `git stash list` shows everything currently stashed; `git stash drop` discards one without applying it.

### Cherry-pick: applying one specific commit elsewhere

`git cherry-pick <commit>` takes the changes introduced by one specific commit (from anywhere in history, any branch) and applies them as a new commit on your current branch — without bringing along anything else from that commit's original branch. Common use case: a critical bug fix was committed on a feature branch, and you need that exact fix on `main` immediately, without merging the entire (still in-progress) feature branch.

### Collaboration: fork, pull requests, and tags

On platforms like GitHub/GitLab, a **fork** is your own personal copy of someone else's repository, hosted under your account — you can push commits to your fork freely without needing write access to the original. A **pull request** (GitHub's term) or **merge request** (GitLab's term) — commonly abbreviated PR/MR — is a formal proposal to merge changes from one branch (often on your fork) into another (typically the original repository's `main`), giving maintainers a place to review the diff, leave comments, and discuss before merging.

`git tag` marks a specific commit with a permanent, human-readable name — most commonly used to mark release points (`v1.0.0`, `v2.1.3`). An **annotated tag** (`git tag -a`) stores extra metadata (tagger name, date, a message) and is the recommended kind for anything meant to represent a real release, versus a lightweight tag (just a name pointing at a commit, no metadata) used more for quick, throwaway markers.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `git branch` | List/create/delete branches | `git branch -a` |
| `git branch -d`/`-D` | Delete a branch (safe / forced) | `git branch -d old-feature` |
| `git switch` | Switch to a branch | `git switch feature-x` |
| `git checkout -b` | Create + switch in one step | `git checkout -b feature-x` |
| `git merge` | Merge another branch into current | `git merge feature-x` |
| `git mergetool` | Open a visual conflict resolver | `git mergetool` |
| `git rebase` | Replay commits onto another base | `git rebase main` |
| `git rebase -i` | Interactive rebase | `git rebase -i HEAD~3` |
| `git stash` | Shelve uncommitted changes | `git stash push -m "wip"` |
| `git stash pop`/`list`/`drop` | Manage the stash | `git stash pop` |
| `git cherry-pick` | Apply one commit elsewhere | `git cherry-pick abc1234` |
| `git tag -a` | Create an annotated tag | `git tag -a v1.0.0 -m "First release"` |
| `git push --tags` | Push tags to remote | `git push origin --tags` |

## Examples / Walkthrough

```bash
# --- creating and switching branches ---
git branch -a                                    # list all branches (local + remote-tracking)
git checkout -b feature/login                    # create AND switch, in one step
git switch feature/login                          # equivalent, newer syntax, if the branch already exists

echo "console.log('login page');" > login.js
git add login.js
git commit -m "Add login page skeleton"

git switch main                                    # back to main, feature/login's commit stays on its branch
git log --all --graph --oneline                    # see both branches and where they diverged

# --- fast-forward merge (main hasn't moved since branching) ---
git merge feature/login
# no new merge commit created — main's pointer just moves forward to feature/login's tip
git log --oneline --graph                           # confirm: still a straight line

# --- 3-way merge with a conflict ---
git checkout -b feature/header main
echo "<h1>Welcome</h1>" > header.html
git add header.html && git commit -m "Add header (feature branch)"

git switch main
echo "<h1>Hello</h1>" > header.html                  # same file, different content, on main
git add header.html && git commit -m "Add header (main)"

git merge feature/header
# CONFLICT (add/add): Merge conflict in header.html
# Git marks the conflicting section directly in the file:
#   <<<<<<< HEAD
#   <h1>Hello</h1>
#   =======
#   <h1>Welcome</h1>
#   >>>>>>> feature/header

# resolve manually with vim/nano (Lesson 3), keeping the version you want, then:
git add header.html                                   # marks the conflict as resolved
git commit -m "Merge feature/header, resolve header.html conflict"
# (git generates a default merge commit message; -m overrides it if you prefer your own)

# --- rebase: clean, linear history for LOCAL, unpushed work ---
git checkout -b feature/search main
echo "function search() {}" > search.js
git add search.js && git commit -m "wip"
echo "function search(query) { return query; }" > search.js
git add search.js && git commit -m "actually implement search"
echo "// added a comment" >> search.js
git add search.js && git commit -m "fix typo"

git log --oneline                                      # 3 messy commits

git rebase -i main
# in the editor, change to:
#   pick abc1111 wip
#   fixup abc2222 actually implement search   <- folds into "wip", message discarded
#   fixup abc3333 fix typo                     <- folds into "wip" too
# save and close -> results in ONE clean commit
git log --oneline                                       # now a single, tidy commit

# then rebase this cleaned branch onto the latest main, if main has moved since
git rebase main
git switch main
git merge feature/search                                 # fast-forwards cleanly since history is now linear

# --- stash: shelve unfinished work to switch context ---
echo "unfinished experiment" >> app.js
git status                                                # app.js shown as modified, not committed

git stash push -m "experimenting with app.js"
git status                                                 # working tree is clean again
git stash list                                              # shows the saved stash with its message

# do something else (fix an urgent bug, switch branches, pull, etc.), then come back:
git stash pop                                                # re-applies AND removes it from the stash list

# --- cherry-pick: grab one specific commit onto another branch ---
git log feature/search --oneline                              # find the commit hash of a fix you need on main
git switch main
git cherry-pick <commit-hash-from-feature-branch>
# applies just that one commit's changes onto main, as a new commit

# --- tags: marking a release ---
git tag -a v1.0.0 -m "First stable release"
git tag                                                        # list all tags
git show v1.0.0                                                 # see the tag's metadata + the commit it points to
git push origin --tags                                          # tags aren't pushed by a plain "git push"
```

## Common Pitfalls

- **Rebasing commits that have already been pushed and shared** — this rewrites commit hashes; anyone who already has the old commits now has a history that conflicts with yours, requiring a forced push (`git push --force`) that can silently discard others' work if they'd pushed on top of the old commits. Only rebase local, unpushed, or explicitly-yours-alone branches.
- **Forgetting `git add` after resolving a merge conflict** — editing the file to resolve the conflict markers isn't enough; Git still considers the file conflicted until you `git add` it, which is what tells Git "I've resolved this."
- **Confusing `git branch -d` refusing to delete as a bug** — `-d` deliberately refuses to delete a branch with commits that aren't reachable from any other branch (i.e. would be lost). This is a safety feature, not an error — either merge the branch first, or use `-D` if you're certain you want to discard that history.
- **Deleting a stash entry with `git stash drop` by accident, or losing track of stash order** — stashes are a stack; `pop` always applies the most recent one. If you have several stashed at once, `git stash list` before popping/dropping to avoid applying or discarding the wrong one.
- **Squashing/fixup-ing commits in the wrong order during interactive rebase** — `fixup`/`squash` fold *into the commit above them* in the list, not the original commit below. Getting the list order wrong produces a different history than intended; always re-check `git log --oneline` after an interactive rebase.
- **Forgetting `git push --tags`** — tags are not included in a normal `git push` by default; a tag created locally stays local until explicitly pushed, which can surprise you when a release tag "isn't showing up" on the remote.

## FAQ

**Q: What's actually different about a merge commit vs a normal commit?**
A: A normal commit has exactly one parent (the commit before it). A merge commit has two (or more) parents — one for each branch being merged — which is how Git represents "these two lines of history came back together here."

**Q: If rebase rewrites history, is it "dangerous"?**
A: Only for commits others might already be building on. For your own local, not-yet-pushed work, rebasing is not dangerous at all — it's a completely normal way to clean up commits before sharing them. The danger is specifically about rewriting history that already exists elsewhere.

**Q: When would I choose `cherry-pick` instead of just merging the whole branch?**
A: When you need exactly one commit's changes without the rest of that branch's (possibly unfinished, possibly unrelated) history — a common real case is backporting a single critical fix to a release branch or to `main` without pulling in an entire in-progress feature.

**Q: What's the difference between a fork and a branch?**
A: A branch is a pointer within one repository. A fork is an entirely separate copy of the whole repository, usually under a different account, used specifically for the case where you don't have write access to push branches directly to the original — you push to your own fork, then open a pull request asking the original repository's maintainers to pull your changes in.

**Q: Why bother with annotated tags instead of just remembering commit hashes for releases?**
A: A tag gives a permanent, memorable, human-readable name (`v2.1.0`) to a specific point in history, and an annotated tag additionally records who created it, when, and why — genuinely useful metadata for anyone auditing release history later, which a bare commit hash doesn't carry on its own.

## Practice / Exercise

**Core:**
1. Create a new branch, make 2-3 commits on it, then merge it back into `main` — confirm from `git log --graph` whether it was a fast-forward or a 3-way merge, and explain why.
2. Deliberately create a merge conflict: edit the same line of the same file differently on two branches, attempt to merge, resolve the conflict manually, `git add` the resolved file, and complete the merge.
3. Create a branch with 3 intentionally messy commits ("wip", "fix", "actually fix"), then use `git rebase -i` to squash them into one clean commit with a good message.
4. Practice `git stash`: make an uncommitted change, stash it, confirm your working tree is clean, switch branches, switch back, and `git stash pop` to restore it.
5. Make a commit on one branch, then `git cherry-pick` it onto a different branch, and confirm with `git log` that it now appears (as a new commit) on both.
6. Create an annotated tag for a commit in your practice repo, and push it to your remote with `git push origin --tags`.

**Stretch:**
1. Simulate a small collaboration: fork a public repository (or have a partner fork yours), make a change on a branch in the fork, and open a pull/merge request back to the original.
2. Deliberately push a branch, then try to rebase it and force-push — in a solo practice repo only — and explain, in your own words, why doing this on a shared branch with active collaborators would cause problems for them.
3. Use `git stash list`/`git stash apply` (not `pop`) to apply the same stashed change to two different branches without removing it from the stash list in between.

## Further Reading

- [Pro Git book: Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [Pro Git book: Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
- `git help rebase`, `git help merge`, `git help tag`
