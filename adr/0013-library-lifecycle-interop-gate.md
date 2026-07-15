# 0013. Non-patch library releases are gated on interop against the stable frontier

- Status: proposed
- Date: 2026-07-15
- Deciders: mikluko

> Builds on the interop matrix of [0010](0010-conformance-model.md) and the
> immutable vectors of [0012](0012-vector-immutability.md). This governs
> **library lifecycle only**: a release that changes the wire contract (a new
> spec version, or a vector that imposes a new requirement) is a
> [0006](0006-spec-versioning.md) spec-version event and is out of scope here.

## Context

[0010](0010-conformance-model.md) established two always-on conformance layers —
static vectors (per implementation, the wire contract) and the live interop
matrix (`server_lang × client_lang × transport`) — and gates a **new language**
joining the matrix. It says nothing about how an existing implementation's
**version upgrades** flow through that matrix.

The forces:

- Interop is anchored to the wire **spec** version, not library semver
  ([0006](0006-spec-versioning.md)); backward compatibility within a spec version
  is permanent. So a library upgrade that does not touch the spec *should* still
  interoperate — but "should" is precisely what must be proven. A refactor, a
  dependency bump, or a swapped serialization path can move the wire with no spec
  change and no intent to.
- The risk concentrates at **non-patch** releases (minor/major), where new
  surface lands. The worst outcome is a **stable** release that breaks the
  frontier: it is already published, and peers pin to it.
- The matrix pins one version per language, but *which* version, and how a new
  release **replaces** it, is undefined. Left implicit, "latest stable" drifts
  and a regression can enter the frontier unnoticed, with no point at which the
  replacement is reviewed.

Because vectors are immutable and append-only ([0012](0012-vector-immutability.md)),
the gate can treat them as a fixed arbiter of who diverged.

## Decision

Adopt a **beta-gate** for non-patch library releases, layered on top of 0010's
always-on conformance.

- **The stable frontier is an explicit manifest.** The `interop` repo
  ([0010](0010-conformance-model.md), [0007](0007-repository-naming-convention.md))
  carries a manifest pinning **exactly one stable version per language** — the
  frontier. The standing matrix runs `frontier × frontier`. The manifest is
  PR-governed ([0005](0005-adr-process.md)).
- **Non-patch releases are cut as a beta first.** A minor or major release is
  published as a pre-release — per each ecosystem's own convention (a
  `-beta`/`-rc` semver tag, a PyPI pre-release, and so on) — **before** it is
  tagged stable. Patch releases are exempt.
- **The beta is validated against the frontier.** The candidate runs against
  every *stable* peer in the manifest, as **both server and client**, across
  **both transports** (HTTP and gRPC) — the `O(N)` slice of the matrix that
  involves the candidate — behind the vector layer, which gates the matrix
  wire-first (0010). Betas are validated against the stable frontier **only,
  never against another beta**, so the gate is deterministic.
- **Green ⇒ promote; promotion moves the frontier.** When the candidate passes
  the vectors and its matrix slice, it is tagged stable and the manifest is
  updated to pin it — one PR, replacing the old stable. That PR **is** the
  promotion.
- **Moving the frontier re-arms in-flight betas.** A beta validated against a
  now-superseded frontier entry is stale and must re-run its slice against the
  updated manifest before it can promote. Promotion is serialized through the
  manifest, so this ordering is explicit rather than racy.
- **Vectors arbitrate failures.** When a candidate and a stable peer disagree,
  the immutable vectors ([0012](0012-vector-immutability.md)) decide who diverged.
  If the **candidate** diverges, fix it and re-cut the beta. If a **peer**
  diverges (a latent bug the candidate exposed), the peer is non-conformant:
  quarantine that cell, file against the peer, and either hold promotion or
  promote with a documented known-issue — a maintainer call, never an automatic
  pass. The corpus is never edited to turn a red cell green (0012).

Two layers stay **always-on** and are **not** replaced by this gate — they are
what covers patch releases and continuous drift: the per-implementation **vector
runner on every release**, and the standing **`frontier × frontier` matrix on
every merge**. The beta-gate is an *additional* pre-release gate, not the only
interop testing.

## Consequences

- A stable release cannot reach the frontier without first proving interop
  against the whole frontier; the worst outcome — a published stable that breaks
  peers — is designed out.
- The gate is `O(N)` per candidate (`candidate × (N−1) peers × 2 directions × 2
  transports`), not the `O(N²)` full matrix — cheap enough to run per beta. Today
  (Go + Python over HTTP and gRPC) a candidate's slice is a handful of cells.
- Promotion is a reviewable manifest edit with an audit trail
  ([0005](0005-adr-process.md)): the frontier's history *is* the history of what
  "latest stable interop" meant over time.
- Patch releases skip the beta step. This is safe **only** because the always-on
  vector runner and standing matrix still cover them; if either lapses,
  patch-level wire regressions could reach the frontier. Those layers are
  load-bearing, not optional.
- A candidate can be blocked by another language's bug. The escape hatch
  (quarantine + file + maintainer hold/known-issue) bounds this, but
  cross-language coupling is real and grows with the language count.
- Pre-1.0 wrinkle: while libraries are pre-1.0 and the wire is unfrozen
  ([0006](0006-spec-versioning.md)), "non-patch" captures nearly every release,
  so the gate runs often. It loosens naturally once libraries reach 1.0 and the
  spec is frozen.

## Alternatives considered

- **Promote straight to stable, no beta.** Simpler, but the first proof that a
  release interoperates would arrive *after* it is published and pinnable —
  backwards. The beta exists so the proof precedes promotion.
- **Re-run the full `N²` matrix per release.** Maximum coverage, but pays
  `O(N²)` for a change that involves one implementation; the candidate slice is
  where any new divergence would surface, and `frontier × frontier` already runs
  continuously.
- **Validate betas against other betas too.** Tests more combinations, but
  against moving targets — a pass would not be reproducible and promotion
  ordering would be ambiguous. Gating against the stable frontier keeps the gate
  deterministic; beta×beta belongs in exploratory CI, not the release gate.
- **Pin the frontier implicitly** ("just use latest stable", no manifest).
  Rejected: the frontier drifts with no gate and no audit trail, and there is no
  atomic point at which promotion happens or in-flight betas are re-armed.
