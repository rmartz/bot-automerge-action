---
type: Design
title: The distribution pipeline
description: The automatic chain that ships new eligibility logic to consumers — CLI bump, bot-automerge, semantic-release tag, and the consumer's own Dependabot pick-up.
tags: [design, releases, dependabot, automerge]
---

# The distribution pipeline

New versions of the eligibility logic reach consumers with no manual step at any
hop. The chain has two halves: producing a new Action release here, and consumers
picking it up.

## Producing a release (this repo)

1. **CLI bump.** `@rmartz/bot-automerge` publishes a new version to GitHub Packages.
   Dependabot's npm ecosystem (with the `github-packages` registry auth wired in
   [`dependabot.yml`](../../.github/dependabot.yml)) opens a PR bumping the pinned
   dependency + lockfile, titled `chore(deps): bump @rmartz/bot-automerge …`.
2. **Auto-merge.** This repo dogfoods bot-automerge on itself
   ([`bot-automerge.yml`](../../.github/workflows/bot-automerge.yml)): the caller
   classifies the bump as a trusted Dependabot patch/minor and enables native
   auto-merge. It lands once the required checks pass — CI plus the
   [`merge-safety`](https://github.com/rmartz/merge-safety) verdict — so nothing
   merges ahead of green.
3. **Release.** On merge to `main`,
   [`release.yml`](../../.github/workflows/release.yml) runs semantic-release.
   [`.releaserc.json`](../../.releaserc.json) maps `chore(deps)` → **patch**, so the
   CLI bump cuts a new tag + GitHub Release. The release publishes nothing to a
   registry and commits nothing back (no `@semantic-release/npm`, no
   `@semantic-release/git`), so the built-in `GITHUB_TOKEN` suffices — no PAT.

A **major** CLI bump falls out of the auto-merge set into its own PR for a human to
review; merging it still cuts a patch Action release (auto-classification cannot
infer consumer-facing breakage), so a reviewer who judges the change breaking retitles
the PR `feat!:` to cut a major.

## Picking it up (consumers)

4. **Consumer Dependabot.** Each consumer pins this Action by SHA
   (`uses: rmartz/bot-automerge-action@<sha> # vX.Y.Z`) and runs Dependabot's
   `github-actions` ecosystem, which opens a PR bumping that pin to the new release.
5. **New logic takes effect.** The updated eligibility logic ships inside the CLI
   version this release pins, so it takes effect the moment the consumer merges the
   bump — no edit to their caller.

## Bootstrap — until the first release exists

This repo cannot dogfood its **own** action until it has cut a release, so during the
bootstrap phase [`bot-automerge.yml`](../../.github/workflows/bot-automerge.yml)
calls `@rmartz/bot-automerge`'s reusable workflow (`@<sha> # v0.1.1`) instead of
`uses: ./`. After the first `bot-automerge-action` release, that caller flips to the
local action — the exact shape every consumer uses. The flip instructions live inline
in the workflow file.

## Why an Action, not a reusable workflow

The predecessor shipped as a reusable workflow (`bot-automerge.yml`) that consumers
called at the job level with `secrets: inherit`. A composite Action lets the consumer
own the job — its triggers, checkout, and permissions — and drop the step into an
existing `pull_request_target` job. The one cost of the switch is that a composite
Action cannot use `secrets: inherit`, so the release-please PAT becomes an explicit
input ([integration-contract.md](integration-contract.md)). The fleet cutover from
the reusable workflow to this Action is a coordinated migration; once every consumer
has migrated, the reusable workflow in `@rmartz/bot-automerge` is retired.
