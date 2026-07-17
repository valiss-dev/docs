# 0021. The valiss CLI command surface

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

[0017](0017-cli-tool.md) fixed the CLI's name, repository, and module path;
[0019](0019-command-framework.md) put it on cobra and viper; and
[0020](0020-credential-storage.md) settled storage (a `Store` interface, one
encrypted SQLite file per operator). What remains is the command surface
itself: the nouns, the verbs, and the addressing model that operators type
every day.

The reference point is NATS' `nsc`, and it is also the anti-goal. `nsc`
carries an invisible "current" operator and account as hidden state, so a
command's effect depends on context the terminal does not show. An
issuance minted against the wrong account is a silent mistake. The valiss
domain is a signing chain of operator, account, and user, keyed by nkeys,
where a single mint writes real authority; the surface must make the target
of every operation legible in the command itself.

Two facts about the domain shape the vocabulary. First, store entities
(operator, account, user) and the tokens they sign are different kinds of
thing with different lifecycles: an entity is a durable identity, a token is
a dated issuance. Second, the store is append-only and generation-ready from
inception per [0020](0020-credential-storage.md); the surface has to expose
generations and history, not just current state, and it has to stay ready for
the wire-level generation reflection that
[0022](0022-entity-generation-floor.md) adds later.

## Decision

**Noun-verb tree on cobra/viper, with explicit path addressing.** Entities
are addressed by their position in the chain, `<operator>/<account>/<user>`,
spelled out on the command line. There is no hidden current-context, ever.
Nothing the CLI does depends on ambient state the invocation does not name.
This is the deliberate reversal of `nsc`'s invisible operator/account
selection.

**Two verb families for two lifecycles.** Store entities (operator, account,
user, template) take `add | list | show | remove | audit`; operator
additionally takes `rotate`. Tokens are issuances and take
`mint | list | show | revoke`. The asymmetry is intentional vocabulary: an
entity is never "revoked" and a token is never "removed." A token leaves the
world by revocation (its jti leaves the allowlist); an entity leaves by
removal (a tombstone in the append-only store).

**`remove` cascades, and shows the blast radius first.** Removing an entity
revokes its live jtis; because authority is a signing chain, descendants'
tokens die cryptographically once the parent no longer resolves. The CLI
prints the full blast radius (which entities and how many live tokens fall)
and asks for confirmation before acting. `--yes` skips the prompt for
scripts.

**Claim templates are per-operator store objects holding claimsets only.** A
template carries repeatable claim material: extension grants (http, grpc,
custom domains), TTL, the bearer flag, a description. It never carries
identity claims: `iss`, `sub`, `jti`, and `iat` come from the addressed
entity and the mint, never from a template. Templates carry generations. A
`template add` with new content under an existing name creates the next
generation of `<name>`; a bare name at mint time pins the latest generation;
`--template name@<N>` pins generation `N` exactly (a numeric pin, no `v`
prefix). A template is a mint-time stamp, not a live reference: every
issuance record stores the template name, generation, and a content hash, so
an audit reads correctly even after the template evolves. Template generations
are reference-retained: a generation is kept as long as any retained issuance
record still references it. `template remove` retires the name for new mints,
and its generations garbage collect as their references age out of retention.
Each template record also carries a short random salt for the wire-level
name-digest scheme of [0022](0022-entity-generation-floor.md); a digest
collision is detected at `template add` and resolved by regenerating the salt.
Composition stays boring: one `--template` per mint, explicit grant flags
union with the template's grants, and `--ttl` overrides the template's TTL.

**Generations are universal.** Every entity carries a generation counter and
the store is append-only from inception. Every generation and every removed
(tombstoned) entity is retained within the retention window. A generation
bumps on an invalidating change (key rotation, removal, an extension-policy
change) and can be forced with an explicit CLI option. The term is chosen
deliberately: "generation" separates an entity's lifecycle counter cleanly
from spec versions ([0006](0006-spec-versioning.md),
[0009](0009-wire-format-versioning.md)) and from implementation and release
versions, three numbers that would otherwise all be called "version." The
store schema accounts for generations from day one; the wire-level reflection
of these generations arrives later per
[0022](0022-entity-generation-floor.md).

**Audit is an append-only journal, read through a per-entity verb.** Every
operation is journaled: entity add and remove, token mint and revoke,
template add and retire, creds export, allowlist operations. Each entry is
timestamped and records the entity path and the operation's details. The
`audit` verb is scoped to the addressed entity's subtree: `operator audit`
covers the whole domain, `account audit` the account plus its users,
`user audit` that user, and `template audit` the generation history plus the
mints that reference it. Retention is a store-global parameter set at
`store init --audit-retention <duration>` (default `2160h`, ninety days;
`0` keeps forever; adjustable later). Sweeping is lazy, on store open, with
no daemon. The journal is time-swept; template generations are
reference-retained; an aged-out terminal issuance record is what eventually
frees an old template generation.

**Artifacts and store management round out the surface.**

- `creds export <path>` (`--bundle`, `--bearer`), covering the four creds
  kinds the format supports.
- `allowlist list | add | remove | export`; `export` produces exactly what
  servers consume.
- `inspect <token>`: offline decode of a token, no trust evaluation.
- `store init | info | config`.

**Mint fails closed on extensions.** `token mint` requires one of: a
`--template`, at least one explicit grant flag (`--http`, `--grpc`, ...), or
an explicit `--no-extension`. An unqualified mint is an error, not an
unrestricted token.

**Mint registers the jti in the local allowlist by default.** `--no-allowlist`
opts out, for the case of minting into a domain whose allowlist lives
elsewhere.

**Scriptable by default.** `--json` on `list` and `show`, `--yes` on every
destructive confirmation. The only interactive prompt is the storage
passphrase per [0020](0020-credential-storage.md), and only when
`VALISS_STORAGE_KEY` is unset.

**Deferred, and named as deferred.** A `valiss nats` compatibility namespace
that reimplements NATS credential management (for example
`valiss nats operator init`) is a plausible later addition and is explicitly
not built now. A `suspend` (soft-stop) verb and template inheritance are
likewise deferred.

## Consequences

- Every command names its target, so a mint or a removal cannot land on the
  wrong entity through invisible context. The cost is verbosity: paths are
  typed in full, with no `nsc`-style shorthand for "the current account."
- The entity/token verb split makes the vocabulary teach the model: you learn
  that entities are removed and tokens are revoked because the CLI will not
  let you say it any other way.
- `remove` showing its blast radius makes cascade deletion a deliberate act.
  The confirmation is the safety rail; `--yes` moves the responsibility to the
  script author.
- Templates as generation-stamped mint-time records keep audits truthful
  across template change, at the cost of retaining old template generations
  until their referencing issuances age out. Storage grows with history; that
  is the point of an append-only store.
- Universal generations obligate the schema to model generations and
  tombstones from the first migration, and it is what makes the
  [0022](0022-entity-generation-floor.md) wire reflection a later, additive
  step rather than a schema break.
- Fail-closed mint and default allowlist registration make the safe outcome
  the one you get by typing less; the unsafe or unusual outcomes
  (`--no-extension`, `--no-allowlist`) are the ones you have to ask for.

## Alternatives considered

- **A hidden current-context, `nsc`-style.** Fewer keystrokes per command, at
  the price of commands whose effect depends on unshown state. Rejected on the
  central lesson of operating `nsc`: the invisible selection is exactly the
  failure mode this CLI exists to avoid.
- **One uniform verb set for entities and tokens** (for example `delete` for
  both). Simpler to document, but it erases the lifecycle distinction the
  domain actually has and invites the category error of "deleting" a token or
  "revoking" an entity.
- **Live template references instead of mint-time stamps.** Smaller store,
  simpler retention, but an audit would then reflect the template's current
  content rather than what was actually minted, and editing a template would
  silently rewrite history. The content-hash stamp is what keeps the audit
  trustworthy.
- **Current-state-only store, no universal generations.** Cheaper schema, but
  it forecloses the generation floors of
  [0022](0022-entity-generation-floor.md) and leaves rotation and removal
  without a history to audit. Retrofitting generations into a current-state
  schema is the migration this ADR spends its schema budget to avoid.
- **Mint defaulting to an unrestricted token when no grants are given.**
  Convenient, and precisely the fail-open behavior a service-auth tool must
  not have. Requiring `--no-extension` to be explicit keeps the empty case
  loud.
