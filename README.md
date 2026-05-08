# EduCenter — Lesson Content

Public repository of markdown lesson notes for the **EduCenter** learning platform
([mvp.zafarsaidov.uz](https://mvp.zafarsaidov.uz)). Each `.md` file is the companion notes
for a video lesson — students click **"Konspektni ko'rish ↗"** in the lesson viewer to land
here. The platform's AI tutor also uses these files as its grounding context when answering
student questions about a lesson.

---

## Structure

```
<course-slug>/<section-slug>/<theme-slug>.md
```

Example: `linux-basic/filesystem/permissions.md`

The folder hierarchy mirrors the platform's `Course → Section → Theme` model. Slugs match
the platform database, so paths are stable across title rewordings.

---

## Courses

<!-- Update this list when courses are added or removed. -->

- [Linux Basic](./linux-basic/)
- [Linux Advanced](./linux-advanced/)
- [Kubernetes Basic](./kubernetes-basic/)
- [Kubernetes Advanced](./kubernetes-advanced/)
- [Ansible Basic](./ansible-basic/)
- [Docker Basic](./docker-basic/)

Each course folder contains its own `README.md` with the curriculum.

---

## Editing

Anyone with write access can edit a lesson:

1. Branch from `main`.
2. Edit the relevant `.md` file. Follow [STYLE.md](./STYLE.md).
3. Open a pull request.
4. An instructor reviews and merges.
5. The platform re-fetches content within minutes (or instantly via webhook, depending on
   deployment).

For more, see [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## What lives here

- ✅ Lesson markdown — explanations, examples, code snippets, references.
- ✅ Per-course `README.md` curriculum outlines.

## What does NOT live here

- ❌ Exam questions, answers, or rubrics.
- ❌ Lab verification scripts or solutions.
- ❌ Hints or walkthroughs that defeat the platform's assessment integrity.
- ❌ Images / screenshots — embed via the platform's media CDN URL instead.
- ❌ Anything sensitive: PII, internal communications, draft course outlines.

This repository is **public**. Treat any commit to `main` as permanent — even if you remove
content later, the git history retains it.
