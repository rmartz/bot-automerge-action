---
okf_version: 0.2
---

# Documentation

Documentation for `bot-automerge-action`, the composite GitHub Action that enables
GitHub-native auto-merge for trustworthy bot PRs in a consuming repo, using the
[`@rmartz/bot-automerge`](https://github.com/rmartz/bot-automerge) CLI. Written in
[Open Knowledge Format](okf-format.md).

- [What bot-automerge-action is](overview.md) — what the action does, its inputs,
  and how it differs from the reusable workflow it succeeds.
- [Using it in a consuming repo](consuming.md) — the caller job to add, the
  permissions and tokens it needs, the merge-safety prerequisite, and how the pin
  stays current.
- [The OKF documentation format](okf-format.md) — how these pages are structured
  and validated in this repo.
- [Design & distribution](design/index.md) — how the action wraps the CLI and how
  new versions reach consumers automatically.
