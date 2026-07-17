# 0020. Credential storage: pluggable backends, one encrypted SQLite file per operator

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

[0017](0017-cli-tool.md) deferred the CLI's credential storage design. The
store holds the most sensitive material in the system: nkey seeds for
operators, accounts, and users, alongside their tokens and chain metadata.
At-rest exposure of a seed is total compromise of that identity, so
encryption cannot be a later addition: retrofitting it into a plaintext
format forces a migration and leaves early adopters exposed in the meantime.

The store must support multiple backends over time (local filesystem first;
OS keychain, KMS, or remote stores later), so storage sits behind an
interface from day one. One database file per operator keeps trust domains
isolated, movable (one file is an operator's whole domain state), and
independently deletable.

The engine selection is bounded by four forces:

- No relational queries are foreseen today, but the option must stay open;
  a pure key-value store forecloses it.
- Pure Go is strongly preferred: cgo is tolerable only as a build-time
  dependency expressible in packaging (Homebrew), never as a runtime one.
- The passphrase-to-key derivation should be a documented library
  construction, not one we design and maintain ourselves.
- Databases need random page access (`io.ReaderAt`), so a streaming
  encryptor like age cannot sit under a live engine; the encryption
  separation must be page-level (a VFS layer) or whole-image (a
  serialize/re-encrypt envelope).

`gosqlite.org` (`github.com/go-again/sqlite`, Apache-2.0) is a pure-Go
SQLite stack on the mature `modernc.org/sqlite` engine. Its `vfs/crypto`
package provides page-level encryption at rest (Adiantum by default,
AES-XTS-256 as an option) covering the main database, journal, WAL, and
temporary files, with `crypto.DeriveKey` for passphrase derivation. Its
`vfs/cksm` package adds page checksums for tamper-evidence, and its
`Serialize`/`Deserialize` support keeps the whole-image envelope pattern
available. `liteorm.org` (same org) is a typed query builder and declarative
ORM over the same core, with Postgres, MySQL, and SQL Server backends.

## Decision

- **Storage is behind a `Store` interface from inception**; the first
  implementation is the local filesystem backend.
- **The local backend is `gosqlite.org` (pure-Go SQLite, WAL mode), one
  database file per operator** at `~/.valiss/store/<operator>.db`.
- **Encryption is mandatory**: the `vfs/crypto` page-level VFS with its
  default Adiantum cipher, always on. There is no plaintext mode. Changing
  the cipher is a store-format change and would be handled as a migration.
- **Tamper-evidence is layered in via `vfs/cksm`**, stacked per gosqlite's
  documented order (checksum over ciphertext), as defense-in-depth for seed
  material.
- **The passphrase is sourced from the `VALISS_STORAGE_KEY` environment
  variable**; when unset, the CLI prompts interactively with hidden input on
  startup. It is passed through `crypto.DeriveKey` and never written to
  disk.
- **`liteorm.org` is the typed mapping layer** for the store schema; the
  schema itself is an implementation detail.
- gosqlite and liteorm are pre-v1: versions are pinned and upgrades are
  reviewed deliberately.

## Consequences

- Pure Go end to end: no cgo, plain cross-compilation, trivial goreleaser
  and Homebrew builds, and `go install valiss.dev/cli/valiss@latest` stays
  light per [0017](0017-cli-tool.md).
- The SQL option stays open: relational queries (rotation history, audit
  trail) are available if the store grows into them.
- A future remote `Store` backend reuses the liteorm mapping over its
  Postgres or MySQL backend; only the `Store` implementation changes.
- Dependency youth is the accepted risk: gosqlite and liteorm are v0.x with
  a fast release cadence. The blast radius is confined by the `Store`
  interface and version pinning, and the engine core (`modernc.org/sqlite`)
  is mature.
- Adiantum or XTS alone provide confidentiality only; the cksm layer is what
  makes tampering detectable.
- Prompt-on-startup means prompt-per-invocation interactively; scripted use
  sets the environment variable. OS keychain caching of the passphrase is
  the designated future relief, added at the `Store` level without a format
  change.
- Backing up or moving an operator is a single-file operation; the WAL
  sidecar is transient and checkpoints on clean close.
- `Serialize` keeps an age-encrypted whole-image export available as a
  backup format, without involving it in live access.

## Alternatives considered

- **DuckDB native encryption** (the earlier draft of this ADR) — genuine
  whole-file AES-256-GCM with an ecosystem-interoperable passphrase, but cgo
  plus libduckdb per target, and an OLAP engine for a KV-shaped workload.
- **Badger v4 built-in encryption** — pure Go, but key-value only (closes
  the SQL option), encrypts values while leaving the key namespace and
  structural metadata cleartext (acceptable with public nkeys as keys, still
  a leak shape), and the passphrase KDF would be ours to design.
- **SQLite + SQLCipher** — the same shape of result through cgo plus a
  bespoke OpenSSL/SQLCipher toolchain per target.
- **age-encrypted blob files** — age is streaming-only and cannot back a
  live database; viable solely as a whole-image envelope, which the chosen
  stack retains via `Serialize`.
- **Plaintext files, relying on full-disk encryption** (the `nsc` model) —
  outsources the decision to the machine's owner and fails the moment a
  store directory is copied elsewhere.
