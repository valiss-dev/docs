# 0024. Distributing the valiss CLI through an in-house Homebrew tap

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

[0017](0017-cli-tool.md) fixed the CLI's name, repository, and install
mechanics, making `go install valiss.dev/cli/valiss@latest` the baseline: it
installs a binary named `valiss` with no `cmd/` ceremony. That path serves
anyone who already runs a Go toolchain. It serves no one else. An operator
managing a valiss trust domain is not necessarily a Go developer, and asking
them to install Go to get a single static binary is a toolchain tax on the
tool's most operational users.

[0020](0020-credential-storage.md) settled the storage stack on pure Go end to
end: no cgo, plain cross-compilation, no C toolchain at build or run time. That
removes the usual obstacle to distributing prebuilt binaries. The CLI
cross-compiles to every target that matters as a self-contained artifact, so a
package manager can ship a downloaded binary rather than build from source.

The question is how to reach non-Go users on macOS (and Linux) without standing
up a bespoke installer. Homebrew is the dominant package manager on macOS and
runs on Linux as Linuxbrew, and it supports third-party formula distribution
through taps: a tap is an ordinary GitHub repository of formulae that a user
adds with `brew tap`. The choice is whether to own that tap, submit the formula
to Homebrew's central `homebrew-core`, or reach for a lighter mechanism such as
a `curl | sh` installer.

## Decision

**The valiss CLI is distributed through an in-house Homebrew tap owned by the
`valiss-dev` org.** The tap is a first-class release channel alongside
`go install`, aimed at users who do not have or do not want a Go toolchain.

- **The tap repository is `valiss-dev/homebrew-tap`.** Homebrew mandates the
  `homebrew-` prefix on tap repositories and strips it when resolving a tap
  reference, so the user-facing invocation is `brew tap valiss-dev/tap`
  followed by `brew install valiss`, or the one-shot
  `brew install valiss-dev/tap/valiss`. The bare formula name stays `valiss`,
  consistent with the binary and everything else named in
  [0017](0017-cli-tool.md).

- **The `homebrew-` prefix is a sanctioned exception to
  [0007](0007-repository-naming-convention.md).** By 0007's dividing test the
  tap is a supporting repository (nothing pulls it into a build as a
  dependency), so its name would be bare. Homebrew's platform requirement
  overrides that: the prefix is not our choice but a precondition for
  `brew tap valiss-dev/tap` to resolve at all. This ADR records `homebrew-tap`
  as a named, platform-dictated exception to the bare-naming rule rather than a
  precedent for prefixing supporting repositories in general.

- **The formula installs prebuilt binaries from GitHub Releases.** Because the
  CLI is pure Go per [0020](0020-credential-storage.md), release artifacts
  cross-compile per target without a C toolchain, and the formula points at
  the release asset for the host platform rather than building from source.

- **Formula updates are automated by the release pipeline.** On each CLI
  release, the pipeline (goreleaser or an equivalent) publishes the binaries
  and pushes the updated formula to the tap. The specific tool is left open;
  this ADR fixes the distribution channel, not the automation that feeds it.

- **`go install valiss.dev/cli/valiss@latest` remains the toolchain-native
  path** per [0017](0017-cli-tool.md), unchanged. The tap does not replace it;
  the two channels coexist, each serving the audience it fits.

## Consequences

- The org owns release cadence and rollback. A bad release is corrected by
  pushing a corrected formula to a repository we control, with no external
  review in the loop.
- The tap works from day zero, at v0.x, with no notability or maturity bar to
  clear. Distribution is available the moment the first release is cut.
- macOS and Linux (via Linuxbrew) are covered by one channel. Windows and other
  package managers (apt, scoop, winget) stay future work; nothing here forecloses
  them.
- It is one more repository to declare in the infra Terraform per
  [0016](0016-vanity-module-roots.md)'s companion practice for org repositories.
  This ADR decides to adopt the tap; it does not create the repository. The tap
  is declared in infra when it is created.
- The release pipeline gains a step: publishing binaries and pushing the formula.
  That coupling is the price of automation, and it keeps the tap from drifting
  behind releases by hand.

## Alternatives considered

- **Contributing the formula to `homebrew-core`.** The central tap installs
  without any `brew tap` step and carries the most trust. Rejected as premature:
  `homebrew-core` expects notability and stability that fit a mature project, its
  review latency is outside the org's control, and pre-1.0 iteration speed
  matters more now than the reach of the central tap. Revisit once the CLI is
  established.
- **`go install` only.** The baseline from [0017](0017-cli-tool.md), and
  retained, but it excludes every user without a Go toolchain, which is exactly
  the operational audience the CLI most needs to reach.
- **A `curl | sh` installer.** A single script can fetch the right binary, but
  it has no upgrade or uninstall story, leaves the user tracking versions by
  hand, and asks them to pipe a remote script into a shell. Homebrew supplies
  the upgrade, uninstall, and provenance machinery that a script would have to
  reinvent, with a better trust posture.
