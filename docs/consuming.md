---
type: Guidance
title: Using bot-automerge-action in a consuming repo
description: The caller job to add, the pull_request_target trigger, write scopes and tokens it requires, the merge-safety prerequisite, and how Dependabot keeps the pin current.
tags: [consumer, setup, ci, auto-merge]
---

# Using bot-automerge-action in a consuming repo

Add one caller workflow, in one of two shapes. Both enforce the same eligibility
policy from the same pinned CLI, both pin by SHA, and both are bumped by
Dependabot's `github-actions` ecosystem — they differ only in what your repo owns.

|                                   | [Reusable workflow](#shape-a--the-reusable-workflow-recommended) | [Composite Action](#shape-b--the-composite-action) |
| --------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------- |
| Caller size                       | 3 lines                                                          | a full job                                         |
| Skip guard for non-bot PRs        | **built in**                                                     | you copy it in                                     |
| release-please PAT                | `secrets: inherit`                                               | explicit input                                     |
| You control the job's other steps | no                                                               | yes                                                |

**Prefer the reusable workflow** unless you need the Action as a step inside a job
you already own. The skip guard mirrors the CLI's classifier, so keeping a private
copy of it in your repo means keeping it in sync by hand — nothing bumps an inline
`if:` expression when the classifier changes.

Unlike a read-only hygiene check, this caller is **not** trigger-free: enabling
auto-merge on a Dependabot PR needs base-context write, so the caller triggers on
`pull_request_target` and grants write scopes.

## Shape A — the reusable workflow (recommended)

```yaml
# .github/workflows/bot-automerge.yml
name: bot-automerge
on:
  pull_request_target:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: write # enable GitHub-native auto-merge on the PR
  pull-requests: write # read PR metadata + turn on auto-merge

jobs:
  bot-automerge:
    uses: rmartz/bot-automerge-action/.github/workflows/bot-automerge-reusable.yml@<sha> # vX.Y.Z
    secrets: inherit
```

That is the whole caller. Notes:

- **Grant both scopes.** A called workflow runs with the _intersection_ of the
  scopes it declares and the scopes you grant, so omitting one fails the run at
  startup validation. The CLI installs from npmjs with no auth, so there's no
  `packages: read`. A pin to a reusable-workflow release from before the move
  still declares it, so keep granting it until Dependabot moves your pin past that.
- **`secrets: inherit`** threads `RELEASE_PLEASE_PAT` through automatically. Nothing
  to pass by hand, and nothing to forget — see
  [the PAT note](#why-release-please-token-matters) below.
- **The skip guard is built in.** A PR the CLI could never trust resolves as
  `skipped` without starting a runner. No `if:` in your repo to keep current.
- **No `pr:` input needed** — it defaults to the caller's own `pull_request` event.
  Pass it only when calling from an event with no PR, such as `workflow_dispatch`.

## Shape B — the composite Action

Use this when you want the step inside a job you already own. You then own the job,
which means you own the skip guard too.

```yaml
# .github/workflows/bot-automerge.yml
name: bot-automerge
on:
  pull_request_target:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: write # enable GitHub-native auto-merge on the PR
  pull-requests: write # read PR metadata + turn on auto-merge

jobs:
  bot-automerge:
    # Skip PRs the CLI could never trust — see "Skip PRs that can never qualify".
    if: >-
      github.event.pull_request.head.repo.full_name == github.repository
      && (github.event.pull_request.user.login == 'dependabot[bot]'
      || startsWith(github.event.pull_request.head.ref, 'release-please--'))
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@<sha> # v7.0.1
      - uses: rmartz/bot-automerge-action@<sha> # vX.Y.Z
        with:
          pr: ${{ github.event.pull_request.number }}
          release-please-token: ${{ secrets.RELEASE_PLEASE_PAT }}
```

Why each piece is there:

- **`pull_request_target`, not `pull_request`.** Dependabot PRs run with a
  read-only token; enabling auto-merge needs base-context write, which
  `pull_request_target` provides. This is expected, not a review flag.
- **Write scopes on the job.** `contents: write` + `pull-requests: write` are what
  `gh pr merge --auto` needs. The `@rmartz/bot-automerge` CLI installs from npmjs
  with no auth, so no `packages: read` and no PAT for the install. (Action versions
  from before the move installed from GitHub Packages and needed `packages: read`.)
- **Check out first.** `uses: ./`-style local actions and this published Action
  both run as a step in your job; a plain `actions/checkout` before the step is
  enough (the Action reads no repo history).
- **`pr`** is required — pass `${{ github.event.pull_request.number }}`.

### Why `release-please-token` matters

- **`release-please-token`** is optional. Set it to a real-actor PAT
  (`${{ secrets.RELEASE_PLEASE_PAT }}`) if you use release-please and want a merged
  release PR to re-trigger your release CD; omit it otherwise (it falls back to the
  job token).

## Skip PRs that can never qualify (Shape B only)

> Shape A has this built in. This section applies only if you own the job.

The bot conditions in the `if:` guard are an optimisation, not a gate — the CLI
already no-ops on anything it does not trust. The first condition, which skips fork
PRs, is a safety check (GHSA-39fm-72q5-676g): a fork picks its own branch name, so
it could otherwise pose as a release-please PR. The Action also rejects fork PRs
itself, but keep the condition anyway so that pins older than the fix stay safe. Without it, every human PR pays for a checkout, a
Node setup and an `npm ci` just to conclude "not a bot PR". With it, those PRs
resolve as `skipped` for free.

The bot conditions mirror the CLI's own classifier exactly, so nothing
eligible is skipped:

| Condition                                  | Why                                                   |
| ------------------------------------------ | ----------------------------------------------------- |
| `head.repo.full_name == github.repository` | Fork PRs are never eligible (required, not optional). |
| `user.login == 'dependabot[bot]'`          | Dependabot is detected by author.                     |
| `startsWith(head.ref, 'release-please--')` | release-please's default branch prefix.               |

The `autorelease: pending` label is **not** a condition. From `@rmartz/bot-automerge`
1.0.1 (GHSA-4f7f-7fcp-gcm6), release-please PRs are identified by branch prefix only.
Anyone with triage access can apply a label, so a label-only match would let them get
auto-merge armed on any same-repo PR. A release-please setup that uses a custom branch
name no longer qualifies.

> Do **not** simplify this to an author test such as
> `endsWith(github.event.pull_request.user.login, '[bot]')`. A release-please PR is
> only authored by a `[bot]` account when release-please runs under the default
> `GITHUB_TOKEN` — run it under a PAT (which is what `release-please-token` is for)
> and the PR is authored by that real user, so an author-only guard would silently
> skip exactly the release PRs you wanted merged.

## Do not add a `concurrency:` group

The caller above deliberately has none, and a burst of `pull_request_target` events
on one PR will start several overlapping runs. That is the intended trade.

GitHub permits only **one pending run per concurrency group** and cancels any run it
supersedes, so a group leaves `cancelled` check-runs on the PR — a conclusion many
merge-gating and triage tools read as a failure, and one indistinguishable from a
genuine timeout. The bursts are routine, not pathological: opening a PR and applying
three labels fires four events in about a second.

The runs a group would have deduplicated are short, idempotent (`gh pr merge --auto`
is a no-op when auto-merge is already on) and, with the skip guard above, usually
`skipped`. If you want to cut them further, trim the trigger list rather than adding
a group — `edited` in particular fires every time Dependabot rewrites a PR body.

## The `auto-merge enabled` label

When the Action arms auto-merge on an eligible PR, it also applies an
**`auto-merge enabled`** label. Triage bots, dashboards, and PR coordinators can read
that label to know bot-automerge already owns the PR and skip it. Add the label to your
repo's label roster (for example with `ai-ensure-labels`). Applying it is best-effort:
if the label is missing, the run logs a warning and still succeeds, because auto-merge
is already armed. It needs no permission beyond the `pull-requests: write` the job
already has.

On the release-please path, the label is applied with `release-please-token` (a real
actor), so it fires one more `labeled` event and one extra run. That run is harmless:
enabling auto-merge again is a no-op.

## Prerequisite — require `merge-safety` + your CI first

> `gh pr merge --auto` merges a PR **immediately** when the repo has no required
> status checks. This Action only makes a bot PR _eligible_ to auto-merge; the
> repo's required checks are what it waits on.

Adopt this Action **only after** the default branch requires
[`merge-safety`](https://github.com/rmartz/merge-safety) and your own CI gates.
Otherwise an eligible bot PR rides straight to `main` with nothing verifying it.
See the
[repository checklist → Automated bot-PR merge](https://github.com/rmartz/ai/blob/main/docs/guidance/repository-checklist.md#automated-bot-pr-merge).

## Keep the pin current

Pin the action by commit SHA with a plain `# vX.Y.Z` comment and run Dependabot's
`github-actions` ecosystem — the same channel every Action consumer uses:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Dependabot opens a PR bumping the SHA + comment to each new release. New eligibility
logic ships inside that release and takes effect with no edit to your caller — see
[the distribution pipeline](design/distribution-pipeline.md).

## Migrating from `@rmartz/bot-automerge`'s reusable workflow

If you currently call `@rmartz/bot-automerge`'s reusable workflow
(`uses: rmartz/bot-automerge/.github/workflows/bot-automerge.yml@<sha>` with
`secrets: inherit`), the smallest migration is to **Shape A**: change the `uses:`
path to this repo's `bot-automerge-reusable.yml` and keep `secrets: inherit`. The
trigger, the write scopes and the eligibility behaviour are unchanged; you gain the
built-in skip guard.

Migrate to **Shape B** instead only if you want to own the job. Then the caller
takes `runs-on` + `steps`, `secrets: inherit` becomes the explicit
`release-please-token` input, and you add the skip guard yourself.

Either way you drop the predecessor's per-PR `concurrency` group, which is what
left `cancelled` check-runs on PRs — see
[the section above](#do-not-add-a-concurrency-group). Neither shape here declares
one.
