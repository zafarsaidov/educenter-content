# Git Hooks

Git hooks are shell scripts that Git executes automatically at specific points in the version-control workflow, letting you enforce quality standards locally before any code reaches a shared remote. They live inside `.git/hooks/` and can abort a Git operation entirely if they exit with a non-zero status code.

## What Are Hooks?

Every Git repository comes with a set of sample hook scripts in `.git/hooks/`. You can see them immediately after `git init`:

```bash
ls .git/hooks/
# applypatch-msg.sample  commit-msg.sample  pre-commit.sample
# pre-push.sample        prepare-commit-msg.sample  ...
```

Each file is a disabled template. To activate a hook, remove the `.sample` extension and make the file executable:

```bash
# Activate the pre-commit hook
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

Git will now run this script automatically before every `git commit`.

### Exit Codes Control the Outcome

| Exit code | Meaning |
|-----------|---------|
| `0`       | Success — Git continues with the operation |
| Non-zero  | Failure — Git aborts the operation and prints the hook's output |

`post-*` hooks are purely informational; their exit code is ignored.

## Why Use Hooks?

Without hooks, quality checks only happen in CI/CD pipelines — after the code has already been pushed and a pipeline triggered. That feedback loop is slow (minutes) and public (everyone can see the failure). Hooks give you fast, private feedback in seconds, right on your machine.

Common use cases:

- Lint code before committing to catch syntax errors early
- Run a fast unit test suite before pushing to a shared branch
- Validate that commit messages follow a convention (Conventional Commits, JIRA ticket prefix, etc.)
- Scan for accidentally committed secrets or large binary files

## Client-Side Hooks

Client-side hooks run on the developer's local machine and are triggered by local Git operations.

### `pre-commit`

Runs immediately before Git creates the commit object. If it exits non-zero, the commit is aborted and nothing is saved to history.

**Typical uses:** linting, static analysis, secret scanning, running fast unit tests.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit
# Run flake8 on every staged Python file

set -e

STAGED_PY=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$' || true)

if [ -z "$STAGED_PY" ]; then
  exit 0
fi

echo "Running flake8 on staged files..."
flake8 $STAGED_PY

# flake8 exits non-zero if there are any style violations,
# which will abort the commit.
```

You can bypass a hook in an emergency with `git commit --no-verify`, but this should be rare and intentional.

### `commit-msg`

Runs after you finish writing the commit message but before the commit is finalised. Git passes the path to the temporary file containing your message as the first argument (`$1`).

**Typical use:** enforce a commit message format.

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg
# Enforce the Conventional Commits format:
#   <type>(optional scope): <description>

MSG_FILE="$1"
COMMIT_MSG=$(cat "$MSG_FILE")

# Regex: type(scope): description   OR   type: description
PATTERN='^(feat|fix|chore|docs|refactor|test|ci|perf|style|build|revert)(\(.+\))?(!)?: .{1,72}$'

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo ""
  echo "ERROR: Commit message does not follow Conventional Commits format."
  echo ""
  echo "Expected:  <type>(scope): <short description>"
  echo "Example:   feat(auth): add OAuth2 login support"
  echo "Example:   fix: correct off-by-one error in pagination"
  echo ""
  echo "Your message: $COMMIT_MSG"
  exit 1
fi

exit 0
```

### `post-commit`

Runs after a commit has been successfully created. Its exit code does not affect anything — the commit already exists. Use it for notifications or logging.

```bash
#!/usr/bin/env bash
# .git/hooks/post-commit
# Print a reminder after every commit

echo "Commit created. Don't forget to push when your feature is complete."
```

### `pre-push`

Runs before `git push` sends data to the remote. It receives the remote name and URL on stdin. Exit non-zero to abort the push.

**Typical use:** run the full test suite locally before code reaches a shared branch.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push
# Run the test suite before pushing

echo "Running tests before push..."
npm test

if [ $? -ne 0 ]; then
  echo "Tests failed. Push aborted."
  exit 1
fi
```

## Server-Side Hooks (Overview)

Server-side hooks run on the Git server (e.g., GitLab, GitHub Enterprise, Gitea) and are managed by server administrators, not individual developers.

| Hook | When it runs | Common use |
|------|-------------|------------|
| `pre-receive` | Before any refs are updated on the server | Reject pushes that violate policy (e.g., no force-pushes to `main`, commit message must pass lint) |
| `update` | Once per branch being pushed | Same as `pre-receive` but scoped to a single branch |
| `post-receive` | After all refs are updated | Trigger CI/CD pipelines, send notifications, update issue trackers |

Because `pre-receive` can reject a push, it is a powerful enforcement point for organisations that need to guarantee certain policies regardless of what hooks developers have locally.

## The Problem with `.git/hooks/`

There is a fundamental limitation: `.git/hooks/` is **not tracked by Git**. When a new developer clones the repository, they get an empty `hooks/` directory with only the `.sample` files. There is no built-in way to distribute hooks as part of the repository.

Common workarounds — such as storing scripts in a `scripts/hooks/` directory and asking developers to symlink them — are error-prone and easy to forget.

## Solution: The `pre-commit` Framework

The [`pre-commit`](https://pre-commit.com) tool solves the distribution problem. You define hooks in a `.pre-commit-config.yaml` file that lives in the repository root and is committed to Git. Every developer installs the hooks with a single command.

### Installation

```bash
# Install the pre-commit tool (requires Python)
pip install pre-commit

# Install the hooks defined in .pre-commit-config.yaml into .git/hooks/
pre-commit install
```

From this point, `pre-commit` manages a `.git/hooks/pre-commit` script automatically.

### Example `.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace        # Remove trailing whitespace from all files
      - id: end-of-file-fixer          # Ensure files end with a single newline
      - id: check-yaml                 # Validate YAML syntax
      - id: check-added-large-files    # Abort if a file larger than 500 KB is staged
        args: ['--maxkb=500']
      - id: detect-private-key         # Abort if a private key is accidentally staged

  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks:
      - id: black                      # Auto-format Python code with Black
        language_version: python3.11

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.27.0
    hooks:
      - id: commitizen                 # Enforce Conventional Commits on commit-msg
        stages: [commit-msg]
```

### Running Hooks Manually

```bash
# Run all hooks against all files (useful when first adopting pre-commit)
pre-commit run --all-files

# Run a specific hook
pre-commit run trailing-whitespace --all-files

# Update all hook versions to the latest
pre-commit autoupdate
```

### How It Works

When a developer runs `git commit`, the managed `pre-commit` script downloads the hook repositories listed in `.pre-commit-config.yaml` (cached in `~/.cache/pre-commit/`), creates isolated virtual environments for each, and runs the hooks in order. If any hook fails, the commit is aborted and the developer sees exactly which check failed and why.

## Common Pitfalls

- **Forgetting `chmod +x`**: a hook script that is not executable is silently ignored by Git — no error, no output, the hook just never runs.
- **Using `--no-verify` habitually**: bypassing hooks defeats their purpose; treat `--no-verify` as a last resort and add a comment explaining why.
- **Writing hooks that are too slow**: a `pre-commit` hook that takes 30 seconds will annoy the whole team and encourage `--no-verify`; keep hooks fast by only linting staged files, not the entire project.
- **Not sharing hooks**: storing hooks only in `.git/hooks/` means new team members never get them; always pair local hooks with a sharing mechanism like the `pre-commit` framework.
- **Hardcoding tool paths**: using `/usr/local/bin/flake8` will break on machines with different Python installations; prefer shebang lines like `#!/usr/bin/env bash` and rely on `PATH`.
- **Server-side vs client-side confusion**: client-side hooks can be bypassed by the developer; if a policy is non-negotiable (e.g., no direct pushes to `main`), enforce it with a server-side hook or protected branch settings, not only a local hook.

## Summary

- Git hooks are executable scripts in `.git/hooks/` that run automatically at defined points in the Git workflow.
- Exit code `0` lets the operation continue; any non-zero exit code aborts it (for non-`post-*` hooks).
- Key client-side hooks: `pre-commit` (lint/test before commit), `commit-msg` (validate message format), `post-commit` (informational), `pre-push` (run tests before push).
- Server-side hooks (`pre-receive`, `post-receive`) run on the Git server and can enforce organisation-wide policies regardless of client configuration.
- `.git/hooks/` is not tracked by Git, so hooks are not shared automatically with the team.
- The `pre-commit` framework solves the sharing problem: hooks are defined in `.pre-commit-config.yaml`, committed to the repo, and installed with `pre-commit install`.
- Keep hooks fast, target only changed files where possible, and avoid making `--no-verify` a habit.
