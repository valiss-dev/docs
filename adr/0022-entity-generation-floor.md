# 0022. Entity generation floor: wire-level reflection of store generations

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

[0021](0021-cli-command-surface.md) makes generations universal in the CLI's
store: every entity carries a generation counter, the store is append-only,
and a generation bumps on an invalidating change (key rotation, removal, an
extension-policy change). That generation counter is store-local. Nothing
about it reaches the wire, so a verifier cannot act on it.

Today revocation is per-jti: a server fails closed against an allowlist whose
entries are jti content hashes, and a jti leaving the allowlist is the
revocation. Rotating a key or invalidating a generation of tokens therefore
means enumerating and removing every affected jti. When an account rotates,
every live user token underneath it must be found and struck one by one, even
though "everything below generation N is stale" is a single fact.

The reference library is valiss-go (v0.13.1); this decision is scoped to what
the wire and the verifier must do to carry an entity's generation, and its
implementation is deferred to valiss-go 0.14.

## Decision

**A new valiss domain extension stamps a token with the issuing entity's own
generation.** It is a self-naming typed claim carried through the standard
`WithExtension` plumbing, like http grants, grpc grants, and custom domains.
It carries the issuing entity's own generation only. There is deliberately no
chain vector: a token reflects the generation of the entity that signed it,
not the generations of its ancestors.

**Backwards compatibility is a hard requirement; the extension is optional at
both ends.** An issuer may stamp or not. A verifier may enforce or not. An
unstamped token verifies exactly as it does today. A verifier not configured
for floors ignores the stamp entirely. Adding the extension changes no
existing verification outcome.

**Enforcement composes with the existing allowlist.** The allowlist keeps its
per-jti entries as today, and gains optional per-entity generation floors (for
example, "account X floor is 8"). A verifier configured to enforce rejects a
stamped token whose generation is below its entity's floor. Rotation and
removal become a floor bump: one entry, rather than an enumeration of every
jti below the line.

**A template reference may travel as a concealed digest.** The extension may
carry a template reference as a short digest (four to six characters) of the
template name, keyed with the per-template salt stored in the template record
per [0021](0021-cli-command-surface.md). Template names never appear on the
wire. Collisions are prevented at template-creation time by regenerating the
salt (per 0021), so a digest resolves to one template within an operator. An
audit joins a token back to its template through the issuance record, not
through anything readable on the wire.

**Sequencing.** The extension name and its claim schema get registered first.
Conformance vectors follow the append-only rule of
[0012](0012-vector-immutability.md): stamped-token vectors are added as
clarifying material, and floor-rejection vectors join the interop matrix once
the library actually enforces. CLI adoption comes after the library ships the
extension; the CLI's store schema is already generation-ready from inception
per [0021](0021-cli-command-surface.md) and
[0020](0020-credential-storage.md), so no schema break is needed to start
stamping.

## Consequences

- Rotation and removal scale as one floor bump instead of an enumeration over
  every descendant jti. Killing a generation of tokens stops being O(tokens).
- The change is purely additive on the wire. Every token minted before the
  extension exists, and every verifier not configured for floors, behaves
  exactly as before; there is no flag day.
- Floors and jti entries coexist in the allowlist. A specific token can still
  be struck individually while a floor handles the wholesale case; the two
  mechanisms do not compete.
- Own-generation-only keeps the stamp and the verifier logic simple: a token
  states one number, the verifier compares it to one floor. The cost is that
  invalidating an ancestor does not by itself lower a descendant's stamp;
  cascade still works through the signing chain (a broken chain fails to
  resolve, per [0021](0021-cli-command-surface.md)), and the floor handles the
  ancestor's own generation.
- Template names stay off the wire; the salted digest reveals only that some
  template was used, and the mapping back is an audit-side join. This closes
  the leak of putting human template names in tokens and logs.
- The append-only conformance corpus grows by clarifying additions now and
  floor-rejection vectors once enforcement lands, exactly as
  [0012](0012-vector-immutability.md) prescribes; no published vector moves.

## Alternatives considered

- **A full chain generation vector in the stamp** (issuer generation plus
  every ancestor's generation). It would let a verifier reason about the whole
  chain from one token, at the cost of a larger claim, a stamp that must be
  recomputed when any ancestor bumps, and more wire surface to conceal.
  Rejected: the signing chain already propagates ancestor invalidation
  (a revoked ancestor fails to resolve), so the extra generations buy little
  for their weight. Own-generation-only is the smaller, sufficient primitive.
- **A mandatory extension on all tokens.** Uniform enforcement, but it breaks
  every existing token and every verifier not yet aware of floors, violating
  the backwards-compatibility requirement. Optionality at both ends is what
  makes this shippable without a flag day.
- **A separate floor list outside the allowlist.** A second distribution
  channel and a second thing for a server to fetch and trust, for a fact that
  is a natural allowlist entry. Folding floors into the allowlist keeps one
  artifact for servers to consume, matching the export in
  [0021](0021-cli-command-surface.md).
- **Plain template names on the wire.** Simplest to implement and to audit,
  and it publishes an operator's internal template vocabulary into every token
  and every server log. The salted digest keeps the audit join while giving
  the wire nothing legible.
- **Keeping per-jti revocation as the only mechanism.** No new extension, no
  new schema, and rotation stays O(tokens) forever. Rejected: the enumeration
  cost is the specific pain this decision removes, and the store already tracks
  the generations that make floors cheap.
