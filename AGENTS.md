# Agent guide — bot-automerge-action

This repo is the **composite GitHub Action** that enables GitHub-native auto-merge
for trustworthy bot PRs in a consuming repo, using the
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge) CLI. It holds the
CLI as a pinned `package.json` dependency, wraps it in [`action.yml`](action.yml),
and re-releases itself via semantic-release whenever Dependabot bumps that pin — the
chain that ships new eligibility logic to the fleet. It is the **successor** to
`@rmartz/bot-automerge`'s reusable workflow. It **dogfoods bot-automerge on itself**:
`.github/workflows/bot-automerge.yml` auto-merges this repo's own trusted bot PRs.
See [README.md](README.md) and the [documentation](docs/index.md).

## Documentation — read it first, maintain it every task

The `docs/` bundle is a first-class part of this repo, not an afterthought. On
**every** task:

- **Read first.** Before changing `action.yml`, a workflow, or a config, read the
  relevant [`docs/`](docs/index.md) page(s) and this file, so your change stays
  consistent with what is already documented.
- **Extend, correct, and remove in the same PR.** If your change adds, alters, or
  contradicts anything a doc says — an input, the CLI invocation, the release
  cadence, a consumer step — fix that doc in the same PR. If a doc describes
  something that no longer exists, delete it. An outdated doc is worse than none.
- **Close gaps you find.** If you notice an undocumented behavior or a stale page
  while doing something else, fix it (or, if truly out of scope, note it) — do not
  leave known-wrong or missing documentation in place.
- **Docs follow OKF.** Pages under `docs/` use Open Knowledge Format frontmatter
  (`type` / `title` / `description` required) and stay reachable from
  [`docs/index.md`](docs/index.md) under the nested-index rule — an index links only
  same-directory files and a direct child directory's `index.md`. The `okf`,
  `okf-index`, and `docs-links` checks enforce this in CI (the Repo Hygiene job).
  See [docs/okf-format.md](docs/okf-format.md).

## Repository conformance

This repo is held to the shared
[repository checklist](https://github.com/rmartz/ai/blob/main/docs/guidance/repository-checklist.md)
and **self-manages** its own config: fix conformance gaps directly here, in a PR.
Bootstrap (`ai-ensure-*`) is a one-time new-repo starter, not an ongoing manager —
do not defer a fix to a bootstrap re-run, and do not treat a `.github/` file as
off-limits because bootstrap once seeded it.

- **Updates arrive the self-updating way:** the `repo-hygiene`, `merge-safety`, and
  `bot-automerge` callers and (once released) this Action's own pin in consumers are
  bumped by Dependabot; CI (incl. PR-title lint + the `commit-convention` tripwire),
  labels, the hardened `dependabot.yml`, and the squash-merge setting are owned here.
- `ai-ensure-labels` / `ai-verify-squash-setting` are useful one-shot helpers, but
  this repo owns its `.github/` config going forward.

## Common commands

```bash
npm ci                 # install deps (needs GitHub Packages auth for @rmartz/*)
npm run format:check   # prettier --check .
npm run format         # prettier --write .
```

There is no build/test suite — the eligibility logic lives in `@rmartz/bot-automerge`.
This repo's real test is running the local action against a live bot PR; the
`bot-automerge.yml` caller dogfoods it on this repo's own Dependabot PRs.

## Releases

Automated via **semantic-release** ([`.releaserc.json`](.releaserc.json)): a merge
to `main` cuts the git tag + GitHub Release. It publishes nothing and commits
nothing back (no `@semantic-release/npm`, no `@semantic-release/git`), so the
built-in `GITHUB_TOKEN` suffices — no PAT. Production-dependency bumps are titled
`fix(deps)` (→ patch) so a Dependabot bump of `@rmartz/bot-automerge` ships a new
Action version; dev-dependency and github-actions bumps stay `chore` and cut no
release. PR titles
are Conventional Commits and the repo squash-merges using the PR title, so a
non-conventional title makes semantic-release skip the release.

The Action versions **independently** of the wrapped CLI (its SemVer describes the
_wrapper's_ contract, not the CLI's number), but a CLI dependency bump **mirrors the
CLI's semver bump type** into the Action's release type, applied automatically by the
[`dependabot-release-type`](.github/workflows/dependabot-release-type.yml) workflow:
patch → `fix(deps):`, minor → `feat(deps):`, major → `feat(deps)!:` + `breaking
change` label. Patch/minor are auto-merged (safe because the CLI honors SemVer); a
**CLI major** falls out to a human-reviewed PR, defaults to a breaking Action major,
and is downgraded only if the break is confirmed invisible to Action consumers. This
is how a consumer-facing break propagates even when it reaches this repo only as a
dependency bump. Full rule: [docs/design/versioning.md](docs/design/versioning.md).

## Self-dogfooding

This repo dogfoods its **own** action: `.github/workflows/bot-automerge.yml` calls
the local action (`uses: ./`) on this repo's own bot PRs — the exact shape every
consumer uses. (Before the first release existed it bootstrapped off
`@rmartz/bot-automerge`'s reusable workflow; that cutover is done.)

## Worktrees & PRs

- **Work in a dedicated worktree** under `.git-worktrees/` (`ai-new-worktree`),
  never on `main` in the root checkout. Run `npm ci` in a fresh worktree.
- **PR titles must be Conventional Commits** (`feat:`, `fix:`, `docs:`, `chore:`,
  `ci:`, …). Render PR/issue numbers as full Markdown links in chat and agent
  output (e.g. `[#12](https://github.com/rmartz/bot-automerge-action/pull/12)`),
  never a bare `#12`.

## Agent directive files

- **`AGENTS.md` is the single source of truth** for a directory's agent
  instructions — author directives here, never in `CLAUDE.md`.
- **Every `AGENTS.md` has a companion `CLAUDE.md`** in the same directory (a bare
  wrapper whose only content is `@AGENTS.md`), enforced by the `md-pairing` check.
  These live at the repo root, outside `docs/`, so they are not pulled into the OKF
  bundle.
