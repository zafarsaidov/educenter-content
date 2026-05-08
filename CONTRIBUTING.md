# Contributing

Thanks for editing EduCenter lesson content. This guide covers the workflow for
adding new lessons or improving existing ones.

---

## Quick start

1. **Branch** from `main`:
   ```bash
   git checkout -b edit/<short-description>
   # e.g. edit/linux-permissions-clarify-octal
   ```

2. **Edit** the relevant `.md` file. Follow [STYLE.md](./STYLE.md).
   When writing a lesson from scratch, copy the skeleton from
   [LESSON-TEMPLATE.md](./LESSON-TEMPLATE.md) — it captures the canonical
   structure (opening paragraph → H2 body sections → Common Pitfalls → Summary).

3. **Commit** with a clear message — see [commit conventions](#commit-conventions) below.

4. **Push** and open a pull request. Use the [PR template](#pr-template).

5. An instructor reviews. After merge, the platform picks up the change on the next
   refresh (typically within minutes; instantly if the GitHub webhook is configured).

---

## What kind of edits are welcome

- ✅ Clarifying a confusing explanation.
- ✅ Adding a missing example or code snippet.
- ✅ Fixing typos, grammar, broken links, outdated commands.
- ✅ Adding cross-references to related lessons.
- ✅ Updating commands or flags that have changed in newer tool versions.
- ✅ Adding a new lesson (coordinate with the platform team — DB row needs to exist
  first, and the path must match `<course>/<section>/<theme>.md`).

## What does NOT belong here

- ❌ Exam questions, answers, scoring rubrics, or anything that defeats the platform's
  assessment integrity.
- ❌ Lab verification scripts or solutions to challenges.
- ❌ Hints / walkthroughs for assessed exercises.
- ❌ `TODO:` / `XXX:` / `FIXME:` comments — file a GitHub Issue or PR review comment
  instead.
- ❌ Drafts of unreleased courses on `main` — use a feature branch and merge only
  when public-ready.
- ❌ Binary assets (images, PDFs, video). Embed images by URL from the media CDN; see
  [STYLE.md → Images](./STYLE.md#images).
- ❌ PII, credentials, or anything you wouldn't want indexed by search engines.

This repository is **public**. Anything pushed to `main` is permanent in the git
history, even after a follow-up commit removes it.

---

## Commit conventions

Follow [Conventional Commits](https://www.conventionalcommits.org/) for human and
tooling friendliness:

```
<type>(<scope>): <subject>

<optional body>
```

Common types:
- `docs:` — content edits (the most common type in this repo)
- `fix:` — typo, broken link, factual error
- `feat:` — new lesson added
- `chore:` — repo housekeeping (README, CI tweaks)

Scope is usually the course slug or `course/section`:

```
docs(linux-basic/filesystem): clarify octal vs symbolic chmod
fix(kubernetes-basic): broken link to kubectl docs
feat(ansible-basic): add new lesson on inventory variables
```

Keep the subject under ~70 characters. Use the body for the *why* if it isn't obvious
from the diff.

---

## Branch naming

- `edit/<short-description>` — small content edits.
- `add/<theme-slug>` — adding a new lesson file.
- `restructure/<course>` — reordering or splitting sections in a course.

Avoid long-running personal branches. Merge or close within a few days.

---

## Pull request template

Open the PR with this minimal description (the repository's PR template auto-fills it):

```markdown
## What changed
<one-sentence description>

## Why
<motivation: student feedback / outdated info / new tool version / etc.>

## Lesson(s) affected
- `linux-basic/filesystem/permissions.md`

## Verification
- [ ] Style checklist in STYLE.md passed
- [ ] No exam answers, lab solutions, or hints added
- [ ] Internal links resolve
- [ ] Code blocks have language tags
- [ ] Previewed on GitHub before requesting review
```

---

## Reviewer guidelines

For instructors reviewing PRs:

1. **Spot-check the technical accuracy** — at least one command in the change should
   be one you can run yourself or independently verify.
2. **Check the assessment-integrity boundary** — did the contributor accidentally
   include something that would help a student cheat on the corresponding exam or lab?
3. **Verify the style checklist** — single H1, code-block language tags, no
   frontmatter, kebab-case filename matching theme slug.
4. **Test the rendering** — GitHub's "Files changed" tab shows the rendered markdown
   for `.md` files. A 30-second preview catches most issues.

If something is wrong but small, push a fix-up commit to the contributor's branch
yourself instead of bouncing the PR back. Faster for everyone.

---

## Adding a new lesson — full procedure

A new lesson requires coordination with the platform team because the database has to
know about the lesson before it can render.

### Steps

1. **Platform side first.** An admin creates the `Theme` row in the database (via the
   admin panel) with a slug like `permissions`. The `markdown_file_path` field gets
   set to the future path: `linux-basic/filesystem/permissions.md`.

2. **Then this repo.** Branch off `main`:
   ```bash
   git checkout -b add/linux-basic-permissions
   mkdir -p linux-basic/filesystem    # if the section folder doesn't exist yet
   $EDITOR linux-basic/filesystem/permissions.md
   ```

3. **Write the lesson** following [STYLE.md](./STYLE.md). Single H1, code blocks
   tagged, no frontmatter, no exam answers.

4. **Open a PR.** Reviewer merges.

5. **Verify on the platform.** The lesson should appear in the lesson viewer under
   the course's section. The AI tutor should pick up the new content within minutes.

If steps 1 and 2 happen in the wrong order, the platform will throw a 404 when
fetching content. That's fine and recoverable — just merge the markdown PR and trigger
a content refresh.

---

## Updating the per-course outline

Each course folder has a `README.md` with the course outline (curriculum). When you
add or remove a lesson, update that file in the same PR so the outline stays current.

---

## Tooling

- [markdownlint](https://github.com/DavidAnson/markdownlint) — runs on PRs (if `.github/workflows/lint.yml` is configured) and locally:
  ```bash
  npx markdownlint-cli '**/*.md'
  ```

- GitHub's online editor is fine for one-line typo fixes; for anything larger, clone
  the repo and use a real editor with a markdown preview.

---

## Questions

- **Found a factual error you can't verify yourself?** Open a GitHub Issue tagged
  `needs-verification` and an instructor will pick it up.
- **Lesson reads poorly but you can't articulate why?** Open an Issue tagged
  `polish` — fresh eyes are valuable even without a concrete fix.
- **Confused about the platform side?** Reach out to the platform team in your usual
  channel.
