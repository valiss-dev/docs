# 0006. Wire spec is versioned independently of library semver; current spec is 1

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

> The format-version discriminator this ADR gates on is specified and
> implemented in [0009](0009-wire-format-versioning.md); the current wire
> version is 1.

## Context

valiss is a multi-language framework whose value depends on
cross-implementation interoperability: a credential minted by one language's
library must verify in every other. Each library evolves at its own pace and
carries its own semantic version, but the wire format they all speak must have a
single, stable identity that consumers can rely on and libraries can advertise.

The Go reference implementation is at 0.11.0 and its wire format is not yet
frozen.

## Decision

- Define a **wire spec version** that is **independent of any library's semantic
  version**. Each library advertises the spec version(s) it implements
  (`spec: 1`).
- The **current spec version is 1.** Freezing spec 1 is gated on the wire format
  carrying a **format-version discriminator** so that a future spec 2 can
  coexist; if the current format lacks one, adding it is the single pre-freeze
  breaking change.
- **Backward compatibility is permanent**: a verifier that implements spec 1
  verifies any spec-1 credential for all time. Libraries may later implement
  multiple spec versions (e.g. {1, 2}).
- Conformance to a spec version is proven against shared, language-neutral test
  vectors (positive and negative, negative cases tagged with a shared
  reason-code registry).

## Consequences

- Consumers reason about interoperability via the spec version, not library
  versions.
- Requires a normative spec document and a conformance vector suite (home and
  format to be decided in follow-up ADRs).
- The pre-freeze audit for a format-version discriminator is a prerequisite task
  and may force one breaking change to valiss-go before spec 1 is frozen.

## Alternatives considered

- Tying interoperability to library semver — breaks down immediately because
  libraries version independently.
- Embedding no version discriminator and relying on structural detection —
  makes a future breaking wire change ambiguous and unsafe.
