---
title: Quickstart
weight: 1
---

This walks the shortest working path with the Go library as it ships today:
install it, create an operator, account, and user, issue their tokens, and
verify a signed request server-side against the allowlist. Everything runs in
one process against a local listener, so you can see the full loop before
splitting it across a real client and server.

## Prerequisites

- Go 1.26 or newer.

## Install

```sh
go get valiss.dev/valiss
```

The core module carries token issue and verify, the request `Verifier`, and
the allowlist. The `contrib/httpauth` package (pulled in by the same module)
adds the net/http middleware and client transport this example uses.

## The whole loop

Drop this into `main.go` and run `go run .`. It issues the three-level key
chain, stands up an HTTP server that trusts only the operator public key, and
makes one signed request through it.

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"net/http/httptest"
	"time"

	"github.com/nats-io/nkeys"

	"valiss.dev/valiss"
	"valiss.dev/valiss/contrib/httpauth"
	"valiss.dev/valiss/creds"
)

func main() {
	// 1. Operator: the trust anchor. Its public key is the one thing a
	// server pins; its seed never leaves the issuer.
	operator, err := nkeys.CreateOperator()
	must(err)
	operatorPub, err := operator.PublicKey()
	must(err)

	// 2. Account: one per tenant. The operator signs its token. The HTTP
	// extension bounds what the tenant may call; the contrib transports
	// enforce it fail closed, so every token in the chain must carry it.
	account, err := nkeys.CreateAccount()
	must(err)
	accountPub, err := account.PublicKey()
	must(err)
	accountToken, err := valiss.IssueAccount(operator, accountPub,
		valiss.WithName("acme"),
		valiss.WithExtension(httpauth.Ext{Methods: []string{"GET"}, Paths: []string{"/v1/*"}}),
		valiss.WithTTL(time.Hour),
	)
	must(err)

	// 3. User: one per end user or service. The account signs its token,
	// attenuating no further than it holds.
	user, err := nkeys.CreateUser()
	must(err)
	userPub, err := user.PublicKey()
	must(err)
	userSeed, err := user.Seed()
	must(err)
	userToken, err := valiss.IssueUser(account, userPub,
		valiss.WithName("alice"),
		valiss.WithExtension(httpauth.Ext{Methods: []string{"GET"}, Paths: []string{"/v1/*"}}),
		valiss.WithTTL(time.Hour),
	)
	must(err)

	// The allowlist admits the account token by its id (jti). Removing that
	// id revokes the whole tenant, every user under it, without touching a
	// single key.
	acctClaims, err := valiss.VerifyAccount(accountToken, operatorPub)
	must(err)
	allowlist := valiss.NewStaticAllowlist(acctClaims.ID)

	// Server: the operator public key plus the allowlist are all it needs.
	// It never sees a seed. The middleware verifies the chain and injects the
	// identity; the handler reads it.
	mux := http.NewServeMux()
	mux.HandleFunc("/v1/whoami", func(w http.ResponseWriter, r *http.Request) {
		id, _ := valiss.IdentityFromContext(r.Context())
		fmt.Fprintf(w, "verified tenant %q, user %q\n", id.Account.Name, id.User.Name)
	})
	verifier := valiss.NewVerifier(operatorPub, allowlist)
	server := httptest.NewServer(httpauth.NewMiddleware(verifier)(mux))
	defer server.Close()

	// Client: its creds are the token chain plus the user seed that signs
	// each request. The transport signs transparently.
	clientCreds := creds.Creds{
		AccountToken: accountToken,
		UserToken:    userToken,
		Seed:         userSeed,
	}
	transport, err := httpauth.NewTransport(clientCreds, nil)
	must(err)
	client := &http.Client{Transport: transport}

	resp, err := client.Get(server.URL + "/v1/whoami")
	must(err)
	body, _ := io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Printf("%s: %s", resp.Status, body)
}

func must(err error) {
	if err != nil {
		log.Fatal(err)
	}
}
```

Running it prints:

```
200 OK: verified tenant "acme", user "alice"
```

## What just happened

- The server was configured with **only** the operator public key and the
  allowlist. It holds no seeds and made no network call to authenticate the
  request.
- The client signed the request with the user seed. The middleware walked the
  chain (operator signed the account token, the account signed the user token),
  checked the signature over the request, and confirmed the account token's id
  is on the allowlist.
- `IdentityFromContext` handed the handler the verified tenant and user. Reads
  after that point trust the identity without re-checking anything.

Try breaking it to see the guarantees hold. Drop `userToken` and `Seed` from
the creds and the server answers 401. Request `POST /v1/whoami` or a path
outside `/v1/*` and it answers 403, because the request falls outside the
account's HTTP extension.

## Revoking

Revocation is an allowlist edit, not a key operation. To cut off the tenant,
remove its account token id:

```go
// remainingIDs is your tenant list minus this one, re-read from the allowlist
// file you maintain today.
allowlist.Set(remainingIDs)
// allowlist.Set(nil) // the nuke: revokes every tenant at once
```

The next request from that tenant, and from every user under it, fails
verification. `StaticAllowlist` is in-memory; production servers load ids from
a file with `valiss.LoadAllowlistFile`. That call reads the file once and
installs no watcher, so the caller drives reloads: a file watch, a SIGHUP
handler, or a poll interval re-reads the file and applies it with `Set` on the
retained allowlist. Validate the new set before `Set`, because an empty file
revokes every tenant. The [Go guide](/docs/guides/go/#the-allowlist) shows the
full pattern.

## From here to production

This example issues keys in-process for clarity. In a real deployment the
signing seeds live in a secrets manager, not the server. The
[`examples/minter`](https://github.com/valiss-dev/valiss-go/tree/main/examples/minter)
tool in the Go repository issues credentials from a manifest of public keys,
resolving seeds from the environment, and writes a credentials file the client loads
with `creds.Load`. The server side is unchanged: it still pins the operator
public key and consults the allowlist.

The valiss CLI (early development) will be the issuer-side counterpart to this
in-process issuance: keeping operator keys and tokens in an encrypted
per-operator store and running issuance and creds export from there, so an
operator issues credentials without writing Go. Its command tree is designed
but not yet runnable (every command is currently a stub), so today that
issuer-side issuance is the library `Issue*` calls above or `examples/minter`.

## Next steps

- [Entities](/docs/concepts/entities/) and [Tokens](/docs/concepts/tokens/):
  the model behind the three-level chain.
- [Allowlist](/docs/concepts/allowlist/): revocation in depth.
- [Extensions](/docs/concepts/extensions/): the typed authorization claims the
  HTTP extension above is one instance of.
- [Rotation](/docs/concepts/rotation/): epochs and operator tokens for rotating
  or mass-revoking a whole trust domain.
- [Go guide](/docs/guides/go/): gRPC wiring, Gin and Echo, bearer tokens, and
  the rest of the library surface.
- [Verifying credentials offline](/docs/guides/verifying/): how a receiver in
  any language checks a valiss credential.
