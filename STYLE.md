# Style Guide

Conventions for writing lesson notes in this repository. Following them keeps the
markdown clean, makes lessons feel consistent across courses, and helps the AI lesson
tutor produce better answers.

---

## File structure

### One H1, at the top
Each `.md` file has exactly one `# Heading 1` — and it's the first non-blank line.
This H1 should match the lesson's title in the platform.

```markdown
# File permissions in Linux

(intro paragraph...)

## Symbolic notation

(content...)
```

❌ Don't use multiple H1s. Don't put the H1 lower in the file.

### Recommended section flow
Every lesson follows the same skeleton — see [`LESSON-TEMPLATE.md`](./LESSON-TEMPLATE.md)
for a copy-paste starting point. The canonical structure:

1. **H1 title** (one line, matches the platform's `Theme.title`).
2. **Opening paragraph** — 2–4 sentences: what / why / what-the-student-can-do.
3. **Body H2 sections** — focused topics, each grouping its sub-cases under H3s.
4. **Key Concepts** (optional) — glossary of newly-introduced terms.
5. **Workflow / Visual Overview** (optional) — ASCII diagram showing relationships.
6. **Common Pitfalls** — 3–6 bullets of `mistake — why it's wrong`.
7. **Summary** — 3–6 takeaway bullets.

Sections (1), (2), (3) and (7) are mandatory. The rest are situational. Aim for
~100–150 lines / 6–9 KB total — long enough to stand alone as a reference, short
enough to skim before a related video.

### No frontmatter
The platform database owns metadata (title, order, duration, video status, slug).
Don't add a YAML or JSON frontmatter block at the top of lesson files. Start with the
H1.

### Hierarchy
- `#` — lesson title (one per file)
- `##` — major sections within the lesson
- `###` — sub-sections, used sparingly
- Avoid `####` and deeper — if you need them, the lesson is too long; split it.

---

## Code blocks

### Always tag the language
Helps GitHub render syntax highlighting and gives the AI tutor a hint about what
shell or language is in play.

````markdown
```bash
chmod 644 file.txt
```

```python
def hello():
    print("hi")
```

```yaml
name: deployment
spec:
  replicas: 3
```
````

❌ Don't use untagged ``` ` ``` blocks for commands.

### Use inline code for single tokens
File names, flags, environment variables, and short commands go in single backticks:
`chmod`, `--no-recreate`, `KUBECONFIG`.

---

## Language

### Prose language
This repository is written in **English** — that's the canonical language for code
samples, command output, and the lesson notes themselves. The platform's AI tutor
auto-detects the student's question language at runtime and replies in Uzbek, English,
or any code-mix the student writes — it does NOT need the source notes to be Uzbek.
Keeping notes in English means a single source of truth that stays consistent across
courses.

If a future course needs Uzbek prose for pedagogical reasons, that's fine — just keep
the choice consistent within that course. In all cases:

- **Commands, flags, code, file paths, technical terms:** English, always.
- **Prose:** English by default; mixed/Uzbek only if the course owner deliberately chooses it.

### Don't translate command names
`chmod`, `kubectl`, `systemctl`, `apt`, `iptables`, `ssh`, `git` — they keep their original spelling.

---

## Wrapping

Choose ONE per repo and stick with it:

- **Hard-wrap at ~100 characters** (easier review on narrow terminals), OR
- **No wrap; one paragraph per line** (lets git track sentence-level changes).

Mixed wrapping in the same file is the worst case — consistent diffs become impossible.

This repo's choice: **no wrap**. One sentence-or-paragraph per logical line. If a paragraph spans multiple sentences, keep them on one line.

---

## Images

Don't put binary images (PNG, JPG, SVG) in this repo. Embed them by URL from the
platform's media CDN:

```markdown
![Linux file permissions diagram](https://fsn1.your-objectstorage.com/educenter-media/lessons/linux-basic/permissions-diagram.png)
```

Upload images to `s3://<media-bucket>/lessons/<course-slug>/<image>.<ext>` via the
platform's admin tools. The Hetzner object storage origin is pre-allowlisted in the
platform's nginx CSP.

Why this convention:
- Repo stays small and text-only — fast clones, clean diffs.
- Images can be optimised or replaced without polluting git history.
- One hosting story (same as the platform blog).

Always include meaningful alt text for accessibility and AI grounding.

---

## Links

### Internal lesson links
Link to other lessons by their relative path:

```markdown
See also [File permissions](../filesystem/permissions.md).
```

### External links
Use plain markdown link syntax. Don't wrap URLs in autolinks (`<https://...>`) — most
markdown renderers handle bare URLs poorly.

```markdown
- [Linux man pages online](https://man7.org/linux/man-pages/)
- [Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
```

---

## Lists

- Use `-` for unordered lists (consistent across the repo, easier to scan in raw markdown).
- Use `1.` for ordered lists; auto-numbering is fine — markdown renderers will display
  sequentially even if every item is `1.`, but humans skim the source so use real
  numbers when ordering matters.
- Indent nested items with **two spaces**, not four.

---

## Tables

Use tables sparingly. They render fine on GitHub but are awkward to maintain.

```markdown
| Permission | Octal | Symbolic |
|-----------:|:-----:|:---------|
| read       | 4     | r        |
| write      | 2     | w        |
| execute    | 1     | x        |
```

Always include the header separator row. Pipe-align columns for readability in
the raw source.

---

## What NOT to write

This repository is **public**. Don't include:

- ❌ Exam questions, answers, scoring rubrics, or anything that would defeat the
  platform's assessment integrity.
- ❌ Lab verification scripts (the bash `exit 0`/`exit 1` checks).
- ❌ Walkthroughs or hints that give away challenge solutions.
- ❌ Internal `TODO:`, `XXX:`, or `FIXME:` notes — open a GitHub Issue instead.
- ❌ Drafts of unreleased courses on `main` — use feature branches and merge only
  when public-ready.
- ❌ Student PII, credentials, internal links, or anything you wouldn't want a
  competitor to read.

A useful test: would a paying student feel cheated if they saw this for free? If yes,
don't include it.

---

## Pre-merge checklist

Before opening a PR, scan your edit for:

- [ ] Exactly one H1 at the top.
- [ ] Every code block has a language tag.
- [ ] Code, flags, and commands stay in English.
- [ ] No frontmatter, no internal `TODO:` notes.
- [ ] Images embedded by URL, not committed as binaries.
- [ ] Links resolve (relative paths to other lessons exist).
- [ ] Filename is the theme's slug (e.g. `permissions.md`, kebab-case if multi-word).
- [ ] No accidental exam answers or lab solutions.
