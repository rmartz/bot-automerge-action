---
type: Reference
title: What bot-automerge-action is
description: The composite Action that wraps the @rmartz/bot-automerge CLI to enable native auto-merge for trusted bot PRs, its inputs, and why a version-pinned Action succeeds the reusable workflow.
tags: [action, overview, ci, auto-merge]
---

# What bot-automerge-action is

`bot-automerge-action` is a **composite GitHub Action** that enables GitHub-native
auto-merge for the **trustworthy bot pull requests** a repo chooses to trust —
Dependabot `patch`/`minor` bumps and release-please release PRs — using the
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge) CLI. A consumer
references it as a single step inside a `pull_request_target` job:

```yaml
- uses: rmartz/bot-automerge-action@<sha> # vX.Y.Z
  with:
    pr: ${{ github.event.pull_request.number }}
```

It is the Action-shaped **successor** to `@rmartz/bot-automerge`'s reusable workflow
(`bot-automerge.yml`). The eligibility _logic_ still lives in `@rmartz/bot-automerge`;
this repo only wraps its CLI in a step consumers can drop into their own job. See
[the integration contract](design/integration-contract.md).

Unlike [`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety), it posts
**no check-run** and carries no fleet check-run contract — it is purely an
eligibility _enabler_. The eligibility rules (which authors and update-types
qualify) are the CLI's; see the
[eligibility contract](https://github.com/rmartz/bot-automerge/blob/main/docs/bot-automerge-contract.md).

## What it does at run time

1. Sets up Node.js.
2. Runs `npm ci` **in the action's own directory** to install the exact
   `@rmartz/bot-automerge` version pinned in this repo's `package-lock.json`, from
   npmjs with no auth.
3. On a Dependabot PR, runs `dependabot/fetch-metadata` (unless an `update-type`
   override is supplied) to obtain the semver update-type.
4. Invokes `ai-bot-automerge enable --pr <n> --repo <owner/repo> [--update-type <t>]`,
   which classifies the PR and, when it is eligible, runs `gh pr merge --auto --squash`
   to turn on native auto-merge.

The action runs **no checkout of the consumer tree** — it operates on the PR through
the GitHub API using the supplied token. Native auto-merge still waits on the repo's
own required status checks (including `merge-safety`, where adopted); the action only
makes the PR _eligible_ to merge itself once those pass.

## Inputs

| Input                  | Default               | Meaning                                                                                                                 |
| ---------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `pr`                   | _(required)_          | PR number to classify and enable auto-merge for.                                                                        |
| `token`                | `${{ github.token }}` | Token for every step; pass a real-actor PAT so the merge fires your push workflows. Empty falls back to `github.token`. |
| `release-please-token` | `''` (→ `token`)      | **Deprecated** — use `token`. When set, still overrides `token` on the release-please / other-bot path, with a warning. |
| `update-type`          | `''`                  | Optional Dependabot semver update-type override; empty derives it via `fetch-metadata`.                                 |
| `node-version`         | `'22'`                | Node.js version the CLI runs under.                                                                                     |

There is deliberately **no `version` input** (unlike the reusable workflow): the
installed CLI version is the one pinned in this Action's lockfile, bumped by
Dependabot and shipped as a new Action release. See
[the distribution pipeline](design/distribution-pipeline.md).

## Why the PAT is an explicit `token`

A composite Action **cannot** use `secrets: inherit` — that is a
reusable-workflow-only feature and secrets do not flow implicitly into a composite
Action. So a real-actor PAT is passed as the explicit `token` input, which every
step uses. GitHub attributes an auto-merge to whoever enabled it and runs no
workflows for events caused by `GITHUB_TOKEN`, so a merge enabled with the default
token fires none of the consumer's push workflows — no CI on `main`, no release CD
([bot-automerge#8](https://github.com/rmartz/bot-automerge/issues/8)). That applies
to Dependabot merges and release-please merges alike, which is why one input now
governs both paths. An empty `token` (an unset secret) falls back to
`github.token`, in which case auto-merge still works but those push workflows do
not fire.

`release-please-token`, which once carried the PAT for the release-please path
only, is deprecated: when set it still overrides `token` on that path and logs a
warning, and it will be removed in the next major.
