---
name: upstream-sync
description: Manually synchronize useful changes from the original wynainfo/pi-permission-modes upstream into this fork, using the local sync/dev branch and one squashed "chore: sync upstream changes" commit on dev. Use only when the user explicitly asks to sync upstream or inspect and import an upstream change. sync/dev is local and must never be pushed.
disable-model-invocation: true
---

# Upstream sync

Use this workflow only on an explicit request. Do not sync upstream on every session, and do not run any step automatically just because a session started.

## Branch roles

- `upstream/dev` - the original project, fetched from the `upstream` remote.
- `sync/dev` - a **local-only** bridge that records the upstream ancestry. It is never pushed.
- `dev` - this fork's development branch. It should contain this fork's commits plus one `chore: sync upstream changes` commit per sync.
- `main` - release branch; update it separately when a release is cut.

The repository already has `sync/dev` as a merge of the local baseline and `upstream/dev`. Do not recreate it, reset it, or use `git merge -s ours` again in normal syncs.

## Preflight

```bash
git status --short --branch
git fetch upstream
```

Stop if the working tree is dirty. Confirm the upstream remote is the original project and not this fork:

```bash
git remote -v
```

## Inspect what is new

```bash
git log --oneline --no-merges sync/dev..upstream/dev
git diff --stat sync/dev..upstream/dev
```

Inspect candidate commits with `git show <sha>`. If nothing is listed, there is nothing to sync; stop.

## Full sync

Use this when the user wants all new upstream changes.

```bash
# 1. Bring upstream into the local bridge. Resolve conflicts here, not on dev.
git switch sync/dev
git merge upstream/dev

# 2. Squash the bridge onto dev as exactly one commit.
git switch dev
git merge --squash sync/dev
git commit -m "chore: sync upstream changes" \
  -m "Upstream: $(git rev-parse --short upstream/dev)"

# 3. Record dev in the bridge so the next sync only picks up new upstream work.
git switch sync/dev
git merge dev -m "Record dev in sync branch"

# 4. Verify the two branches have the same tree.
git switch dev
git diff --stat dev sync/dev
```

`git diff --stat dev sync/dev` must be empty. If either merge reports `Already up to date.`, skip the following commit; do not create an empty commit.

## Conflict notes

Conflicts usually happen in files both sides changed:

- `package.json`, `package-lock.json` - keep this fork's version and metadata unless the user is deliberately taking the upstream version.
- `CHANGELOG.md` - keep the fork's sections and add the new upstream entries into the appropriate section; do not drop the fork's entries.
- `README.md`, `docs/`, `CONTRIBUTING.md`, `SECURITY.md` - keep the fork's repository URLs, author and fork notes, while taking useful upstream wording.
- `permission-mode.defaults.json` - take upstream default changes deliberately. If upstream changed a default, keep its matching `defaults-history.json` entry; do not duplicate or invent one.

Resolve conflicts on `sync/dev` during step 1. If the merge cannot be finished:

```bash
git merge --abort
```

If `git merge --squash sync/dev` conflicts on `dev`, resolve, `git add` the files, then commit with the sync message. Do not use `git reset --hard` to discard the user's work.

## Selective sync

Use this only when the user wants one specific upstream fix, not the whole upstream branch.

```bash
git log --oneline sync/dev..upstream/dev
git show <sha>

git switch dev
git cherry-pick -x <sha>
```

Caveats:

- `sync/dev` does not record a cherry-picked commit as upstream-merged.
- A later full sync can apply the same upstream change again or conflict. Mention this to the user and resolve it then.
- Prefer a full sync plus reverting unwanted changes when the user expects to stay close to upstream.

## Push policy

`sync/dev` stays local. Never run:

```bash
git push origin sync/dev
```

Only push `dev` when the user asks to publish it:

```bash
git push origin dev
```

Create tags and GitHub Releases as a separate, explicit release step. Do not tag from this skill unless the user asks.

## First-time initialization (already done in this repository)

Only for a fresh clone where `sync/dev` does not exist. Use the fork baseline commit that is already in `dev`, not a newer upstream tip:

```bash
git switch -c sync/dev dev
git merge -s ours --allow-unrelated-histories <baseline-upstream-sha> \
  -m "Record upstream/dev baseline"
```

Then fetch and merge newer upstream commits normally. Do not use `-s ours` for later upstream updates: it would mark new commits as merged while discarding their tree.
