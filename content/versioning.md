---
title: Versioning and compatibility
weight: 6
description: "The wire-spec and library version axes, and how cross-language interoperability is gated by the conformance vectors."
---

valiss is a multi-language framework, and its whole value is that a credential
minted by one implementation verifies in every other. That guarantee rests on
keeping two different things versioned separately: the **wire specification**
that all implementations speak, and the **library** that each language ships.
You reason about interoperability through the spec version, never through a
library's semantic version.

This page covers both axes, how conformance to a spec version is proven, and how
cross-library compatibility is gated in practice.

## Two version axes

- **The spec version** identifies the wire format and verification rules. It is
  independent of any library's version, and an implementation advertises the
  spec version or versions it implements (`spec: 1`). Backward compatibility
  within a spec version is permanent: a verifier that implements spec 1 can
  always parse and check any spec-1 credential, for all time. That is a format
  guarantee, not a verdict: whether a given credential then passes is a separate
  question of validity windows, allowlist membership, and epoch, which the format
  guarantee says nothing about.
- **The library version** is each implementation's own semantic version, moving
  at its own pace. valiss-go, the reference implementation, is at v0.13.x. A
  library upgrade that does not touch the spec still interoperates, and that is
  proven rather than assumed (see the interop gate below).

This separation is deliberate. Tying interoperability to library semver breaks
down immediately, because libraries version independently (ADR 0006).

## Spec versioning

The current spec version is **1**, specified in `SPEC-1.md` in the `spec`
repository. It is normative and language-neutral: it pins the format at the byte
and algorithm level, and the Go reference implementation is the source of truth
where the two ever disagree.

Once accepted, a spec document is frozen. Only errata (clarifications that do not
change conforming behavior) are applied to it. A breaking change is a new
`SPEC-2.md`, never an edit to `SPEC-1.md`. An implementation may later implement
several spec versions at once (for example {1, 2}), which is how a future format
coexists with the current one without ever breaking a spec-1 holder (ADR 0006).

## Wire-format versioning

For two spec versions to coexist, a verifier must decide from the bytes alone
which version an artifact is, then dispatch to the right rules or reject cleanly,
rather than mis-parse. valiss carries an explicit version discriminator on each
of its three independently versioned artifacts (ADR 0009):

- **Tokens** carry `"ver":1` in the JWT header. A version-agnostic reader reads
  the envelope and the version without touching the payload, then dispatches to
  the matching per-version decoder. The current wire version is 1.
- **The creds file** carries a `VALISS-CREDS-VERSION: 1` header line, checked
  before the payload. It versions the file container only; the tokens inside
  carry their own token version. An absent header reads as the current version.
- **The request signature** binds a version tag (`valiss-req-v1`) into the
  signed bytes.

Two rules hold for every version. The version selects the parser, but the
signature is always verified, so a forged version either routes to a real parser
that rejects the bad signature or is an unknown version rejected outright.
Downgrade is blocked because the version is part of the signed material: a
request signed under any other version cannot match a v1 reconstruction, so it
fails closed. And the JWS-compact envelope is fixed across versions, so the
version tag stays readable.

Tokens are always minted at the current version; any supported version can be
read. Adding a version is additive: a new set of per-version types, one decoder,
and one dispatch case, with nothing outside that set changed.

## Conformance

Conformance to a spec version is proven, not asserted, through a two-layer model
(ADR 0010).

**Static vectors are the wire contract.** The `spec/vectors/` corpus pairs each
artifact (tokens, signatures, creds files) with its expected outcome: either
success with the verified claims, or failure with a reason code from the spec's
error taxonomy. The vectors are verify-side and deterministic, and every
implementation ships an offline runner that must pass all of them. They are
normative and live with the specification, frozen per spec version.

**The live interop matrix is the transport proof.** A dedicated `interop`
repository runs the grid of server language against client language against
transport (HTTP and gRPC), starting each server and running each client suite
against it, cross-language included. It catches the divergences the vectors
cannot: header names, content negotiation, middleware wiring on a real exchange.

The ordering is wire-first. The vector layer gates the matrix, so if the wire
disagrees the matrix does not run on a broken foundation.

The vectors are the arbiter of who diverged when two implementations disagree,
which only works if they cannot be edited to match a failing implementation. So
the corpus is immutable and append-only (ADR 0012): a published vector is never
removed and never altered. New vectors may be appended only when they make an
*existing* spec requirement executable; anything that would flip an outcome or
pin new behavior is a spec-version event, not an edit. If regenerating the
vectors would change a published one, that is a bug to fix, not a corpus to
mutate.

## The interop gate

Conformance runs continuously, but the risk of a regression reaching a published
release is concentrated at non-patch releases, where new surface lands. A stable
release that breaks the frontier is the worst outcome, because peers already pin
to it. The library-lifecycle gate is built to design that outcome out (ADR
0013).

The stable frontier is an explicit manifest in the `interop` repository, pinning
exactly one stable version per language. The standing matrix runs that frontier
against itself on every merge, and the per-implementation vector runner runs on
every release. These two always-on layers cover patch releases and continuous
drift.

On top of them sits a beta gate for non-patch releases. A candidate is validated
against every stable peer in the manifest, as both server and client, across both
transports, behind the wire-first vector layer. When it passes, it is tagged
stable and the manifest is updated to pin it, and that manifest edit is the
promotion. Candidates are validated against the stable frontier only, never
against another candidate, so the gate is deterministic. When a candidate and a
peer disagree, the immutable vectors decide who diverged, and the corpus is never
edited to turn a red cell green.

The effect is that a stable release cannot reach the frontier without first
proving it interoperates with the whole frontier.

## Library version lines

- **valiss-go** is the reference implementation, module `valiss.dev/valiss`,
  currently on the v0.13.x line. It is the source of truth for the spec and the
  generator for the conformance vectors.
- **Other implementations** (valiss-py, and future languages) track the same
  spec version and prove it through the same vectors and the same matrix. A new
  language is done only when it passes the vectors and joins the matrix.

While libraries are pre-1.0 and the wire is not yet frozen, nearly every release
touches versioned surface, so a candidate is cut often. That loosens naturally
once libraries reach 1.0 maturity: from then on, minor and major releases cut a
candidate first and patch releases are exempt.

## Supported Go version

valiss-go targets **Go 1.26 or newer**, as its `go.mod` declares.

## Related

- [Tokens](/docs/concepts/tokens/): the wire shape the version discriminator
  lives in, and why valiss tokens are deliberately not stock JWTs.
- [Verifying credentials offline](/docs/guides/verifying/): the language-neutral
  verification recipe every implementation conforms to.
- [Security model](/docs/security/): the threat model behind these guarantees.
