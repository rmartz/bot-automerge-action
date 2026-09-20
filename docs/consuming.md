---
type: Guidance
title: Using bot-automerge-action in a consuming repo
description: The caller job to add, the pull_request_target trigger, write scopes and tokens it requires, the merge-safety prerequisite, and how Dependabot keeps the pin current.
tags: [consumer, setup, ci, auto-merge]
---

# Using bot-automerge-action in a consuming repo

Add one caller workflow. Unlike a read-only hygiene check, this caller is **not**
trigger-free: enabling auto-merge on a Dependabot PR needs base-context write, so
the caller triggers on `pull_request_target`, grants write scopes, and passes any
PAT as an **explicit input** — a composite Action cannot declare its own triggers or
use `secrets: inherit`.

```yaml
# .github/workflows/bot-automerge.yml
name: bot-automerge
on:
  pull_request_target:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: write # enable GitHub-native auto-merge on the PR
  pull-requests: write # read PR metadata + turn on auto-merge
  packages: read # install the public @rmartz/bot-automerge package

jobs:
  bot-automerge:
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
  `gh pr merge --auto` needs; `packages: read` lets the built-in `GITHUB_TOKEN`
  install the public `@rmartz/bot-automerge` package — no PAT for the install.
- **Check out first.** `uses: ./`-style local actions and this published Action
  both run as a step in your job; a plain `actions/checkout` before the step is
  enough (the Action reads no repo history).
- **`pr`** is required — pass `${{ github.event.pull_request.number }}`.
- **`release-please-token`** is optional. Set it to a real-actor PAT
  (`${{ secrets.RELEASE_PLEASE_PAT }}`) if you use release-please and want a merged
  release PR to re-trigger your release CD; omit it otherwise (it falls back to the
  job token).

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

## Migrating from the reusable workflow

If you currently call `@rmartz/bot-automerge`'s reusable workflow
(`uses: rmartz/bot-automerge/.github/workflows/bot-automerge.yml@<sha>` with
`secrets: inherit`), replace that job with the step-based caller above. The two key
differences: the caller now owns `runs-on` + `steps` (with a checkout), and
`secrets: inherit` becomes the explicit `release-please-token` input. Everything
else — the trigger, the write scopes, the eligibility behavior — is unchanged.
