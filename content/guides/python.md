---
title: Python
weight: 2
---

`valiss` is the Python client for the valiss scheme: it parses credentials
files, issues short-lived tokens, signs HTTP and gRPC requests, and verifies
them server-side without a round trip to the Go reference. It implements wire
spec 1 and interchanges credentials, tokens, and signatures byte for byte with
the Go and TypeScript implementations.

This guide covers installing, loading credentials, attaching credentials to
`httpx` and `requests` clients, and verifying tokens and requests. It does not
cover the gRPC adapters, the framework middleware (Django, ASGI), or the
message-token webhook transports beyond a pointer; those live in the package
README.

## Install

The core (creds parsing, token issuance, request signing, verification) has one
dependency, `cryptography`. HTTP client support is optional extras.

```sh
pip install valiss              # core
pip install 'valiss[httpx]'     # + the httpx auth hook
pip install 'valiss[requests]'  # + the requests auth hook
```

With `uv`:

```sh
uv add valiss
uv add 'valiss[httpx]'
uv add 'valiss[requests]'
```

Python 3.11 or newer is required. Other extras (`grpc`, `grpcsig`, `django`,
`fastapi`) enable the gRPC and server-side integrations.

## Load credentials

A credentials file holds a subject's tokens plus the Ed25519 seed that signs
its requests. `creds.load` reads and parses one:

```python
from valiss import creds

c = creds.load("alice.creds")
```

`creds.parse(contents)` parses an in-memory string, and you can build a `Creds`
directly from its parts. The three fields are `account_token` (operator-signed),
`user_token` (account-signed), and `seed`:

```python
from valiss import creds

c = creds.Creds(
    account_token=account_token,
    user_token=user_token,
    seed=user_seed,
)
```

Credentials without a seed are bearer credentials: `c.signer()` returns `None`
and the holder cannot sign requests. The server accepts them only when the
effective token is a bearer user token.

## Authenticate requests with httpx

`httpauth.Auth` is an httpx auth hook. It attaches the credentials' tokens and,
when the credentials carry a seed, a fresh per-request signature bound to the
request method, host, and path:

```python
import httpx
from valiss import creds, httpauth

c = creds.load("alice.creds")
client = httpx.Client(auth=httpauth.Auth(c))
client.get("https://api.example.com/v1/whoami")
```

If the server runs a replay cache, pass `nonce=True` to attach a fresh
per-request nonce folded into the signature:

```python
client = httpx.Client(auth=httpauth.Auth(c, nonce=True))
```

`Auth` validates the seed at construction time, so a malformed seed fails
immediately rather than on the first request.

## Authenticate requests with requests

`httpauth.RequestsAuth` is the sibling hook for the `requests` library, with the
same behavior and the same `nonce=True` flag. Set it on a session or pass it per
call:

```python
import requests
from valiss import creds, httpauth

session = requests.Session()
session.auth = httpauth.RequestsAuth(creds.load("alice.creds"))
session.get("https://api.example.com/v1/whoami")
```

The signature binds the host the wire actually carries: an explicit `Host`
header wins, otherwise the URL host with a non-default port kept.

## Authenticate any other client

Both hooks are thin wrappers over `httpauth.credential_headers`, which returns
the header dict for one request. Use it with any HTTP client. The signature is
bound to the method, host, and path you pass, so pass the real ones and build a
fresh header set per request:

```python
from valiss import creds, httpauth

c = creds.load("alice.creds")
headers = httpauth.credential_headers(c, "GET", "api.example.com", "/v1/whoami")
```

For a replay-cache server, generate a nonce and pass it through:

```python
from valiss import token

nonce = token.new_nonce()
headers = httpauth.credential_headers(
    c, "GET", "api.example.com", "/v1/whoami", nonce=nonce,
)
```

## Verify tokens and requests

A Python service can authenticate a request itself. The library ships the full
verification chain, at parity with the Go reference.

### The integrated verifier

`Verifier` pins the operator public key (the trust anchor) and turns the
credential a transport pulled off a request into a verified `Identity`. It
checks the account token against the operator key, expiry and activation, the
allowlist (revocation), the optional user-token chain, and the request signature
(or a bearer waiver), and suppresses replays:

```python
from valiss import Verifier, Request, StaticAllowlist, MemoryReplayCache, ValissError

verifier = Verifier(
    operator_pub,                      # the pinned trust anchor
    StaticAllowlist(account_jti),      # revocation: drop the id to revoke
    replay_cache=MemoryReplayCache(),  # reject a replayed nonce
)

try:
    identity = verifier.verify(Request(
        account_token=account_token,
        user_token=user_token,
        timestamp=timestamp,
        signature=signature,
        context=context,
        nonce=nonce,
    ))
except ValissError as exc:
    ...  # exc.reason is the spec section 7 reason code
```

The verified `Identity` carries `identity.account` (always present),
`identity.user` (`None` for account-level requests), and `identity.operator`.
Each is a claims object with `name`, `subject`, `epoch`, expiry, and decoded
extension claims.

`Request.context` is the transport's canonical description of the request that
the signature is bound to. For HTTP, `httpauth.request_context(method, host,
path, nonce)` produces the exact bytes the client signed. The `httpauth`
middleware and the `grpcauth` interceptor wrap the verifier with header
extraction and status-code mapping, so a handler only ever sees an authenticated
request; see the package README for those.

`ALLOW_ALL` accepts any account id (for development or when revocation is handled
elsewhere). The allowlist is a protocol tested with `in`, so any container of ids
works in place of `StaticAllowlist`, including a database-backed object that reads
your own revocation store. For a service that trusts several operators, build the
verifier with `Verifier.with_keyring(Keyring(*operator_tokens), allowlist)`; the
credential's account token names the trust domain. Pass `operator_token=` to
enforce the trust domain's epoch and validity window, and `resolver=` (a callable
or a `{account_pub: token}` mapping) to accept user-only credentials.
`replay_cache=` likewise takes any `ReplayCache` implementation, so back the
process-local `MemoryReplayCache` with Redis or a database for exactly-once
suppression across instances.

### Custom checks

Register application checks that run after possession is proven:

```python
@verifier.validator
def tenant_is_active(request, identity):
    ...  # raise ValissError to reject

@verifier.extension(httpauth.Ext)
def enforce_paths(request, identity, account_ext, user_ext):
    ...  # typed HTTP extension enforcement
```

### Verifying a single token

For tooling and tests, the `token` module verifies one token's signature, type,
and issuer without the full chain. These check the cryptographic binding, not
expiry, the allowlist, or the trust chain:

```python
from valiss import token

account = token.verify_account(account_token, operator_pub)  # -> AccountClaims
user = token.verify_user(user_token, account.subject)        # -> UserClaims
```

And to check a bare request signature against a subject public key within the
skew window:

```python
context = httpauth.request_context("GET", "api.example.com", "/v1/whoami")
token.verify_signature(user.subject, timestamp, signature, context)
```

### Verifying a message token

A message token is a self-signed proof of origin (a webhook body, say), verified
offline against the operator key. It is a proof, never a credential:

```python
from valiss import message

claims = message.verify_message(
    proof, operator_pub,
    audience="https://api.example.com/ingest",
    payload=body,
)
```

`verify_message` walks the operator, account, user, and message chain, checks
epoch agreement and every validity window, and enforces the audience and
checksum bindings.

## Errors

Every failure raises `ValissError`. Its `reason` attribute is the stable spec
section 7 reason code (a `Reason` enum member), so a server can map a failure to
a status code without string matching.
