# Tags and Releases

Tags are fixed pointers to specific commits — unlike branches, they never move. They are the standard way to mark release points in a project's history so that you can always return to exactly the code that was shipped as version `v2.3.1`.

## What Is a Tag?

A branch pointer advances with every new commit. A tag is permanently anchored to the commit it was created on. Think of a branch as a sticky note on the latest page of a book, and a tag as a bookmark glued to a specific page — it will always open to that exact spot.

Tags are most often used to mark releases, but you can tag any commit for any reason (e.g., marking a stable checkpoint before a risky refactor).

## Lightweight Tags

A lightweight tag is the simplest form — just a named pointer to a commit, nothing more.

```bash
git tag v1.0.0
```

This tags the current `HEAD` commit. To tag a specific commit:

```bash
git tag v1.0.0 a1b2c3d
```

Lightweight tags have no metadata — no author, no date, no message. They are useful for quick personal bookmarks but are not recommended for public releases.

## Annotated Tags

An annotated tag is a full Git object. It stores the tagger's name, email, date, and a message. It can also be signed with GPG.

```bash
git tag -a v1.0.0 -m "Release 1.0.0 — initial public release"
```

Always use annotated tags for release versions. They appear correctly in `git describe`, carry metadata that tools and forges (GitLab, GitHub) can display, and can be cryptographically signed for supply-chain verification.

To tag a specific past commit:

```bash
git tag -a v1.0.0 a1b2c3d -m "Release 1.0.0"
```

## Listing Tags

```bash
git tag
```

Lists all tags alphabetically.

### Filtering Tags

```bash
git tag -l "v1.*"
```

Sample output:

```text
v1.0.0
v1.1.0
v1.2.3
```

The `-l` flag accepts glob patterns. You can also use `--sort` to control ordering:

```bash
git tag --sort=-version:refname
```

This sorts tags in descending version order, so the newest appears first.

## Inspecting a Tag

```bash
git show v1.0.0
```

For an annotated tag, this shows the tag object (tagger, date, message) followed by the commit the tag points to and its diff. For a lightweight tag it shows the commit directly.

## Deleting a Local Tag

```bash
git tag -d v1.0.0
```

This only removes the tag from your local repository. The remote is unaffected.

## Pushing Tags to a Remote

Tags are **not** pushed automatically when you run `git push`. You must push them explicitly.

### Push a Single Tag

```bash
git push origin v1.0.0
```

### Push All Local Tags

```bash
git push origin --tags
```

This pushes every local tag that the remote does not yet have. Be careful — this includes any temporary lightweight tags you may have created locally.

A safer approach is to push only annotated tags:

```bash
git push origin --follow-tags
```

`--follow-tags` pushes annotated tags reachable from the commits being pushed. It skips lightweight tags.

## Deleting a Remote Tag

```bash
git push origin --delete v1.0.0
```

Or using the older refspec syntax:

```bash
git push origin :refs/tags/v1.0.0
```

Both are equivalent. Note that anyone who has already fetched the tag will still have it locally — coordinate with your team before deleting public tags.

## Semantic Versioning

Most projects follow [Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`, always prefixed with `v`.

```text
v1.2.3
│ │ └── PATCH — backwards-compatible bug fix
│ └──── MINOR — new backwards-compatible feature
└────── MAJOR — breaking change that is not backwards-compatible
```

Examples:

| Change                             | Old version | New version |
|------------------------------------|-------------|-------------|
| Fix a crash in the login handler   | `v2.1.4`    | `v2.1.5`    |
| Add a new `/export` API endpoint   | `v2.1.5`    | `v2.2.0`    |
| Remove the deprecated `/v1` API    | `v2.2.0`    | `v3.0.0`    |

During initial development, use `v0.x.y` — the `0` major version signals that breaking changes may occur without a MAJOR bump.

## Checking Out a Tag

You can inspect the code at a tag at any time:

```bash
git checkout v1.0.0
```

This puts you in **detached HEAD** state — you are no longer on a branch. Any commits you make here are not attached to any branch and will be lost if you switch away without first creating a branch.

```text
HEAD detached at v1.0.0
```

### Making Changes from a Tagged Commit

If you need to apply a hotfix to a release, create a branch from the tag:

```bash
git switch -c hotfix/v1.0.1 v1.0.0
```

Now you are on a real branch rooted at `v1.0.0`. After making and testing your fix, tag the new commit:

```bash
git tag -a v1.0.1 -m "Hotfix: correct null pointer in auth module"
git push origin hotfix/v1.0.1 --follow-tags
```

## Common Pitfalls

- Using lightweight tags for releases — annotated tags are strongly preferred because they carry metadata and work correctly with `git describe` and release tooling.
- Forgetting that `git push` does not push tags — always push tags explicitly with `git push origin <tag>` or `--follow-tags`.
- Deleting a tag that has already been published and used in CI pipelines — coordinate with the team and check whether any system (Docker registry, package repo, changelog generator) references the tag.
- Checking out a tag and committing in detached HEAD without creating a branch first — those commits are easy to lose.
- Naming tags without the `v` prefix (e.g., `1.0.0` instead of `v1.0.0`) — many tools expect the `v` prefix; choose a convention and stick to it across the entire project.

## Summary

- Tags are permanent pointers to specific commits — they do not move as new commits are added.
- Lightweight tags are simple pointers with no metadata; annotated tags are full Git objects with author, date, and message.
- Use annotated tags (`git tag -a`) for all releases.
- `git tag -l "v1.*"` filters tags by pattern; `git show <tag>` inspects a tag.
- Tags are not pushed automatically — use `git push origin <tag>` or `git push origin --follow-tags`.
- Delete a remote tag with `git push origin --delete <tag>`.
- Semantic versioning (`MAJOR.MINOR.PATCH`) gives tags a meaningful, predictable structure.
- Checking out a tag enters detached HEAD state; create a branch with `git switch -c <branch> <tag>` before making any commits.
