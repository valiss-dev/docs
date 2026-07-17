# 0020. Credential storage: pluggable backends, one encrypted DuckDB file per operator

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
interface from day one. For the local backend, one database file per
operator keeps trust domains isolated, movable (one file is an operator's
whole domain state), and independently deletable.

DuckDB 1.4.0 LTS (Andium) added native database encryption in the core
engine: AES-256-GCM over the main file, the WAL, and temporary files, with
only a minimal header left plaintext, opened via `ATTACH ... (ENCRYPTION_KEY
'...')`. The `duckdb-go` driver ships an Andium LTS release line, so the
feature is available from Go.

## Decision

- **Storage is behind a `Store` interface from inception**; the first
  implementation is the local filesystem backend.
- **The local backend keeps one DuckDB database file per operator** under
  the storage directory, `~/.valiss/store/<operator>.duckdb`. The schema is
  an implementation detail; this ADR fixes the layout and the encryption,
  not the tables.
- **Encryption is mandatory**: the backend always creates and opens files
  with `ENCRYPTION_KEY`, pinning DuckDB ≥ 1.4.0 with the default
  AES-256-GCM cipher. There is no plaintext mode.
- **The passphrase is sourced from the `VALISS_STORAGE_KEY` environment
  variable**; when unset, the CLI prompts interactively with hidden input on
  startup. The CLI never writes the passphrase to disk.

## Consequences

- cgo enters the CLI build through `duckdb-go`; there is no pure-Go DuckDB
  driver. `go install` works on the driver's supported platforms, and
  goreleaser cross-compilation needs per-target C toolchains (the zig-cc
  pattern). The binary grows by the libduckdb footprint. Per
  [0019](0019-command-framework.md) this weight is confined to the
  distributable; libraries are untouched.
- Prompt-on-startup means prompt-per-invocation interactively; scripted use
  sets the environment variable. OS keychain caching of the passphrase is
  the designated future relief, added at the `Store` level without a format
  change.
- Backing up or moving an operator is a single-file operation. The
  plaintext minimal header reveals that the file is DuckDB but nothing about
  its contents.
- Passphrase rotation is a re-encryption of the file, a follow-up command,
  not an inception requirement.
- Future backends (keychain, KMS, remote) implement the same interface; the
  local format is stable within the DuckDB LTS line.

## Alternatives considered

- **Badger v4 built-in encryption** — pure Go and KV-shaped, but its
  encryption covers values only: keys and metadata stay in cleartext, and
  for a credential store the operator, account, and subject names are
  themselves sensitive.
- **SQLite + SQLCipher** — same cgo cost as `duckdb-go` plus a bespoke
  OpenSSL/SQLCipher toolchain per target, for the same shape of result.
- **age-encrypted blob files** (one per entity) — pure Go and the simplest
  crypto, but every list or lookup decrypts whole files, multi-entity writes
  are not atomic, and the result is a database rebuilt badly by hand.
- **Plaintext files, relying on full-disk encryption** (the `nsc` model) —
  outsources the decision to the machine's owner and fails the moment a
  store directory is copied elsewhere.
