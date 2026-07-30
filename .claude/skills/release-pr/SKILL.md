---
name: release-pr
description: >
  Open a release PR for stream-automatic-release-test: bump the version(s) of one or more packages, finalise their
  CHANGELOGs (promote `## Upcoming` → `## X.Y.Z`), and open a PR from a `release/` branch. Per-package independent
  versioning — release one package or several in a single PR.
disable-model-invocation: true
argument-hint: "[<package> <version> ...]"
arguments: [packages]
allowed-tools:
  - Bash(git *)
  - Bash(gh *)
  - Bash(melos *)
  - Bash(which *)
  - Bash(grep *)
  - Bash(sed *)
  - Read
  - Edit
  - Write
---

# release-pr

Opens a release PR. Branch `release/<...>` → base `main` → title `chore(repo): release <...>`.

**This skill only opens the PR.** After merge, tagging and pub.dev publishing are automatic:
`release_tag.yml` tags every bumped package (`<pkg>-vX.Y.Z`) and `release_publish.yml` publishes each (OIDC) and
cuts a per-package GitHub Release.

## Key facts

- **Independent per-package versioning.** Each package releases on its own tag `<pkg>-vX.Y.Z`. A single release PR
  may bump **one package or several** — each gets its own tag + publish run.
- **CHANGELOGs are hand-curated.** Never run `melos version`. Releasing means *promoting* the existing `## Upcoming`
  heading to `## X.Y.Z`, not rewriting it.
- List packages with `melos list --no-private`.

## Inputs

1. **Which packages + versions.** If given as args (e.g. `/release-pr stream_automatic_release_package_one 2.2.0`),
   use them; strip any leading `v`. Otherwise **detect and confirm**: a package needs releasing when its
   `CHANGELOG.md` has a non-empty `## Upcoming` section. List those and ask the user for each new version.
2. **Base branch** is always `main`.

## Pre-flight

Run these. **If any fails, stop and surface it — do not auto-fix.**

- `git checkout main && git pull --ff-only` leaves `git status --short -uno` clean.
- `which melos`, `gh auth status` succeed.
- No open release PR for the same branch: `gh pr list --head <branch> --state all --json number` returns `[]`.

## Steps

### 1. Branch off main

`git checkout -b <branch>` — `release/<pkg>-vX.Y.Z` for a single package, or `release/YYYY-MM-DD` for several.

### 2. Bump version(s)

For **each** package being released: set `version: <newver>` in `packages/<pkg>/pubspec.yaml`.

If a released package is a **dependency a dependent must now require at the new version**, also bump that constraint
in the dependent's `pubspec.yaml` and release the dependent too. Then propagate:

```bash
melos bootstrap
```

Do **not** run `melos version`.

### 3. Finalise each released package's CHANGELOG

For every package being released, rename the top `## Upcoming` heading in `packages/<pkg>/CHANGELOG.md` to
`## <newver>`. Keep the curated bullets exactly. A package released only because of a dependency bump gets a
`## <newver>` section with a single bullet noting the bump. Every released package must have a non-empty
`## <newver>` section (an empty one fails publishing).

### 4. Sanity-check

```bash
melos run lint:pub
```

If it fails, surface it and stop.

### 5. Commit and push

```bash
git add -A
git commit -m "<title>"
git push -u origin <branch>
```

Single commit. **The title is load-bearing** — `release_tag.yml` gates on the `chore(...): release` prefix:

- One package: `chore(repo): release <package> vX.Y.Z`.
- Several: `chore(repo): release packages` — generic, so the title stays short.

Tagging derives from package state, not this title, so a typo can't mis-tag — but keep the prefix intact.

### 6. Open the PR

Build the body from the promoted CHANGELOG sections (the same content that becomes each GitHub Release):

```bash
gh pr create --base main --head <branch> --title "<title>" --body-file <notes>
```

Return the PR URL.

## Don't

- **Never run `melos version`** — it clobbers the hand-curated CHANGELOGs.
- **Never tag or push a tag** — `release_tag.yml` does it on merge.
- **Never run `melos run release:pub` / `release:tag`** locally — those are the CI publish/tag steps.
- **Never create a GitHub release** — `release_publish.yml` creates it after the tag is pushed.
- **Never merge the PR.** Return the URL and stop.
