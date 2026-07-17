# 0019. Command framework: the spf13 suite for CLI tools, stdlib flags for servers

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

[0017](0017-cli-tool.md) fixed the valiss CLI's name, repository, and module
path, and deferred the command framework. Its surface is a noun-verb tree in
the spirit of NATS' `nsc` (`valiss account add`, `valiss user mint`,
`valiss creds export`), with config conventions already established there:
`~/.valiss/` and `VALISS_*` environment variables.

The org ships two distinct binary shapes. End-user CLI tools (distributables
per [0007](0007-repository-naming-convention.md)) live on discoverability:
nested help, shell completion, and config bound across files, environment,
and flags. Servers (the interop harness servers today, service binaries
tomorrow) are single-purpose processes whose flag surface is flat and small.
The two shapes call for different parsing trade-offs, and leaving the choice
ad hoc invites every new binary to reinvent it.

## Decision

- **CLI tools are built on the full spf13 suite**: cobra for the command
  tree, viper for configuration (binding the `~/.valiss/` dot-dir, `VALISS_*`
  environment variables, and flags), pflag as it ships with cobra.
- **Servers follow the house style**: single-entry commands with no
  subcommands, the standard library `flag` package only, no third-party
  parsing or configuration libraries.
- The decision scope is binary entry points. Libraries never take these
  dependencies: a distributable's parsing and configuration stack must not
  leak into library modules, which stay dependency-minimal per
  [0015](0015-contrib-auth-sig-separation.md).

## Consequences

- CLI dependency weight is confined to distributable modules; valiss-go and
  future libraries keep their minimal dependency sets.
- Servers keep a one-file main with zero parsing dependencies. A server
  growing subcommands becomes a visible smell rather than an accident.
- The [0017](0017-cli-tool.md) config conventions get a standard
  implementation through viper instead of hand-rolled binding.
- valiss-cli bootstraps on cobra and viper from day one.

## Alternatives considered

- **stdlib `flag` dispatch for the CLI** (the valiss-go plain-toolchain
  style) — serviceable for flat surfaces; on a noun-verb tree it reimplements
  help routing and shell completion within weeks.
- **kong** — lighter and structurally typed, but cobra is the incumbent for
  nsc-alike tools and its completion and doc generation are the mature parts
  we would otherwise build.
- **cobra without viper** — the dot-dir/env/flag binding would be hand-rolled
  against conventions [0017](0017-cli-tool.md) has already fixed.
- **cobra for servers as well** — a single-purpose process gains nothing from
  a subcommand tree and pays the dependency cost for it.
