# 0005. ADRs are MADR-format and governed by pull request

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

Cross-cutting, org-level decisions (module path, naming, hosting, tooling, spec
versioning) need a durable, reviewable record that spans every repo in the
`valiss-dev` org. The Uptime upcodex documentation system is a separate concern
and is not used here.

## Decision

- Decisions are recorded as **Architecture Decision Records** in lightweight
  **MADR** Markdown format: `NNNN-title.md` with Status, Date, Deciders,
  Context, Decision, Consequences, and Alternatives considered. `0000` is the
  template.
- ADRs are **governed by pull request**: an ADR is proposed as `status: proposed`
  in a PR, becomes `accepted` on merge (flip the status in the same PR before
  merging), and is rejected by closing the PR. Superseding is a new PR that sets
  the old ADR to `superseded by NNNN` and adds the replacement. Direct pushes to
  the default branch are disallowed.
- Numbers are assigned at authoring; collisions between open PRs are resolved by
  renumbering the later one before merge.
- Approval count stays at 0 while the org has a single maintainer (GitHub blocks
  self-approval, so requiring a review would deadlock); it rises to 1 once a
  second maintainer exists.

## Consequences

- Every decision is discussable before it lands and auditable afterward.
- Branch protection enforcing PR-only merges is itself infrastructure and is
  codified in `infra` (see [0004](0004-infrastructure-tooling.md)).
- ADRs currently live in a temporary `./adr` directory in the org workspace.
  Their permanent home is the docs/book repo, to be named later; this ADR will
  be updated when they move.

## Alternatives considered

- Committing decisions directly to a docs page without review — no gate, no
  discussion trail.
- The upcodex ADR workflow — belongs to the Uptime documentation system, not a
  standalone OSS org.
