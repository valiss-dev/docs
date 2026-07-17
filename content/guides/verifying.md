---
title: Verifying tokens
weight: 4
---

valiss tokens are structurally JWTs but not verifiable with stock JWT
libraries: the algorithm identifier and key encoding are valiss-specific. In
Go the `valiss.dev/valiss` package does all of this for you, `Decode` and the
`Verify*` functions for identity tokens, `VerifyMessage` for message tokens,
`VerifySignature` for request signatures. This guide is the specification-level
recipe underneath those functions. Read it when you verify offline and want to
know exactly what is checked, or when you port verification to another
language. Everything here is derivable from the Go source, which is canonical
if they ever disagree.

## Wire format

A token is JWS Compact Serialization: `base64url(header) "." base64url(payload)
"." base64url(signature)`, no padding.

The header is byte-exact:

```json
{"typ":"JWT","alg":"ed25519-nkey","ver":1}
```

`ver` is the wire-format version. Read it before parsing the payload and
reject a version you do not implement; the current version is `1`.

The payload is a JSON object with RFC 7519 field names, all optional except
`valiss`, serialized in this order:

| field    | meaning                                              |
| -------- | ---------------------------------------------------- |
| `jti`    | content-derived token id (see below)                 |
| `iat`    | issue time, Unix seconds                             |
| `iss`    | issuer public key, nkey-encoded                      |
| `name`   | optional human label of the subject                  |
| `sub`    | subject public key, nkey-encoded                     |
| `aud`    | destination identity (message tokens)                |
| `exp`    | expiry, Unix seconds; absent = never expires         |
| `nbf`    | not-before, Unix seconds; absent = immediately valid |
| `valiss` | the level-specific claim body                        |

The `valiss` section is discriminated by its `type` field:

- `operator` - `{type, epoch?, ext?}`; self-signed (`iss == sub`).
- `account` - `{type, epoch?, ext?}`; signed by an operator key.
- `user` - `{type, epoch?, bearer?, ext?}`; signed by an account key.
- `message` - `{type, epoch?, checksum?, chain?, ext?}`; self-signed by a
  user key. `checksum` is the lowercase-hex SHA-256 of the message payload.
  `chain` is `{account, user}`: the embedded provenance tokens, verbatim.

`ext` is an object of named, consumer-defined claim payloads; transport
carries them opaquely. The registry of names is open: a verifier that does not
recognize an extension name must carry it through untouched rather than reject
it, so a port that ignores unknown extensions stays forward-compatible.
Fail-closed decoding applies only to names a verifier explicitly opts to check;
under those, a present entry that does not match the expected shape rejects the
token.

> [!IMPORTANT]
> A verifier must carry an extension name it does not recognize through untouched
> rather than reject it. SPEC-1 keeps the extension registry open, and a port that
> drops or refuses unknown names breaks forward compatibility with tokens issued
> against a newer vocabulary. Fail-closed shape checking applies only to the names
> a verifier explicitly opts to enforce.

## Signature

Standard Ed25519 (RFC 8032) over the signing input `base64url(header) "."
base64url(payload)`, the ASCII bytes exactly as they appear in the token. The
signature part is the raw 64-byte Ed25519 signature, base64url-encoded without
padding. Any Ed25519 implementation verifies it once you extract the raw
public key from the nkey encoding.

## Decoding an nkey public key

`iss` and `sub` carry public keys in the NATS nkey text format:

```
base32( prefix_byte || raw_ed25519_public_key[32] || crc16_le[2] )
```

- Base32: RFC 4648 alphabet, uppercase, no padding.
- Prefix byte encodes the key role: operator `112` (renders as leading `O`),
  account `0` (`A`), user `160` (`U`).
- CRC16: CCITT/XMODEM (polynomial 0x1021, init 0x0000), computed over
  `prefix || key`, appended little-endian.

To decode: base32-decode, check length 35, verify the trailing CRC16 over the
first 33 bytes, check the prefix byte matches the role required by context,
take bytes 1..33 as the raw Ed25519 public key.

Role requirements are strict: an operator key signs account tokens, an account
key signs user tokens, a user key signs message tokens; operator and message
tokens are self-signed (`iss == sub`).

## jti derivation

`jti` is content-derived, not random: serialize the payload with `jti`
omitted (every field is omit-when-empty), SHA-256 the JSON bytes, and
base32-encode the digest (RFC 4648, uppercase, no padding). Reproducing it
requires byte-identical JSON: fields in the table's order, no whitespace, and
Go `encoding/json` escaping rules (which HTML-escape `<`, `>`, and `&`), so
re-deriving from a reparsed payload is only possible if your serializer
reproduces that exact shape. Two tokens with identical claims issued within
the same second therefore share a `jti`; the allowlist's registration dedup
relies on this. Verifiers that only check signatures may treat `jti` as
opaque.

## Verifying a token (one level)

1. Split on `.`; require exactly three parts.
2. Base64url-decode the header; require `typ == "JWT"`,
   `alg == "ed25519-nkey"`, and `ver == 1`. Reject an unrecognized `ver`
   before parsing the payload.
3. Base64url-decode the payload; parse JSON.
4. Decode the `iss` nkey; verify the Ed25519 signature over `part1 "." part2`.
5. Require `valiss.type` to match the expected level, `iss` to equal the
   expected issuer key (established by the level above, or pinned), and `sub`
   to be a valid nkey of the level's subject role.

This establishes authenticity only. Trust and freshness are chain-level checks
below. Validity windows: a token is expired when `now > exp + skew`, not yet
valid when `now + skew < nbf`; absent fields impose nothing. valiss uses a
default skew of 2 minutes.

> [!NOTE]
> Freshness rests entirely on the verifier's own clock. There is no issuer to
> consult, so the skew window is measured against local time, and a verifier with
> a wrong clock silently accepts expired tokens or rejects valid ones. Keep the
> verifier's clock disciplined: it is in the trusted computing base.

## Verifying a message token (full chain)

Given a message token, the operator public key(s) you trust, and the received
payload bytes:

1. Verify the message token per above: `type == "message"`, `iss == sub`,
   `sub` is a user-role nkey.
2. Obtain the chain: `valiss.chain` embedded in the token, or supplied out of
   band. If both exist they must match byte-for-byte.
3. Verify the chain account token: `type == "account"`, signed by a trusted
   operator key, `sub` an account-role nkey.
4. Verify the chain user token: `type == "user"`, `iss` equal to the account
   token's `sub`, `sub` a user-role nkey.
5. Require the chain user token's `sub` to equal the message token's `iss`:
   the chain must delegate to exactly the key that signed the message.
6. Require all three tokens to agree on `valiss.epoch` (absent = 0). If you
   hold the domain's self-signed operator token, additionally require the
   message epoch to equal its epoch and the operator token to be within its
   own validity window.
7. Check all three validity windows at the verification instant. For stored
   messages, that is the instant of receipt, not now.
8. If the token carries `aud`, require it to equal the destination you are
   verifying for. Reject tokens without `aud` if you expect one.
9. If you require payload binding, hash the payload exactly as received and
   require `valiss.checksum` to equal its lowercase-hex SHA-256, rejecting a
   token that carries no checksum. Checksum verification is receiver-opt-in (the
   Go transports gate it behind `WithPayload`/`RequireChecksum`); a receiver that
   does not ask for it leaves the claim unchecked.

A verified message token proves origin only. It is not a credential: grant
nothing for possession of one. Offline receivers hold no allowlist; an online
receiver that wants revocation checks the chain account token's `jti` against
its own allowlist. The allowlist is a predicate over the `jti`, not a fixed
data structure, so a port can back it with a file, a database, or a live policy
call; see [Allowlist](../concepts/allowlist.md) for the shape of that seam.

## Verifying a request signature

Per-request proof of possession, separate from tokens. This is the mechanism
of the credential transports (`contrib/httpauth`, `contrib/grpcauth`); the
message-token transports (`contrib/httpsig`, `contrib/grpcsig`) carry no
request signature, since their binding lives inside the message token
(audience and checksum, above).

The client sends a timestamp (RFC 3339 with nanoseconds, UTC) and a signature
(base64 standard encoding, unlike the token's base64url). The signed payload
is:

```
"valiss-req-v1\n" + timestamp + "\n" + lowercase_hex(sha256(request_context))
```

The `valiss-req-v1\n` prefix is the version tag, bound into the signed bytes.
A verifier that reconstructs v1 bytes fails closed on any other version.

`request_context` is the transport's canonical request description:

- HTTP: `"http\n" + method + "\n" + host + "\n" + path + "\n" + nonce`
- gRPC: `"grpc\n" + full_method + "\n" + nonce`

The nonce is empty when replay suppression is off. Whether to suppress replay
at all is left to the implementation, but the behavior is pinned once you do: a
signed request must then carry a nonce, a nonce seen before is rejected, and an
entry is retained for `2 * skew` (the longest a replay could still land inside a
valid timestamp window). Verify the Ed25519
signature against the effective subject key (the user token's `sub` when a
chain is present, else the account token's `sub`) and bound the timestamp to a
skew window around now.
