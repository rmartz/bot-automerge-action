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
- **A CLI major is not automatically an Action major** (and vice versa). A CLI
  `4.0.0` that only reworks an internal API the wrapper never touches breaks nothing
  for Action consumers; mirroring would force a spurious `action@v4` that churns
  every consumer's pin for no behavior change. Conversely a CLI _patch_ that changes
  the meaning of an eligibility decision consumers observe is an Action _major_. The
  numbers simply do not line up.

This is the ordinary GitHub-Actions norm: `setup-node` does not track Node's
version, `setup-python` does not track Python's. The Action version describes the
Action.

## The translation rule

Because the numbers are independent, every CLI change must be **translated** into an
Action bump type — decided by the Action's _own_ observable contract, never by
copying the CLI's number:

| Change here                                                                          | Action bump | Mechanism                                                  |
| ------------------------------------------------------------------------------------ | ----------- | ---------------------------------------------------------- |
| CLI bump, wrapper contract unchanged (new logic, same inputs/outputs/behavior class) | **patch**   | `fix(deps):` (Dependabot default) → semantic-release patch |
| New optional input or additive wrapper capability                                    | **minor**   | `feat:`                                                    |
| **Consumer-facing breaking change** (see below)                                      | **major**   | `feat!:` / `fix!:` (a `!` marker) → semantic-release major |

The default — a CLI bump titled `fix(deps)` cutting a **patch** — is correct for the
common case: new eligibility logic behind an unchanged wrapper contract. The
[distribution pipeline](distribution-pipeline.md) auto-merges those. What follows is
the exception that must _not_ ride that default silently.

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
   is `patch`/`minor` only), titled `fix(deps)`, and cut a **patch** Action release
   with no human in the loop. This is safe **only because the CLI honors SemVer**: a
   CLI patch/minor is promised non-breaking to _its_ consumers, and the wrapper is
   one of them. That promise is the load-bearing assumption of the auto-merge path.
   If a CLI patch/minor is ever found to have shipped a consumer-facing break (a CLI
   SemVer defect), the fix is to cut a corrective **major** Action release
   immediately (a `fix!:` follow-up) and to get the CLI re-versioned — do not let the
   silent patch stand.

2. **A CLI major bump falls out of the auto-merge group into its own PR** for a human
   to review — and a CLI major is the single strongest signal of consumer-facing
   breakage. The **default expectation is therefore to propagate it as an Action
   major**: retitle the Dependabot PR with a breaking marker (`fix(deps)!:` or
   `feat!:`) before merging, so semantic-release cuts a major. Downgrade to a
   patch/minor **only** when the reviewer positively confirms the CLI's breaking
   change is invisible to Action consumers (e.g. it touched only an internal CLI API
   the wrapper does not exercise), and records that reasoning on the PR.

   > This is the reverse of a "default to patch, opt into major" stance. Defaulting a
   > reviewed CLI-major to a patch would let exactly the breakage this policy exists
   > to catch reach consumers unflagged. The safe default is to flag; the burden of
   > proof is on _not_ flagging.

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
