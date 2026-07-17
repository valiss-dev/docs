---
title: Extensions
weight: 4
---

Authentication tells you *who* is calling. Authorization tells you *what* they
may do. In valiss the second question is answered by **extensions**: named,
typed claims carried in a token's `ext` field, signed along with the rest of the
token and delivered unchanged to the server.

The scheme itself assigns extensions no meaning. It signs and transports each
one opaquely; meaning belongs to whoever registered the name. That is what lets
the transport layers and your own application both ride the same mechanism
without colliding.

## Self-naming typed claims

An extension is any struct that names itself. Implement one method and valiss
carries the value end to end, handing the same concrete type back on the server
with no string plumbing in between:

```go
type QueryFilters struct {
    Regions []string `json:"regions"`
}

func (QueryFilters) ExtensionName() string { return "acme.filters" }
```

The name is the key under `ext`, so it must be unique within a token; a producer
cannot emit two extensions with the same name. Mint the value into a token at
issue time, recover it in the handler by type:

```go
tok, _ := valiss.IssueUser(account, alicePub,
    valiss.WithName("alice"),
    valiss.WithExtension(QueryFilters{Regions: []string{"eu"}}),
    valiss.WithTTL(time.Hour),
)

// In the handler, on the verified identity:
filters, ok, err := valiss.ExtOf[QueryFilters](id.User.Ext)
```

You can also validate extensions at authentication time rather than in the
handler: registering `valiss.WithExtensionType[QueryFilters]()` on the verifier
rejects a request whose token carries a malformed `acme.filters` claim before
your handler runs.

## Transport authorization is built on extensions

The HTTP and gRPC integrations are not special-cased in the core; they are
extensions like any other, which is the clearest demonstration that the
mechanism is enough to build real authorization on.

- **`contrib/httpauth`** defines an extension named `http` with three
  independent dimensions, `Ext{Hosts, Methods, Paths}`. Requests outside the
  stated bounds are rejected with 403.
- **`contrib/grpcauth`** defines an extension named `grpc`, `Ext{Methods}`.
  Methods outside the stated set are rejected with PermissionDenied.

Each dimension constrains only when populated. A dimension you leave empty
imposes no restriction, so `Ext{Paths: []string{"/admin/*"}}` permits those
paths under *any* method. To scope a read-only admin surface you must name every
dimension you mean to bound:

```go
httpauth.Ext{Methods: []string{"GET"}, Paths: []string{"/admin/*"}}
```

## Fail closed, and enforce down the chain

Transport enforcement fails closed. Every token in the chain must carry the
transport extension, the zero-value extension grants nothing, and allow-all is
the *explicit* wildcard (`Methods: ["*"]`, `Paths: ["*"]`), never the default.
A deployment that authorizes entirely outside the transport can opt out with
`AllowMissingExtension()`, but that is a deliberate choice, not an accident of
an unset field.

Enforcement also compounds down the chain, and it is a verify-time property, not
a mint-time one. `IssueUser` runs no subset check: an account can mint a user
token whose extension names broader bounds than the account's own. What contains
it is verification. When both the account and the user token carry the
extension, the verifier authorizes a request only if every level permits it (an
AND across the chain), so the effective grant is the intersection. A user token
that claims more than its account is simply capped at the account's bounds when
verified, and a tenant handed a capped account credential cannot exceed that cap
no matter what it mints.
