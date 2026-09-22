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
   dependency + lockfile, titled `fix(deps): bump @rmartz/bot-automerge …` (the
   npm ecosystem uses `commit-message.prefix: fix` for production deps).
2. **Map the release type.** The
   [`dependabot-release-type`](../../.github/workflows/dependabot-release-type.yml)
   workflow rewrites that title to mirror the CLI's semver bump into the Action's
   release type — patch stays `fix(deps):`, minor becomes `feat(deps):`, major
   becomes `feat(deps)!:` + a `breaking change` label. See the
   [versioning policy](versioning.md).
3. **Auto-merge.** This repo dogfoods bot-automerge on itself
   ([`bot-automerge.yml`](../../.github/workflows/bot-automerge.yml)): the caller
   classifies the bump as a trusted Dependabot patch/minor and enables native
   auto-merge. It lands once the required checks pass — CI plus the
   [`merge-safety`](https://github.com/rmartz/merge-safety) verdict — so nothing
   merges ahead of green.
4. **Release.** On merge to `main`,
   [`release.yml`](../../.github/workflows/release.yml) runs semantic-release.
   [`.releaserc.json`](../../.releaserc.json) uses the conventionalcommits preset,
   which maps `fix` → **patch** and `feat` → **minor** (a `!` marker → **major**), so
   the CLI bump cuts a new tag + GitHub Release at the mirrored level.
   The release publishes nothing to a
   registry and commits nothing back (no `@semantic-release/npm`, no
   `@semantic-release/git`), so the built-in `GITHUB_TOKEN` suffices — no PAT.
   Because `release.yml` only runs post-merge, a broken release toolchain would
   otherwise surface only after an auto-merged bump lands and silently stall this
   chain. The `Release dry-run` job in [`ci.yml`](../../.github/workflows/ci.yml)
   guards against that by rendering the release notes on every PR via a
   `--dry-run` semantic-release invocation, so a toolchain regression fails the PR
   instead.

A **major** CLI bump falls out of the auto-merge set into its own PR for a human to
review (the `production-dependencies` Dependabot group is patch/minor only). A CLI
major is the strongest signal of consumer-facing breakage, so the `feat(deps)!:` +
`breaking change` mapping the `dependabot-release-type` workflow applies on open is
the **default**, not the last word: the reviewer downgrades it (removes the `!` and
label) only when they confirm the break is invisible to Action consumers. The full
rule, including how a breaking change is propagated even when it reaches this repo
only as a dependency bump, is the [versioning policy](versioning.md).

## Picking it up (consumers)

5. **Consumer Dependabot.** Each consumer pins this Action by SHA
   (`uses: rmartz/bot-automerge-action@<sha> # vX.Y.Z`) and runs Dependabot's
   `github-actions` ecosystem, which opens a PR bumping that pin to the new release.
6. **New logic takes effect.** The updated eligibility logic ships inside the CLI
   version this release pins, so it takes effect the moment the consumer merges the
   bump — no edit to their caller.

## Self-dogfooding

This repo dogfoods its **own** action:
[`bot-automerge.yml`](../../.github/workflows/bot-automerge.yml) calls the local
action (`uses: ./`) on this repo's own bot PRs — the exact shape every consumer
uses, so the wrapper is exercised end to end on every Dependabot bump. Before the
first release existed it bootstrapped off `@rmartz/bot-automerge`'s reusable workflow
(`@<sha> # v0.1.1`); that cutover is done.

## Why an Action, not a reusable workflow

The predecessor shipped as a reusable workflow (`bot-automerge.yml`) that consumers
called at the job level with `secrets: inherit`. A composite Action lets the consumer
own the job — its triggers, checkout, and permissions — and drop the step into an
existing `pull_request_target` job. The one cost of the switch is that a composite
Action cannot use `secrets: inherit`, so the release-please PAT becomes an explicit
input ([integration-contract.md](integration-contract.md)). The fleet cutover from
the reusable workflow to this Action is a coordinated migration; once every consumer
has migrated, the reusable workflow in `@rmartz/bot-automerge` is retired.
