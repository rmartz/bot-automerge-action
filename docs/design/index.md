# Design & distribution

How the composite action is put together, and how a new version of the eligibility
logic reaches consumers with no per-repo work.

- [The integration contract](integration-contract.md) — how this action consumes
  `@rmartz/bot-automerge`, the CLI invocation, the two enable paths, and the
  action-path vs API-operation split.
- [The distribution pipeline](distribution-pipeline.md) — the CLI bump →
  bot-automerge → release → consumer Dependabot chain that ships new versions
  automatically.
- [The versioning policy](versioning.md) — why the Action carries its own SemVer
  line independent of the CLI, and how a consumer-facing breaking change is
  propagated as a major release even when it arrives only as a dependency bump.
