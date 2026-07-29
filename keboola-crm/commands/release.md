---
description: 'Cut a unified CRM release end-to-end: work out the next version from the commits since the last tag, draft `docs/releases/vX.Y.Z.md` in the in-app changelog style, get your approval, then bump `/VERSION`, commit, and run `make release` — which tags, pushes, publishes the GitHub release, and triggers the automatic production deploy. Stops for explicit approval before anything leaves the machine: the tag push deploys production and announces the release in Slack, and neither is undoable. Refuses to run outside `main`, on a dirty tree, or when the tag already exists.'
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: '[optional version, e.g. 0.59.0 or patch/minor/major; inferred from the diff if omitted]'
---

# /keboola-crm:release — cut a CRM release

One command from "main is ready" to "production is serving the new
version and Slack knows about it".

**`make release` does NOT bump `/VERSION`.** It reads whatever is in
that file and releases *that*. If you run it without bumping first, it
tries to re-tag the version that already shipped and aborts on
`Tag vX.Y.Z already exists.` The bump and the release notes must be
committed to `main` *before* the target runs, because it also refuses a
dirty working tree. Getting that order right is most of what this
command is for.

## What the release train actually does

Pushing the tag is the deploy. It triggers
[`.github/workflows/deploy-prod.yml`](../../../.github/workflows/deploy-prod.yml),
which restarts the production Data App, waits for the released version
to serve, verifies the deployed commit contains the tag, and posts the
announcement to `#tmp_agentic_crm`. So the tag push is **outward-facing
and effectively irreversible** — there is no rollback primitive, by
design. That is why this command has a hard approval gate.

`make release` also auto-bumps `cli/VERSION` and `mcp_server/VERSION`
when those subtrees changed since their last bump, regenerates
`docs/MODULES.json`, runs `scripts/check-versions.sh`, commits
`chore: release vX.Y.Z`, pushes `main`, tags, pushes the tag, and calls
`gh release create` with the notes minus their frontmatter.

## Behavior

### 1. Refuse early if the ground is not right

Run these and stop on the first failure, reporting what the user must
fix — do not try to fix any of it yourself:

```bash
git rev-parse --abbrev-ref HEAD      # must be main
git status --porcelain               # must be empty
git fetch origin --tags -q && git status -sb | head -1   # must not be behind
```

If the user is on a feature branch or has uncommitted work, say so and
stop. Their working tree is theirs; never stash, reset, or check out
over it.

### 2. Work out the version

You bump `/VERSION`; the user never has to. They only approve the
result in step 4.

```bash
CURRENT=$(tr -d '[:space:]' < VERSION)
LAST_TAG=$(git tag --sort=-v:refname | head -1)
git log "$LAST_TAG..HEAD" --oneline --no-merges
```

Diff from the **last tag**, not from `v$CURRENT`. If an earlier attempt
bumped `/VERSION` without completing the release, no `v$CURRENT` tag
exists and `git log v$CURRENT..HEAD` fails outright. Detect that case
and do not double-bump:

```bash
git rev-parse "v$CURRENT" >/dev/null 2>&1 || echo "VERSION already ahead of the last release"
```

When it fires, ship `$CURRENT` as-is and say so — the bump already
happened, it just never got released.

Otherwise pick `$NEW`: `$ARGUMENTS` if it names a version; `patch` /
`minor` / `major` applied to `$CURRENT` if it names one of those;
otherwise infer from the commit subjects. **Commits in this repo are
scoped** — `feat(hr):`, `fix(rbac):`, `test(exports):` — so match the
type before the optional `(scope)`, never a bare `feat:`, which would
miss almost everything:

```bash
git log "$LAST_TAG..HEAD" --no-merges --format=%s \
  | grep -qE '^feat(\(|!|:)' && echo minor || echo patch
```

`fix` and the rest fall into the `patch` branch on purpose — they do not
need matching, only `feat` changes the decision.

**Do not let an unprefixed commit decide the bump by omission.** About
1 in 20 commits here carries no conventional prefix at all (`Add
Account.billing_email …`, `Unify opportunity activity logging …`), and
some of those are features. They silently fail the `feat` test and land
in `patch`. List them and read them before deciding:

```bash
git log "$LAST_TAG..HEAD" --no-merges --format=%s \
  | grep -vE '^[a-z]+(\([^)]*\))?!?:'
```

If any describe user-visible new capability, treat the release as
`minor`. When it is genuinely ambiguous, put the question in the step-4
approval rather than guessing — the user is already reading that
message.

A `major` bump is never inferred: this repo has no `!:` or
`BREAKING CHANGE` commits in its history, so it must be requested
explicitly via `$ARGUMENTS`.

State which you picked and why, in one line.

Then confirm the tag is free:

```bash
git rev-parse "v$NEW" >/dev/null 2>&1 && echo "TAG EXISTS — stop"
```

### 3. Draft the release notes

Write `docs/releases/v$NEW.md`. **This file is the user-facing changelog
rendered in-app at `/releases`** — it is read by salespeople and CSMs,
not by engineers. Match the narrative style of the three most recent
files; read them first rather than guessing.

Required frontmatter, exactly this shape:

```markdown
---
title: "v0.59.0 — <one-line theme, no trailing period>"
date: <YYYY-MM-DD>
tag: v0.59.0
---
```

Section convention — these three carry the release and appear in every
recent one, in this order:

- `## Highlights` — the headline features, each opening with a **bold
  lead sentence** stating what the user can now do, then a short
  paragraph of what changed and why it matters.
- `## Also in this release` — smaller user-visible changes, one bullet
  each.
- `## Notes` — upgrade notes, behaviour changes, anything an operator
  should know.

The list is not a closed set: a release with a distinct theme may add a
section of its own between `Highlights` and `Also in this release` —
`v0.58.0.md` adds `## Performance and reliability`. Add one when the
release earns it, rather than forcing everything into the three.

Two details that are easy to get wrong:

- **The first paragraph under `## Highlights` is what lands in Slack.**
  [`scripts/release_slack_summary.py`](../../../scripts/release_slack_summary.py)
  extracts exactly that, truncates it to ~600 characters on a word
  boundary, and posts it. Write it so it stands alone as an
  announcement.
- **Mention the CLI.** If `cli/` changed since its last bump, say the
  CLI is bumped and that users get it via `crm update`. If it did not,
  write "CLI unchanged at vX.Y.Z" so readers know it was considered
  rather than forgotten. Check with:
  ```bash
  git diff --quiet "$(git log -1 --format=%H -- cli/VERSION)"..HEAD -- cli/ && echo unchanged || echo changed
  ```

Translate commits into user-facing outcomes. A reader should learn what
they can now do, not which modules were touched. Internal-only work
(CI, refactors, test fixes) belongs in `## Notes` at most, usually
nowhere.

### 4. Show the draft and STOP

Present the full notes and the version to the user, plus a one-line
summary of what will happen when they approve: tag pushed → production
redeployed → Slack announcement to `#tmp_agentic_crm`.

**Wait for explicit approval.** Do not proceed on silence, on "looks
good" about something else, or on a general earlier authorisation to
"do a release" — the content of the announcement is what is being
approved here. If the user wants changes, revise and show it again.

### 5. Bump, commit, release

Only after approval:

```bash
echo "$NEW" > VERSION
git add VERSION "docs/releases/v$NEW.md"
git commit -m "chore: prepare release v$NEW"
git push origin main
make release
```

Skip the `echo`/`git add VERSION` when step 2 found `/VERSION` already
ahead of the last release — it is at `$NEW` already, and only the notes
need committing.

`make release` produces its own commit and pushes the tag. Do not tag by
hand.

### 6. Watch the deploy

The tag push triggers the deploy. Follow it to completion rather than
declaring victory at the tag:

```bash
sleep 15 && gh run list --workflow=deploy-prod.yml --limit 1
gh run watch <run-id> --exit-status --interval 20
```

Report the outcome honestly. If the run is red, read the log before
suggesting anything — **re-running restarts production again**, and the
workflow's own failure messages distinguish "still building" from
"rebuild failed" from "the deployed commit does not contain the tag".
Do not re-run to see if it passes the second time.

On success, confirm what the user can check:

- the in-app changelog at `<app-url>/releases?v=v$NEW`
- the Slack announcement in `#tmp_agentic_crm`
- `/api/health` reporting the new `version` and `git_commit`

## Rehearsing without releasing

To exercise the deploy path against the **Test** app instead of
production — no tag, no Slack announcement to real readers (the message
is prefixed `🧪 [rehearsal]`), no production impact:

```bash
gh workflow run "Deploy Prod Data App" -f tag=<existing-tag> -f target=test
```

Suggest this when the user wants to check the machinery rather than ship
a release, or when the deploy workflow itself has just changed.

## Hard rules

- **Never push a tag without explicit approval of the notes.** The tag
  is the deploy and the announcement.
- **Never bump `/VERSION` without writing the notes** — `make release`
  will abort, leaving a committed version bump with no release.
- **Never edit a released `docs/releases/*.md`.** Published notes are a
  record; corrections go in the next release.
- **Never run from a feature branch or a dirty tree**, and never clean
  the tree on the user's behalf.
- **Never re-run a failed deploy to see if it works.** Read the log.
