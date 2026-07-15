# 0012. Conformance vectors are immutable and append-only

- Status: proposed
- Date: 2026-07-15
- Deciders: mikluko

> Makes explicit the vector lifecycle that [0010](0010-conformance-model.md)
> leaves partly implicit ("frozen per spec version"). It neither supersedes nor
> edits 0010; it states the mutation rules the conformance corpus was always
> meant to have. The library-lifecycle gate
> [0013](0013-library-lifecycle-interop-gate.md) relies on this immutability.

## Context

[0010](0010-conformance-model.md) makes the static vectors normative: a corpus
of artifacts paired with expected outcomes, generated from the reference
implementation, living in `spec/vectors/` and "frozen per spec version (a change
that flips an outcome is a new spec version, not an edit)."

That sentence settles one case — flipping an outcome — and leaves the rest of the
corpus's lifecycle unstated:

- may a published vector be **removed**?
- may a published vector be **altered** without flipping its outcome — different
  artifact bytes, a different reason code, edited metadata?
- may new vectors be **added** within a spec version, or does any addition
  require a new one?

These are not academic. The conformance story ([0006](0006-spec-versioning.md),
0010) and the library-lifecycle gate (0013) both use the corpus as the
**arbiter**: the thing that decides, when two implementations disagree, which one
diverged. An arbiter that can be edited to match a failing implementation is not
an arbiter. The corpus's mutation rules therefore have to be explicit and strict,
independent of the spec-version-bump rule.

## Decision

Generated vector material is **immutable and append-only.**

- **Never removed, never altered.** Once a vector is published in a spec version,
  its artifact bytes, its expected outcome, its reason code, and its identity are
  permanent. It is a fixture of that spec version for all time.
- **May be added, under one condition.** A vector may be appended only when it
  codifies behavior the current spec version **already requires** — a
  *clarifying* vector that makes an existing prose-level requirement executable.
  Such additions need no spec bump; any implementation they newly fail was
  already non-conformant.
- **New requirements and outcome flips are spec events.** Any change that would
  flip an existing vector's outcome, or that pins behavior the current spec does
  **not** already determine, is a new requirement — a spec-version event under
  [0006](0006-spec-versioning.md), a new frozen set for the new version — not an
  edit to the existing one.
- **Generation is one-directional.** Vectors are generated from the reference
  implementation and reviewed (0010); once published they are frozen as above.
  Regeneration must never alter or drop a published vector. If a regeneration
  *would* change one, that divergence is a bug in the generator or the reference
  implementation, to be fixed without mutating the published corpus.

The boundary between a clarifying addition and a new requirement is a review
judgment; it is exactly where a spec-vs-library call gets made.

## Consequences

- The corpus is a stable, permanent contract. A conformance result is
  reproducible for all time within a spec version, and the permanent backward
  compatibility of [0006](0006-spec-versioning.md) gains an executable witness
  that cannot silently move.
- The library-lifecycle gate (0013) can treat vectors as a fixed arbiter: when a
  candidate and a stable peer disagree, the vectors decide who diverged, and
  neither side can be exonerated by editing the corpus.
- The corpus only grows. Storage and runner cost rise monotonically — acceptable;
  vectors are small and the growth is the point.
- Clarifying additions are the sanctioned way to close an interop gap found in
  the field without cutting a new spec version, *provided* the behavior was
  already required.
- A genuinely wrong published vector cannot be patched in place; correcting it is
  a spec-version event. This is deliberate — it makes shipping a wrong vector
  expensive and keeps the frozen set trustworthy.

## Alternatives considered

- **Mutable vectors with review** — allow editing a published vector when
  reviewers agree. Rejected: it lets the arbiter be edited to match a failing
  implementation and makes a past conformance claim unreproducible. The entire
  value of vectors as a contract is that they do not move.
- **Freeze the whole set per spec version, no in-version additions** — a simpler
  rule, but it forces a spec-version bump to codify behavior the spec already
  requires, conflating "we clarified an existing requirement" with "we changed
  the contract." The append-only carve-out keeps clarification cheap without
  weakening the freeze.
- **Per-vector versioning** (each vector carries its own semver) —
  over-engineered; the spec version is already the unit of the contract, and
  per-vector versioning invites exactly the in-place mutation this ADR forbids.
