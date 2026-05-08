# Branching Strategies and GitLab Workflow

A branching strategy is an agreement about how your team uses branches, when it creates them, what they are named, and how they eventually land back into the codebase. Choosing the right strategy aligns the whole team, keeps the repository clean, and makes releases predictable.

## Why Branching Strategies Matter

Without a shared strategy, teams accumulate problems quickly:

- Long-lived branches diverge so far from `main` that merging them becomes a painful multi-day conflict resolution exercise.
- Nobody knows which branch to base new work on, or which one is "safe to deploy".
- Hotfixes get applied to the wrong branch and never reach production.
- Code review becomes inconsistent because there is no agreed process for getting changes merged.

A strategy solves all of this by giving every branch a clear purpose, a defined lifetime, and an agreed merge path.

## Feature Branch Workflow

The simplest strategy that works for most teams. Every change — no matter how small — happens on a dedicated branch. Nothing goes directly to `main`.

### How It Works

1. Create a branch from `main` named after the feature or fix.
2. Do the work on that branch, committing as you go.
3. Open a Merge Request (or Pull Request) targeting `main`.
4. A teammate reviews the code; the author addresses feedback.
5. After approval, the branch is merged into `main` and deleted.

```text
         feat/login-page        feat/payment-api
               |                      |
main ──────────┼──────────────────────┼──────────────► main
         (branch)  (work)  (MR+merge) │ (branch)(work)(MR+merge)
                                      │
                              fix/null-pointer
                                      │
               main ─────────────────┼──────────────► main
                               (branch)(fix)(MR+merge)
```

Diagram as a timeline:

```text
main:        A────────────────B────────────────C──────►
                \            /  \             /
feat/login:      D──E──F────     G──H────────
```

### When to Use It

- Teams of any size.
- Projects with no strict release schedule — you deploy `main` whenever it is ready.
- Good starting point before adopting a more complex strategy.

### Commands

```bash
# Start new feature
git switch main
git pull origin main
git switch -c feat/user-profile

# Work, commit, push
git add .
git commit -m "feat(profile): add avatar upload"
git push -u origin feat/user-profile

# After MR is merged, clean up locally
git switch main
git pull origin main
git branch -d feat/user-profile
```

## Gitflow

Gitflow is a more structured strategy designed for teams that ship software in versioned releases on a defined schedule. It was introduced by Vincent Driessen in 2010 and remains widely used for libraries, mobile apps, and any software that has distinct release points.

### Branch Types

| Branch | Lifetime | Purpose |
|--------|----------|---------|
| `main` | Permanent | Production-ready code; every commit is a tagged release |
| `develop` | Permanent | Integration branch; new features are merged here first |
| `feature/*` | Short-lived | Individual features; branch from `develop`, merge back to `develop` |
| `release/*` | Short-lived | Release stabilisation; branch from `develop`, merge to both `main` and `develop` |
| `hotfix/*` | Short-lived | Urgent production fixes; branch from `main`, merge to both `main` and `develop` |

### Flow Diagram

```text
main:     ──────────────────────────────────────1.0──────────────────1.1──►
               \                               / \                  / \
hotfix:         \                             /   hotfix/1.0.1─────    \
                 \                           /                           \
develop:  ─────────────────────────────────────────────────────────────────►
               \ \                         /                         /
feature:        \ feat/login──────────────                          /
                 feat/payment────────────────────────release/1.1───
```

Full text diagram of a typical Gitflow cycle:

```text
develop ──┬──────────────────────────────────────────────┬──────►
          │                                              │
          ├─── feature/search ──────────────────────────┤
          │                                              │
          ├─── feature/checkout ────────────────────────┤
          │                                              │
          └─── release/1.2 ──── (bugfixes) ─────────────┴──► main (tag: v1.2.0)
                                                              │
                                                     hotfix/1.2.1
                                                              │
                                                         main (tag: v1.2.1)
                                                              │
                                                    also merged back ──► develop
```

### When to Use It

- Libraries or packages with public APIs where versioning is critical.
- Mobile apps that go through an app store review process.
- Enterprise software with scheduled quarterly releases.
- Teams that need a stabilisation period (the `release` branch) between feature completion and deployment.

### Common Gitflow Commands

```bash
# Start a feature
git switch develop
git pull origin develop
git switch -c feature/search-autocomplete

# Finish a feature (merge back to develop)
git switch develop
git merge --no-ff feature/search-autocomplete
git branch -d feature/search-autocomplete
git push origin develop

# Start a release branch
git switch develop
git switch -c release/1.2
# fix last-minute bugs here...
git commit -m "fix(search): handle empty query string"

# Finish a release: merge to main AND develop
git switch main
git merge --no-ff release/1.2
git tag -a v1.2.0 -m "Release version 1.2.0"
git switch develop
git merge --no-ff release/1.2
git branch -d release/1.2

# Hotfix
git switch main
git switch -c hotfix/1.2.1
git commit -m "fix(auth): patch session token expiry bug"
git switch main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git switch develop
git merge --no-ff hotfix/1.2.1
git branch -d hotfix/1.2.1
```

## Trunk-Based Development

In trunk-based development (TBD), all developers commit to a single shared branch — `main` (the "trunk") — either directly or via very short-lived branches that are merged within a day. There are no long-lived feature branches.

### How It Works

1. Pull the latest `main`.
2. Make a small, self-contained change.
3. Commit directly to `main` (or open a branch that lives less than 24 hours).
4. The CI pipeline runs immediately.
5. `main` is always deployable; teams release by tagging any passing commit.

Incomplete features are hidden behind **feature flags** — runtime configuration that enables or disables code paths without deploying a separate branch.

```text
main: ──A──B──C──D──E──F──G──H──I──J──►
        │     │        │        │
        │     │        │        └── short-lived: merged same day
        │     │        └── direct commit
        │     └── direct commit
        └── direct commit
```

### Feature Flags

```python
# Feature flag controlled by environment variable
ENABLE_NEW_CHECKOUT = os.getenv("FEATURE_NEW_CHECKOUT", "false") == "true"

if ENABLE_NEW_CHECKOUT:
    return new_checkout_flow(cart)
else:
    return legacy_checkout_flow(cart)
```

The new code is deployed but inactive for users until the flag is turned on in production.

### When to Use It

- Teams practising continuous deployment (deploying multiple times per day).
- Mature CI/CD pipelines that can validate changes in minutes.
- Experienced teams that can keep individual commits small and focused.
- SaaS products where there is only one "version" in production.

## Comparison Table

| Workflow | Branch lifetime | Release style | Best team size | CI requirement |
|----------|----------------|---------------|---------------|----------------|
| Feature Branch | Days to weeks | Deploy `main` when ready | Any | Moderate |
| Gitflow | Weeks to months | Scheduled, versioned releases | Medium to large | Moderate |
| Trunk-Based | Hours to 1 day | Continuous deployment | Any (with discipline) | Strong (fast, reliable) |

No single strategy is universally best. Feature Branch Workflow is the safest starting point; adopt Gitflow if you need versioned releases; move toward trunk-based development as your CI matures and your team builds discipline around small commits.

## GitLab Merge Requests

A Merge Request (MR) in GitLab is the mechanism for proposing, reviewing, discussing, and merging a branch into another. It is the central collaboration point for the Feature Branch and Gitflow workflows.

### Creating a Merge Request

When you push a branch to GitLab, it shows a prompt to create an MR. You can also create one from the GitLab UI:

**Repository → Merge Requests → New merge request**

Fill in:

- **Source branch**: the branch with your changes (e.g., `feat/user-profile`).
- **Target branch**: the branch to merge into (usually `main` or `develop`).
- **Title**: a concise summary of what the MR does; should match or reference the commit message.
- **Description**: what changed and why; link to the relevant issue; include screenshots for UI changes.
- **Assignee**: the person responsible for the MR (usually the author).
- **Reviewers**: teammates who should review the code.
- **Labels / Milestone**: optional metadata for tracking.

```bash
# Push and create MR from the CLI using the GitLab CLI (glab)
git push -u origin feat/user-profile
glab mr create --title "feat(profile): add avatar upload" \
               --description "Closes #88" \
               --target-branch main
```

### Reviewing an MR

Reviewers navigate to the MR and use the **Changes** tab to see a unified diff of every file. GitLab provides several tools:

**Inline comments**: click the `+` icon next to any line to leave a comment or suggestion.

**Inline suggestions**: use the suggestion syntax to propose a specific code change that the author can apply with one click:

```text
```suggestion
const MAX_FILE_SIZE_MB = 5;
```
```

The author can apply the suggestion without leaving GitLab, which creates a commit automatically.

**Resolving threads**: every comment thread can be marked as "resolved" once the feedback has been addressed. The MR shows a count of unresolved threads so nothing gets missed.

**Approving**: once a reviewer is satisfied, they click **Approve**. The MR can be configured to require a minimum number of approvals before merging is allowed.

### Merge Strategies

GitLab offers three strategies when merging an MR:

| Strategy | What it does | When to use |
|----------|-------------|-------------|
| Merge commit | Creates a merge commit; preserves full branch history | Default; good audit trail |
| Squash and merge | Combines all commits into one before merging | Noisy feature branches; keeps `main` log clean |
| Rebase and merge | Replays commits on top of target; linear history | Teams that prefer linear history without merge commits |

## Protected Branches in GitLab

Protected branches prevent accidental or unauthorised changes to critical branches like `main` or `develop`.

**Settings → Repository → Protected Branches**

### What You Can Configure

| Setting | Recommended value for `main` |
|---------|------------------------------|
| Allowed to push | No one (force everyone through MRs) |
| Allowed to merge | Maintainers (or Developers + Maintainers for smaller teams) |
| Require MR approvals before merge | Enabled; minimum 1–2 approvals |
| Allowed to force push | Disabled |
| Require code owner approval | Optional; useful for large codebases |

### Setting Up via the UI

1. Go to **Settings → Repository**.
2. Scroll to **Protected Branches**.
3. Type `main` in the **Branch** field.
4. Set **Allowed to push** to "No one".
5. Set **Allowed to merge** to "Maintainers".
6. Click **Protect**.

Once protected, even a user with Maintainer role cannot push directly to `main` without going through an MR — GitLab will reject the push:

```text
remote: GitLab: You are not allowed to push code to protected branches on this project.
```

### Require Pipeline to Succeed

Enable **Pipelines must succeed** in the protected branch settings so that an MR can only be merged after all CI jobs pass. This guarantees that broken code can never reach `main` through the UI, regardless of the reviewer's judgement.

## GitLab MR Best Practices

Following these practices makes code review faster, more focused, and less painful for everyone involved.

**Keep MRs small and focused.** An MR that touches 50 files across 10 different concerns is very hard to review thoroughly. Aim for MRs that change one thing. If a feature requires database migrations, API changes, and frontend work, consider splitting them across multiple MRs.

**Write a descriptive title and description.** The title should answer "what does this change?". The description should answer "why?" and provide context the reviewer needs: background on the problem, links to the issue or specification, and any decisions made during implementation.

```text
Title (good):   feat(checkout): add Apple Pay as payment option
Title (bad):    update payment

Description:
## What
Adds Apple Pay support to the checkout flow using the Stripe Payment
Request Button API.

## Why
Requested in #312. Mobile users currently have no one-tap payment option.

## How
- Added `PaymentRequestButton` component to `CheckoutForm`
- Updated `stripeService.createPaymentIntent()` to accept `apple_pay` method
- Added feature flag `ENABLE_APPLE_PAY` (disabled by default)

Closes #312
```

**Link to the issue.** Use GitLab's [closing patterns](https://docs.gitlab.com/ee/user/project/issues/managing_issues.html#closing-issues-automatically) in the description so the issue closes automatically when the MR merges:

```text
Closes #312
Fixes #298
Resolves #401
```

**Self-review before requesting review.** Read your own diff in the GitLab UI before assigning reviewers. You will catch typos, forgotten debug statements, and obvious improvements that save your reviewer's time.

**Respond to every comment.** Even if you disagree, acknowledge the feedback. Use "Done" or "Fixed in abc1234" to confirm you addressed it. Unresponded threads signal an incomplete review.

**Do not force-push after review starts.** Force-pushing rewrites history and makes it impossible for reviewers to see what changed since their last review. If you need to incorporate feedback, add new commits; squash only at merge time.

## Common Pitfalls

- **Merging directly to `main` without a review**: even experienced developers make mistakes; a second pair of eyes consistently catches issues that automated tests miss.
- **Keeping branches alive for weeks**: long-lived branches accumulate conflicts and context drift; the longer a branch lives, the harder it is to merge.
- **Using the wrong base branch**: in Gitflow, branching a feature from `main` instead of `develop` is a common mistake that leads to complicated merges later.
- **Treating protected branches as optional**: if you configure protected branches but allow "Developers" to push directly, the protection is ineffective for most team members.
- **Giant MRs**: a 2000-line MR takes hours to review properly and often gets rubber-stamped; consistent small MRs maintain high review quality.
- **No description in the MR**: a merge request with only a title forces the reviewer to read all the code without context, slowing the review significantly.
- **Not requiring pipelines to pass before merging**: a reviewer can accidentally approve and merge an MR that breaks the build if this setting is not enabled.

## Summary

- A branching strategy gives every branch a defined purpose, lifetime, and merge path, preventing the chaos of ad-hoc branching.
- Feature Branch Workflow is the simplest strategy: every change goes on its own branch and merges to `main` via an MR; works for most teams.
- Gitflow uses two permanent branches (`main` and `develop`) and three supporting types (`feature/`, `release/`, `hotfix/`); best for teams with scheduled versioned releases.
- Trunk-Based Development commits everything to `main` (or very short-lived branches); requires strong CI and feature flags; enables continuous deployment.
- Choose the strategy that matches your release cadence; Feature Branch is a safe default, Gitflow for versioned releases, TBD for continuous deployment.
- GitLab Merge Requests are the mechanism for proposing, reviewing, and merging changes; always include a clear title, description, and link to the issue.
- Protected branches in GitLab prevent direct pushes to critical branches and can require MR approvals and passing pipelines before merging.
- Keep MRs small, self-review before requesting review, respond to all comments, and do not force-push after review starts.
