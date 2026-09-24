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

| Input                  | Default               | Meaning                                                                                     |
| ---------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `pr`                   | _(required)_          | PR number to classify and enable auto-merge for.                                            |
| `token`                | `${{ github.token }}` | GH token for reading the PR and the Dependabot enable path.                                 |
| `release-please-token` | `''` (→ `token`)      | Real-actor PAT for the release-please / other-bot path so the merge re-triggers release CD. |
| `update-type`          | `''`                  | Optional Dependabot semver update-type override; empty derives it via `fetch-metadata`.     |
| `node-version`         | `'22'`                | Node.js version the CLI runs under.                                                         |

There is deliberately **no `version` input** (unlike the reusable workflow): the
installed CLI version is the one pinned in this Action's lockfile, bumped by
Dependabot and shipped as a new Action release. See
[the distribution pipeline](design/distribution-pipeline.md).

## Why an explicit `release-please-token`

The reusable workflow used `secrets: inherit` to reach `RELEASE_PLEASE_PAT`. A
composite Action **cannot** use `secrets: inherit` — that is a reusable-workflow-only
feature and secrets do not flow implicitly into a composite Action. So the PAT is
passed as an explicit input. When set, the release-please / other-bot path enables
auto-merge as that real actor, which lets the merged release PR re-trigger the
consumer's release CD (a `GITHUB_TOKEN`-attributed merge does not fire further
workflows — [bot-automerge#8](https://github.com/rmartz/bot-automerge/issues/8)). It
falls back to `token` when unset, in which case auto-merge still works but a release
PR's downstream CD will not re-fire.
