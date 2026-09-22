---
type: Design
title: The versioning policy
description: Why the Action carries its own SemVer line independent of the @rmartz/bot-automerge CLI, and how a consumer-facing breaking change is propagated as a major Action release even when it reaches this repo only as a dependency bump.
tags: [design, releases, semver, versioning]
---

# The versioning policy

The Action versions **independently** of the
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge) CLI it wraps. The
Action's `vMAJOR.MINOR.PATCH` is its own line, not a mirror of the pinned CLI
version. This page is the authoritative statement of that policy and of how a
breaking change is propagated to consumers — including the load-bearing case where
the breakage reaches this repo only as a **dependency version bump**, never as an
edit to the wrapper.

## Why independent, not mirrored

The Action and the CLI change for different reasons and answer to different
audiences, so one version number cannot honestly serve both:

- **They have separate change streams.** The CLI's version tracks eligibility
  _logic_ (which authors and update-types qualify). The Action's version tracks the
  _wrapper's_ contract — its [inputs](../overview.md#inputs), the CLI invocation
  shape, the composite steps, the two [enable paths](integration-contract.md#the-two-enable-paths).
  A mirrored version has no way to say "the wrapper's input contract broke but the
  logic didn't" — or the reverse.
- **SemVer is a promise to the _consumer_, and the Action's consumer is not the
  CLI's consumer.** A repo pinning `rmartz/bot-automerge-action@vX` cares about the
  Action's input/output/behavior contract; whether that ref bundles CLI `3.x` or
  `4.x` is an implementation detail it should never have to reason about. Mirroring
  leaks the CLI's numbering into a contract it isn't party to.
- **The version _numbers_ do not line up.** A CLI `4.0.0` that only reworks an
  internal API the wrapper never touches breaks nothing for Action consumers;
  mirroring the _number_ would force a spurious `action@v4` that churns every
  consumer's pin for no behavior change. So the Action keeps its own number line.

This is the ordinary GitHub-Actions norm: `setup-node` does not track Node's
version, `setup-python` does not track Python's. The Action version describes the
Action.

Independent _numbering_ is not the same question as which _bump type_ a CLI change
maps to. The Action does mirror the CLI's bump _type_ (next section) as a default,
because for this single-purpose CLI the type usually tracks the Action's own
consumer-facing change — but it stays a default with a human override, precisely
because the two are not guaranteed identical.

## The translation rule

Because the numbers are independent, every CLI change must be **translated** into an
Action bump type. For a CLI dependency bump the Action **mirrors the CLI's own
semver bump type** — a defensible default because this CLI's semver axis _is_ its
eligibility behavior (its whole job), so a CLI minor really is a new capability the
Action gained and a CLI patch really is a fix. The mapping is applied automatically
by the [`dependabot-release-type`](../../.github/workflows/dependabot-release-type.yml)
workflow, which rewrites the `fix(deps):` title Dependabot opens with:

| CLI bump  | Action release | Title mechanism                                                   |
| --------- | -------------- | ----------------------------------------------------------------- |
| **patch** | patch          | `fix(deps):` (left as opened) → semantic-release patch            |
| **minor** | minor          | `feat(deps):` → semantic-release minor                            |
| **major** | major          | `feat(deps)!:` + `breaking change` label → semantic-release major |

Mirroring the bump _type_ keeps the Action's version history honest (a CLI capability
addition shows as an Action minor, not a flattened patch) while the version _numbers_
stay independent. It is a **heuristic, not a law**: the CLI's semver describes the
CLI's own contract, which is not identical to the Action's consumer-facing contract.
The one mismatch that carries real cost — a CLI major for reasons invisible to Action
consumers — is why the major is human-reviewed and downgradable (below). A change to
the wrapper's _own_ surface is versioned separately again; see
[When the wrapper itself changes](#when-the-wrapper-itself-changes).

## Propagating a breaking change through a dependency bump

The subtle, load-bearing case: **a change that is breaking for _Action consumers_
can arrive here as nothing more than a bumped `@rmartz/bot-automerge` version.** No
file in this repo other than `package.json` / `package-lock.json` changes, yet the
observable behavior of `rmartz/bot-automerge-action@vX` shifts. Examples:

- The CLI narrows or widens which authors/update-types are eligible, so a PR class a
  consumer relied on auto-merging (or _not_ auto-merging) flips.
- The CLI changes its exit-code semantics or the `gh pr merge` mode it invokes.
- The CLI adds a required argument the wrapper must now pass, changing the
  effective input contract.

When a CLI bump carries any such consumer-observable break, the Action release **must
be a major**, so that a consumer pinning by major (the normal Dependabot
`github-actions` shape) sees it flagged rather than absorbed as a routine patch. The
breakage must propagate on the _Action's_ terms even though it reached us only as a
dependency bump.

### Where the trust boundary sits

Two bump paths reach this repo, and the safety of each rests on a clear assumption:

1. **CLI patch/minor bumps are auto-merged** by this repo dogfooding itself (the
   `production-dependencies` group in [`dependabot.yml`](../../.github/dependabot.yml)
   is `patch`/`minor` only), with no human in the loop — cutting an Action **patch**
   (`fix(deps):`) or **minor** (`feat(deps):`) respectively. This is safe **only
   because the CLI honors SemVer**: a CLI patch/minor is promised non-breaking to
   _its_ consumers, and the wrapper is one of them. That promise is the load-bearing
   assumption of the auto-merge path. If a CLI patch/minor is ever found to have
   shipped a consumer-facing break (a CLI SemVer defect), the fix is to cut a
   corrective **major** Action release immediately (a `feat!:` follow-up) and to get
   the CLI re-versioned — do not let the silent patch/minor stand.

2. **A CLI major bump falls out of the auto-merge group into its own PR** for a human
   to review — and a CLI major is the single strongest signal of consumer-facing
   breakage. The **default is therefore to propagate it as an Action major**, and
   that default is applied automatically: the
   [`dependabot-release-type`](../../.github/workflows/dependabot-release-type.yml)
   workflow detects a major bump of the wrapped CLI (via `dependabot/fetch-metadata`,
   gated to the npm ecosystem and the exact package name), adds the `breaking change`
   label, and rewrites the title `fix(deps):` → `feat(deps)!:` so the squash-merge
   cuts a major. The reviewer's job is to **confirm or downgrade**: if they positively
   confirm the CLI's breaking change is invisible to Action consumers (e.g. it
   touched only an internal CLI API the wrapper does not exercise), they remove the
   `!` and the label before merging and record that reasoning on the PR — otherwise
   the flagged major stands. The workflow fires only on `opened`/`reopened`, so a
   later Dependabot rebase does not re-apply the marker over a deliberate downgrade.

   > This is the reverse of a "default to patch, opt into major" stance. Defaulting a
   > reviewed CLI-major to a patch would let exactly the breakage this policy exists
   > to catch reach consumers unflagged. The safe default is to flag; the burden of
   > proof is on _not_ flagging — which is why the flag is applied by default and
   > removed only by a deliberate human decision.

## When the wrapper itself changes

A change to this repo's own surface — an [input](../overview.md#inputs) added,
renamed, defaulted differently, or removed; the enable-path behavior; the required
consumer permissions or caller shape — is versioned on its own merits by its
Conventional-Commits PR title, independent of any CLI bump: additive → `feat:`
(minor), breaking → `feat!:` / `fix!:` (major), non-behavioral → `docs:` / `chore:` /
`ci:` (no release). The [self-dogfood caller](distribution-pipeline.md#self-dogfooding)
and this repo's squash-merge-by-PR-title setup mean the PR title _is_ the release
input.

## Traceability — record the pinned CLI in the release

Because the numbers diverge, a consumer cannot read the Action version to learn which
CLI it bundles. Preserve that link where it belongs: the pinned CLI version travels
in the `fix(deps): bump @rmartz/bot-automerge …` commit that cuts the release, so
semantic-release's generated notes name it. Keep the dependency bump as its own
release-cutting commit (rather than folding it into unrelated work) so every Action
release's notes state the CLI version inside it.
