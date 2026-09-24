---
type: Design
title: The integration contract
description: How bot-automerge-action consumes @rmartz/bot-automerge — the pinned npm dependency, the CLI invocation, the two enable paths, and the action-path vs API-operation split.
tags: [design, integration, cli]
---

# The integration contract

The eligibility logic lives in
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge); this repo is only
the Action wrapper. The contract between them is deliberately narrow so each side can
evolve independently.

## The package and CLI

- **Package:** `@rmartz/bot-automerge`, published publicly to npmjs
  (`https://registry.npmjs.org/`, with provenance). Installs with no auth. Versions
  up to 0.2.1 were also published to GitHub Packages, which older Action releases
  installed from.
- **CLI (bin):** `ai-bot-automerge`. The action invokes it as
  `ai-bot-automerge enable --pr <n> --repo <owner/repo> [--update-type <t>]`. The CLI
  classifies the PR and, when it is eligible, runs `gh pr merge --auto --squash`. Its
  eligibility rules and exit-code semantics are the
  [eligibility contract](https://github.com/rmartz/bot-automerge/blob/main/docs/bot-automerge-contract.md).

## Version consumption — a pinned dependency, not an install string

The action holds `@rmartz/bot-automerge` as a **pinned `package.json` dependency**
(exact `major.minor.patch`) with a committed `package-lock.json`, not as an
`npm install -g @rmartz/bot-automerge@<literal>` string in a run step. This is the
crux of the design:

- A literal install string is invisible to Dependabot, which cannot bump a version
  buried in shell. A lockfile dependency **is** on Dependabot's npm channel.
- So the CLI version is bumped by Dependabot → auto-merged by bot-automerge → cut as
  a new Action release. See [the distribution pipeline](distribution-pipeline.md).
- The pinned version is therefore the single source of truth for "which eligibility
  logic this Action ref enforces," which is why the Action has no `version` input.

The repo's [`.npmrc`](../../.npmrc) pins the `@rmartz` scope to npmjs. That keeps
a runner- or user-level `.npmrc` that maps `@rmartz` to GitHub Packages (other
`@rmartz` packages still live there) from redirecting the install.

## Fork PRs are rejected first

Before anything else, the action's `Reject fork PRs` step reads the PR's
`isCrossRepository` and `headRepository` from `gh pr view`. Every later step runs
only if the PR's head branch lives in the base repository. A fork picks its own
branch name, so it could otherwise pose as a release-please PR while the caller's
`pull_request_target` job holds a write token (GHSA-39fm-72q5-676g). A missing
field or a deleted head repository counts as a fork. If the PR can't be read at
all, the step fails. The step doesn't depend on the pinned CLI version, which
rejects fork PRs too from `@rmartz/bot-automerge` 0.2.1 onward. The reusable
workflow and this repo's own caller also skip fork PR events at the job level.

## The two enable paths

The action reproduces the reusable workflow's branch on PR author, because the two
paths differ in both metadata and token:

1. **Dependabot** (`github.event.pull_request.user.login == 'dependabot[bot]'`) —
   runs `dependabot/fetch-metadata` (a nested `uses:` step) to obtain the semver
   update-type, unless the `update-type` input overrides it, and passes it as
   `--update-type` so only patch/minor bumps qualify. Enables auto-merge with the
   `token` input's GH token.
2. **release-please / other bot** (any other author) — passes no update-type; the CLI
   detects release-please itself (branch/label markers) and no-ops on anything it
   does not trust. Enables auto-merge with `release-please-token` when set (falling
   back to `token`), so a real-actor PAT can re-trigger the consumer's release CD.

Both guards key off the PR **author**, which is robust across
opened/synchronize/labeled events — unlike `github.actor`, which can be a human who
relabeled the PR.

## Why `release-please-token` is an explicit input

A composite Action cannot use `secrets: inherit` (a reusable-workflow-only feature),
and secrets do not flow implicitly into a composite Action. The reusable workflow
reached `RELEASE_PLEASE_PAT` via `secrets: inherit`; here the consumer passes it as
the `release-please-token` input. This is the one behavioral difference a migrating
consumer must make — see [consuming.md](../consuming.md).

## The action-path vs API-operation split

A composite action runs in the **consumer's** checkout, but its own
`package.json` / lockfile live wherever GitHub places the action
(`$GITHUB_ACTION_PATH`). So the action:

1. runs `npm ci` with `working-directory: ${{ github.action_path }}` — installing the
   pinned CLI into the action's own `node_modules`, not the consumer's tree; and
2. invokes the CLI by absolute path
   (`${GITHUB_ACTION_PATH}/node_modules/.bin/ai-bot-automerge`).

Unlike a hygiene checker, the CLI does **not** scan the consumer workspace — it
operates on the PR through the GitHub API using the GH token in its environment. The
consumer still runs `actions/checkout` before the step (the standard shape for a
step-based action), but the action reads no repo history itself.
