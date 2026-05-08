# Conventional Commits

A commit message is a permanent note to your future self and your teammates explaining why a change was made. Conventional Commits is a lightweight specification that adds a structured prefix to each message, making history not only human-readable but also machine-parseable for automated changelogs and semantic version bumps.

## Why Commit Messages Matter

The Git log is the narrative of your project. When a bug surfaces in production, a well-written log tells you exactly when a behaviour changed and why. A poorly written log tells you nothing.

Compare these two log excerpts from the same codebase:

```text
Bad history
───────────
a3f1c2b  stuff
7e9d012  fix
c20ab44  wip
b334f01  changes
```

```text
Good history
────────────
a3f1c2b  fix(cart): prevent duplicate items on rapid double-click
7e9d012  feat(checkout): add Apple Pay as a payment option
c20ab44  refactor(api): extract order validation into its own service
b334f01  chore(deps): upgrade stripe-node from 14.x to 15.x
```

The second log answers "what changed, where, and why" at a glance. It is also parseable by tools — a script can scan it to decide whether the next release is a patch, minor, or major version.

## The Conventional Commits Format

The specification defines a single line header, an optional body, and optional footers:

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Header Rules

- `<type>` — required; one of the defined types listed below.
- `[optional scope]` — a noun in parentheses describing the area of the codebase (e.g., `auth`, `api`, `cart`).
- `<description>` — a short imperative summary, no capital letter at the start, no period at the end, maximum ~72 characters.

### Full Example with Body and Footer

```text
feat(auth): add OAuth2 login via Google

Users can now sign in using their Google account through the OAuth2
authorization code flow. The existing username/password login remains
unchanged. Refresh tokens are stored encrypted in the database.

Closes #412
Reviewed-by: Maria Garcia
```

## Commit Types and Their Meanings

Each type signals the nature of the change. When paired with a semantic versioning tool, types also control which part of the version number is incremented.

| Type       | Meaning | Version impact |
|------------|---------|----------------|
| `feat`     | A new feature visible to the user | MINOR bump (`1.2.0` → `1.3.0`) |
| `fix`      | A bug fix visible to the user | PATCH bump (`1.2.0` → `1.2.1`) |
| `chore`    | Maintenance: dependency updates, build scripts, tooling | No bump |
| `docs`     | Documentation changes only | No bump |
| `refactor` | Code restructuring that neither adds a feature nor fixes a bug | No bump |
| `test`     | Adding or correcting tests | No bump |
| `ci`       | Changes to CI/CD configuration (pipelines, workflows) | No bump |
| `perf`     | A code change that improves performance | No bump (or PATCH, configurable) |
| `style`    | Formatting, whitespace, missing semicolons — no logic change | No bump |
| `build`    | Changes to the build system or external dependencies | No bump |
| `revert`   | Reverts a previous commit | Depends on what is reverted |

## Breaking Changes — MAJOR Version Bump

A breaking change is any change that requires consumers to modify their code. There are two ways to signal one.

### Option 1: `!` After the Type

```text
feat(api)!: remove the v1 endpoints

All v1 REST endpoints have been removed. Clients must migrate to v2.
Migration guide: https://docs.example.com/api/v2-migration
```

### Option 2: `BREAKING CHANGE:` Footer

```text
refactor(auth): replace session cookies with JWT tokens

BREAKING CHANGE: The `Set-Cookie` header is no longer returned by the
login endpoint. Clients must store the JWT from the `Authorization`
response header instead.
```

Either form causes a MAJOR version bump (`1.4.2` → `2.0.0`) when processed by a semantic versioning tool.

## Good vs Bad Commit Messages

The following examples show the same change written poorly and then correctly.

```text
BAD:  fix bug
GOOD: fix(login): redirect to dashboard after successful OAuth callback
```

```text
BAD:  update stuff
GOOD: chore(deps): upgrade axios from 1.6 to 1.7
```

```text
BAD:  WIP
GOOD: test(cart): add unit tests for coupon discount calculation
```

```text
BAD:  changed the api
GOOD: feat(api)!: replace REST endpoints with GraphQL

BREAKING CHANGE: All /api/v1/* endpoints are removed. Consumers
must switch to the GraphQL endpoint at /graphql.
```

Rules for a good description line:

- Use the imperative mood: "add feature", not "added feature" or "adds feature".
- Do not capitalise the first word (the type already signals the category).
- Do not end with a period.
- Keep it under 72 characters so it fits in `git log --oneline`.

## Scope — Optional But Valuable

The scope is a short noun identifying which part of the codebase was changed. Good scopes come from your project's own vocabulary: module names, packages, layers, or domain concepts.

```text
feat(auth): add two-factor authentication
fix(checkout): correct tax calculation for EU customers
docs(api): add rate-limiting section to README
ci(docker): cache pip dependencies in the build stage
```

Scopes are optional but become very useful when you run `git log` filtered by scope:

```bash
# Show only changes that touched the auth module
git log --oneline --grep="(auth)"
```

## Automated Changelogs and Semantic Release

Because every commit now follows a predictable structure, tools can read the log and produce a changelog automatically.

[`semantic-release`](https://github.com/semantic-release/semantic-release) is the most widely used tool for this. On every push to `main` it:

1. Reads all commits since the last tag.
2. Determines the next version number based on commit types (`fix` → patch, `feat` → minor, breaking → major).
3. Generates a `CHANGELOG.md` entry grouping commits by type.
4. Creates a Git tag for the new version.
5. Publishes the package to npm, PyPI, or another registry.

A typical `.releaserc.json` configuration:

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git"
  ]
}
```

An automatically generated `CHANGELOG.md` section looks like:

```text
## [2.1.0] - 2024-11-15

### Features
- **auth:** add OAuth2 login via Google (#412)
- **checkout:** add Apple Pay as a payment option (#388)

### Bug Fixes
- **cart:** prevent duplicate items on rapid double-click (#401)
- **api:** handle empty response body from payment gateway (#395)
```

## Enforcing Conventional Commits

### With a `commit-msg` Hook (reference: previous lesson)

The `commit-msg` hook you saw in the previous lesson already enforces this format. Add it to `.pre-commit-config.yaml` to share it with the team:

```yaml
repos:
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.27.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

### With `commitlint`

[`commitlint`](https://commitlint.js.org) is a JavaScript tool that validates commit messages against a configurable ruleset.

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

Run it in CI to catch any messages that slipped through:

```yaml
# .gitlab-ci.yml
lint-commits:
  image: node:20
  script:
    - npm ci
    - npx commitlint --from $CI_MERGE_REQUEST_DIFF_BASE_SHA --to HEAD
```

### With `commitizen`

[`commitizen`](https://commitizen-tools.github.io/commitizen/) turns commit creation into an interactive prompt so developers do not need to memorise the format.

```bash
pip install commitizen
```

```bash
# Instead of git commit, run:
cz commit
```

Output:

```text
? Select the type of change you are committing:
  feat     - A new feature
  fix      - A bug fix
  docs     - Documentation only changes
  ...
? What is the scope of this change? (press enter to skip): auth
? Write a short, imperative summary: add OAuth2 login via Google
? Provide a longer description? (press enter to skip):
? Are there any breaking changes? No
? Does this change affect any open issues? Yes
? Add issue references: Closes #412

feat(auth): add OAuth2 login via Google

Closes #412
```

## Common Pitfalls

- **Using `fix` for everything**: `fix` has a specific meaning (bug visible to users); using it for refactors or dependency bumps pollutes the changelog and leads to incorrect patch version bumps.
- **Vague descriptions**: "fix bug" and "update stuff" are useless to future readers; always describe what changed and where, not just that something changed.
- **Forgetting the breaking change marker**: a change that removes a public API or changes response shapes without a `BREAKING CHANGE` footer or `!` will produce a minor or patch release when it should be a major — this can break consumers.
- **Inconsistent scope naming**: using `Auth`, `auth`, and `authentication` interchangeably makes log filtering unreliable; agree on a scope list with your team and stick to it.
- **Trying to cram multiple changes into one commit**: if a commit touches the API, the database schema, and the frontend, the message cannot be accurate; prefer small, single-purpose commits.
- **Skipping the body when context matters**: a one-line message is fine for trivial changes, but complex bug fixes or architectural decisions deserve a body explaining the reasoning.

## Summary

- Commit messages are a long-lived record of why the codebase changed; clarity now saves hours of debugging later.
- Conventional Commits defines the format `<type>(scope): description` with optional body and footers.
- Core types: `feat` (MINOR), `fix` (PATCH), `chore`, `docs`, `refactor`, `test`, `ci`, `perf` (no bump by default).
- Breaking changes are signalled with `!` after the type or a `BREAKING CHANGE:` footer, triggering a MAJOR version bump.
- Write descriptions in imperative mood, lowercase, no trailing period, under 72 characters.
- Scopes are optional but valuable for filtering history and generating scoped changelogs.
- Tools like `semantic-release` read the structured history to automate version bumps and changelog generation.
- Enforce the format with a `commit-msg` hook, `commitlint` in CI, or `commitizen` for interactive commit creation.
