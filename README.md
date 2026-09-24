# bot-automerge-action

A composite GitHub Action that enables GitHub-native auto-merge for the
**trustworthy bot pull requests** a repo chooses to trust — Dependabot
`patch`/`minor` bumps and release-please release PRs — using the
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge) CLI. It is
packaged so that:

1. **Updates propagate automatically.** Consuming repos pin this Action by version;
   Dependabot's `github-actions` ecosystem opens PRs to bump that pin on its normal
   schedule.
2. **New eligibility logic is low-friction.** The classification logic ships inside
   the pinned `@rmartz/bot-automerge` CLI; consumers pick up new behavior on the
   next Dependabot bump with no per-repo YAML edits.

The eligibility _logic_ lives in `@rmartz/bot-automerge`. This repo only wraps its
CLI in an Action step, holds the CLI as a pinned dependency, and re-releases itself
whenever Dependabot bumps that pin — so the whole chain from new logic to a
consumer's CI runs itself.

This Action **supersedes** `@rmartz/bot-automerge`'s reusable workflow
(`bot-automerge.yml`); consumers migrate off it, and once the fleet has, that
predecessor is retired. What was retired is the predecessor's copy, not the
reusable-workflow _shape_: this repo offers a thin reusable-workflow wrapper around
the Action as well, so a consumer picks whichever shape suits its repo. Both run the
same pinned CLI and enforce the same eligibility policy — see
[the consumer setup guide](docs/consuming.md),
[the integration contract](docs/design/integration-contract.md) and
[the distribution pipeline](docs/design/distribution-pipeline.md).

## Using it in a consuming repo

Add one caller workflow, in one of two shapes. Both pin by SHA, both are bumped by
Dependabot's `github-actions` ecosystem, and both enforce the same policy — they
differ only in what your repo owns. **Prefer the reusable workflow** unless you need
the Action as a step inside a job you already own. The full comparison and the
migration path are in [the consumer setup guide](docs/consuming.md).

Because enabling auto-merge on a Dependabot PR needs base-context write, either
caller triggers on `pull_request_target` and grants write scopes.

**Shape A — the reusable workflow (recommended).** Three lines, and it carries the
skip guard for non-bot PRs so you have no `if:` expression to keep in sync:

```yaml
# .github/workflows/bot-automerge.yml
name: bot-automerge
on:
  pull_request_target:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: write
  pull-requests: write

jobs:
  bot-automerge:
    uses: rmartz/bot-automerge-action/.github/workflows/bot-automerge-reusable.yml@<sha> # vX.Y.Z
    secrets: inherit
```

**Shape B — the composite Action.** Use this when you want the step inside a job you
own. A composite Action **cannot** declare its own triggers or use
`secrets: inherit`, so this caller owns the job and passes any PAT as an explicit
input — and owns the skip guard too:

```yaml
# .github/workflows/bot-automerge.yml (same on:/permissions: as above)
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

> **Require `merge-safety` + your CI checks on the default branch _before_ adopting
> this.** `gh pr merge --auto` merges a PR immediately if the repo has no required
> status checks — this Action only makes a bot PR _eligible_ to auto-merge; the
> repo's required checks are what it waits on. See the
> [consumer setup guide](docs/consuming.md) for the full prerequisite.

The `@rmartz/bot-automerge` CLI is public on npmjs and installs with no token, so
neither shape needs `packages: read`. One exception: a Shape A caller pinned to a
reusable-workflow release from before the move must keep granting it, because
those releases still declare it and a caller must grant every scope a called
workflow declares. Drop it once Dependabot moves your pin past that.

### Inputs

| Input                  | Default               | Meaning                                                                               |
| ---------------------- | --------------------- | ------------------------------------------------------------------------------------- |
| `pr`                   | _(required)_          | PR number to classify and enable auto-merge for.                                      |
| `token`                | `${{ github.token }}` | GH token for reading the PR and the Dependabot-path enable.                           |
| `release-please-token` | `''` (→ `token`)      | Real-actor PAT for the release-please path so the merge re-triggers release CD.       |
| `update-type`          | `''`                  | Optional Dependabot semver update-type override; empty derives it via fetch-metadata. |
| `node-version`         | `'22'`                | Node.js version the CLI runs under.                                                   |

There is no `version` input — the CLI version is the one pinned in this Action's
lockfile.

## How it relates to `@rmartz/bot-automerge`

This Action supersedes that package's reusable workflow — the predecessor's copy,
not the reusable-workflow shape, which is still offered here as a wrapper around
this Action. It carries **no
check-run** (unlike [`@rmartz/merge-safety`](https://github.com/rmartz/merge-safety))
— it is purely an eligibility enabler. See
[the eligibility contract](https://github.com/rmartz/bot-automerge/blob/main/docs/bot-automerge-contract.md).

## Documentation

Full docs, written in [Open Knowledge Format](docs/okf-format.md), start at
[docs/index.md](docs/index.md).

## Releases

Versioned by [semantic-release](https://semantic-release.gitbook.io/): a merge to
`main` cuts the tag + GitHub Release. It publishes no package and commits nothing
back. A Dependabot `fix(deps)` bump of `@rmartz/bot-automerge` cuts a patch
release, which is how new eligibility logic reaches consumers. A `Release dry-run`
CI job validates the semantic-release config (that the changelog toolchain renders)
on every PR, so a broken release setup is caught before merge rather than on the
post-merge release run.

---

🤖 Created by Claude Opus 4.8
