---
title: Go
weight: 1
---

The Go reference implementation is the module `valiss.dev/valiss`. It covers
the whole scheme: minting operator, account, user, and message tokens;
packaging credentials; and verifying per-request credentials server-side.
Transport and framework adapters live under `contrib/`.

This guide assumes the key model. An **operator** key is the trust anchor a
server pins. The operator signs **account** (tenant) tokens; an account
signs **user** tokens; the subject signs every request with its own key.
Keys are NATS nkeys, and levels are strict: operator (`SO...` seed, `O...`
public), account (`SA...`/`A...`), user (`SU...`/`U...`).

## Installing

```
go get valiss.dev/valiss
```

The module requires Go 1.26. Key generation and signing use
`github.com/nats-io/nkeys`, which comes in as a dependency.

## Issuing tokens

Tokens are minted from key pairs. A signer's seed never leaves the issuing
side; the server only ever sees public keys and tokens.

```go
import (
	"time"

	"github.com/nats-io/nkeys"
	"valiss.dev/valiss"
)

// One-time: the operator key pair is the trust anchor. Keep the seed
// offline; publish only the public key.
operator, _ := nkeys.CreateOperator()
operatorPub, _ := operator.PublicKey()

// Per tenant: an account key pair. The tenant holds the seed.
account, _ := nkeys.CreateAccount()
accountPub, _ := account.PublicKey()

// The operator signs an account token binding the tenant's public key.
accountToken, err := valiss.IssueAccount(operator, accountPub,
	valiss.WithName("acme"),
	valiss.WithTTL(720*time.Hour),
)
```

`IssueAccount(operator, tenantPubKey, opts...)` returns the compact token
string. The parallel issuers are `IssueOperator(operator, opts...)` for the
self-signed operator token (see [Rotation](#rotation-and-epochs)),
`IssueUser(account, userPubKey, opts...)` for a tenant delegating to an end
user, and `IssueMessage(user, opts...)` for per-message proofs (see
[Message tokens](#message-tokens)). Each checks that the signing key is the
right nkey type and that the subject key matches the level.

A tenant delegates to a user with its own account seed:

```go
user, _ := nkeys.CreateUser()
userPub, _ := user.PublicKey()

userToken, err := valiss.IssueUser(account, userPub,
	valiss.WithName("alice"),
	valiss.WithTTL(24*time.Hour),
)
```

### Issue options

`IssueOption` values customize a minted token. The identity issuers
(`IssueOperator`, `IssueAccount`, `IssueUser`) share these:

- `WithName(string)` labels the entity (the token's `name`). An unnamed
  entity is represented by its public key. Names are issuer-asserted and not
  checked for uniqueness at issuance.
- `WithTTL(d)` / `WithExpiry(t)` set the expiry (`exp`). Without either, the
  token never expires.
- `WithNotBefore(t)` sets the activation time (`nbf`).
- `WithEpoch(uint64)` stamps the trust-domain epoch (see
  [Rotation](#rotation-and-epochs)).
- `WithExtension(Extension)` embeds a named authorization claim (see
  [Extensions](#authorization-extensions)).
- `WithBearer()` (only `IssueUser`) marks a token the server accepts without
  per-request signatures. Bearer tokens are replayable until they expire, so
  pair them with TLS and a short window.

`WithAudience`, `WithChecksum`, and `WithChain` apply only to message
tokens; passing them to an identity issuer is an error.

### Packaging credentials

The `valiss.dev/valiss/creds` package renders a client's tokens and signing
seed into one marker-delimited file, everything a client needs to
authenticate:

```go
import "valiss.dev/valiss/creds"

seed, _ := account.Seed()
text := creds.Format(creds.Creds{AccountToken: accountToken, Seed: seed})
// write text to acme.creds

// Client side:
c, err := creds.Parse(text)      // or creds.Load(path)
```

`Creds` carries `AccountToken`, an optional `UserToken`, and the `Seed` that
signs requests (nil for bearer creds). User-level creds may omit the account
token when the server resolves it (see
[Account token resolution](#user-only-credentials)); a bundle embeds it.

The `examples/minter` command in the repository is a complete,
manifest-driven credential minter built on these APIs.

## Server-side verification

A `Verifier` checks the full per-request credential: the account token
against the pinned operator key, expiry and activation, allowlist
membership, the optional user-token chain, extensions, and the request
signature within the skew window.

```go
verifier := valiss.NewVerifier(operatorPub, allowlist, opts...)
```

The server needs only the operator public key and an allowlist. It never
holds any seed. Transport adapters (below) wrap the verifier with header
extraction and error mapping; you rarely call `VerifyRequest` directly.

### The allowlist

The allowlist decides whether an issued account token (by its `jti`) is
still accepted, so a token can be revoked before expiry by removing it. The
`jti` a server accepts is the account token's id; the minter prints it as
token metadata.

```go
// In-memory set, reloadable with (*StaticAllowlist).Set.
allowlist := valiss.NewStaticAllowlist(jti1, jti2)

// Newline-delimited file of jtis ('#' comments and blank lines ignored).
allowlist, err := valiss.LoadAllowlistFile("/etc/valiss/allowlist")

// Accept every token; for local development. Signature and expiry still gate.
allowlist := valiss.AllowAll{}
```

### Verifier options

- `WithSkew(d)` overrides the default 2-minute window for timestamp drift
  and expiry slack.
- `WithReplayCache(cache)` rejects a signed request whose nonce was already
  seen. Signed clients must then send a nonce (enable `WithNonce` on the
  client transport). `valiss.NewMemoryReplayCache()` is a built-in cache.
- `WithExtensionType[T]()` eagerly rejects a request carrying a malformed
  instance of extension `T`, moving the failure to auth time. Retrieval with
  `ExtOf` never requires it.
- `WithClaimsValidator(fn)` injects custom validation that runs after the
  chain is verified and possession of the subject key is proven, so an
  expensive check never fires for a party that merely captured a token.
  `ExtValidator[T]` adapts a typed extension validator to this shape.
- `WithOperatorToken(token)` enforces a trust-domain policy (see
  [Rotation](#rotation-and-epochs)).
- `WithAccountTokenResolver(fn)` accepts user-only credentials (see
  [below](#user-only-credentials)).

### The verified identity

Verification yields a `*valiss.Identity`. Handlers read it from the request
context and use it to segment data:

```go
id, ok := valiss.IdentityFromContext(ctx)
// id.Account *AccountClaims  - the tenant; always present.
// id.User    *UserClaims     - the delegated user; nil for account-level requests.
// id.Operator *OperatorClaims - the trust domain; set under a keyring or operator token.
```

`id.Account.Name` is the tenant label to key storage on; `id.Account.ID` is
the allowlist `jti`.

### User-only credentials

A request may carry only a user token, leaving the server to supply the
account token. Configure a resolver:

```go
resolver, err := valiss.StaticAccountTokens(accountToken1, accountToken2)
verifier := valiss.NewVerifier(operatorPub, allowlist,
	valiss.WithAccountTokenResolver(resolver))
```

Without a resolver, a credential missing its account token is rejected.

### Trusting several operators

A server serving more than one trust domain verifies against a `Keyring` of
self-signed operator tokens. The credential names its domain (the account
token's issuer plus its epoch), so the right entry is selected without
trial, and handlers tell domains apart by `id.Operator.Name`.

```go
keyring, err := valiss.NewKeyring(operatorTokenA, operatorTokenB)
verifier := valiss.NewKeyringVerifier(keyring, allowlist)
```

Keyring entries carry their own policy (validity window and exact epoch), so
`WithOperatorToken` does not combine with a keyring. The allowlist is shared
across domains; account `jti`s are content hashes and cannot collide between
producers.

### Rotation and epochs

The operator can publish a self-signed operator token: a policy statement
over the trust domain carrying an epoch counter and an optional validity
window. A single-anchor verifier configured with it accepts only account and
user tokens that echo the current epoch.

```go
opToken, _ := valiss.IssueOperator(operator, valiss.WithName("prod-us"), valiss.WithEpoch(3))
acctToken, _ := valiss.IssueAccount(operator, accountPub,
	valiss.WithName("acme"), valiss.WithEpoch(3), valiss.WithTTL(720*time.Hour))

verifier := valiss.NewVerifier(operatorPub, allowlist,
	valiss.WithOperatorToken(opToken))
```

Bumping the epoch and re-minting rotates the whole domain at once: every
token from an earlier epoch is rejected cryptographically, with no allowlist
edits. The operator token's own `exp` bounds the entire domain. The pinned
public key remains the trust anchor; the operator token only carries policy
signed by it. Selective revocation stays with the allowlist. Unstamped
tokens are epoch 0.

## Authorization extensions

Authorization rides named extension claims: typed payloads the scheme signs
and transports but assigns no meaning. The HTTP and gRPC transports each
define and enforce their own; your application adds domain extensions the
same way.

An extension is a value struct whose method reports its name:

```go
type queryFilters struct {
	Regions []string `json:"regions"`
}

func (queryFilters) ExtensionName() string { return "example.filters" }
```

Embed it at mint time with `valiss.WithExtension(queryFilters{...})` and read
it back in a handler with `valiss.ExtOf[queryFilters](id.Account.Ext)`, which
returns the value, an `ok` bool, and a decode error. Extensions on both
chain levels are enforced together, so an account-level grant bounds all of
its users.

### The HTTP grant

`contrib/httpauth` defines `Ext`, binding a token to hosts, methods, and
paths:

```go
import "valiss.dev/valiss/contrib/httpauth"

valiss.WithExtension(httpauth.Ext{
	Methods: []string{"GET"},
	Paths:   []string{"/v1/*"},   // trailing "*" is a prefix wildcard
})
```

The three dimensions are independent AND-filters, each constraining only
when populated: an empty dimension imposes no restriction on that dimension.
Naming only `Paths` therefore leaves every method and host open. Scope a
read-only surface by naming every dimension you care about. Allow-all within
a dimension is the explicit wildcard `[]string{"*"}`; the zero-value `Ext{}`
grants nothing.

### The gRPC grant

`contrib/grpcauth` defines a single-dimension `Ext` over full method names:

```go
import "valiss.dev/valiss/contrib/grpcauth"

valiss.WithExtension(grpcauth.Ext{
	Methods: []string{"/example.v1.WidgetService/*"},   // or a specific method, or "*"
})
```

An empty `Methods` list grants nothing; allow-all is the explicit `"*"`.

Enforcement in both transports is fail-closed: every token in the chain must
carry the extension, or the request is denied. Pass `AllowMissingExtension()`
to the middleware only when authorization is handled entirely outside the
transport.

## Wiring the transports

### net/http

`httpauth.NewMiddleware` returns a standard `func(http.Handler) http.Handler`
that authenticates every request and enforces the HTTP extension.
Unauthenticated requests get 401, extension denials get 403.

```go
mw := httpauth.NewMiddleware(valiss.NewVerifier(operatorPub, allowlist))
srv := &http.Server{Handler: mw(appHandler)}

// In a handler:
id, _ := valiss.IdentityFromContext(r.Context())
```

Because it is standard middleware, any router that routes plain
`net/http` handlers is covered without a dedicated adapter: chi,
gorilla/mux, httprouter, negroni, and the like. Wire it at each router's
middleware point.

The client attaches the credential and signs each request with an
`http.RoundTripper`:

```go
transport, err := httpauth.NewTransport(c, nil) // c is creds.Creds; nil base = http.DefaultTransport
client := &http.Client{Transport: transport}
```

Enable `httpauth.WithNonce()` on the transport whenever the server has a
replay cache. Bearer creds (no seed) attach the token but no signature.

### gRPC

`contrib/grpcauth` provides server interceptors and client per-RPC
credentials:

```go
auth := grpcauth.NewAuthenticator(valiss.NewVerifier(operatorPub, allowlist))
srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(auth.UnaryInterceptor()),
	grpc.StreamInterceptor(auth.StreamInterceptor()),
)

// Client:
rpcCreds, err := grpcauth.NewCredentials(c)
conn, err := grpc.NewClient(target,
	grpc.WithTransportCredentials(tlsCreds),
	grpc.WithPerRPCCredentials(rpcCreds),
)
```

Rejections map to `codes.Unauthenticated` (auth) and `codes.PermissionDenied`
(extension). gRPC refuses to send per-RPC credentials over an insecure
connection; `rpcCreds.AllowInsecure()` relaxes that for an already-encrypted
local tunnel.

## Framework integrations

Frameworks with their own handler and context types get dedicated adapters
that expose the verified identity through the native context and use native
abort semantics. Per [ADR 0014](../../adr/0014-go-framework-integration-targets.md),
targets are selected by GitHub stars and shipped effort-class first; the
first wave is Gin and Echo. Every adapter wraps `httpauth`'s verification
core (`httpauth.Authenticate`) rather than reimplementing it, so the HTTP
extension and all verifier options behave identically. The client side stays
framework-agnostic: use `httpauth.NewTransport`.

Adapters follow a naming split ([ADR 0015](../../adr/0015-contrib-auth-sig-separation.md)):
`<framework>auth` packages carry credential authentication and grant an
identity (`IdentityFrom`); `<framework>sig` packages carry message-token
verification and grant no identity (`MessageFrom`). Protection on a route is
visible in its imports.

### Gin

```go
import "valiss.dev/valiss/contrib/ginauth"

engine := gin.New()
engine.Use(ginauth.NewMiddleware(valiss.NewVerifier(operatorPub, allowlist)))

engine.GET("/v1/whoami", func(c *gin.Context) {
	id, _ := ginauth.IdentityFrom(c)
	c.String(http.StatusOK, "tenant %q", id.Account.Name)
})
```

`NewMiddleware` returns a `gin.HandlerFunc`; rejections abort with the right
status. `ginauth.IdentityFrom(c)` reads the verified identity.

### Echo

```go
import "valiss.dev/valiss/contrib/echoauth"

app := echo.New()
app.Use(echoauth.NewMiddleware(valiss.NewVerifier(operatorPub, allowlist)))

app.GET("/v1/whoami", func(c echo.Context) error {
	id, _ := echoauth.IdentityFrom(c)
	return c.String(http.StatusOK, "tenant "+id.Account.Name)
})
```

`NewMiddleware` returns an `echo.MiddlewareFunc`; rejections surface as
`*echo.HTTPError` (401 or 403), so the app's error handler renders them.

Runnable end-to-end programs for both are in `examples/ginauth` and
`examples/echoauth` in the repository.

## Message tokens

A service that emits artifacts to third parties (webhooks, queue messages,
exported documents) can extend the chain one level further with message
tokens: a short-lived token minted with a user key, one per emitted message,
that any receiver verifies offline knowing only the operator public key. A
message token binds the destination (`aud`), the exact payload bytes (a
SHA-256 checksum), a short window, and the epoch.

```go
payload := renderWebhook(event)
tok, err := valiss.IssueMessage(user,
	valiss.WithAudience("https://receiver.example/hook"),
	valiss.WithChecksum(valiss.Checksum(payload)),
	valiss.WithTTL(30*time.Second),
	valiss.WithChain(accountToken, userToken),
)

// Receiver, holding only the operator public key:
claims, err := valiss.VerifyMessage(tok, operatorPub,
	valiss.ExpectAudience("https://receiver.example/hook"),
	valiss.WithPayload(receivedBody),
)
// claims.Account.Name / claims.User.Name identify the emitter.
```

Message tokens must carry an expiry. `WithChain` embeds the provenance chain
so the token is self-contained; `WithChainTokens` on the verify side instead
takes the chain out of band. `ExpectAudience` closes cross-destination
replay and `WithPayload` closes payload tampering; receivers should set both.
Stored messages verify after expiry with `valiss.At(receivedAt)`. Transport
adapters `contrib/httpsig`, `contrib/grpcsig`, `contrib/ginsig`, and
`contrib/echosig` wire this up end to end.

**A message token is a proof, not a credential.** Possession grants nothing:
`Verifier.VerifyRequest` never accepts one, and receivers must not treat one
as a bearer credential. The detailed verification algorithm, for reference or
for porting to another language, is in
[Verifying tokens](verifying.md).
