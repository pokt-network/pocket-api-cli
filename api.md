# HTTP API reference

This page is for developers who call the Pocket API Console's account API
directly, without the `budgetkitctl` CLI or the web console. It covers how to
authenticate, every route a customer can call, the error codes, how relays are
billed and refused, and how to pay.

Notation: `<PLACEHOLDER>` is a value you supply. Response examples are labelled
**shape**: they show field names and types, not real values.

---

## Where the API lives

| what | production | beta |
|---|---|---|
| The account API (every `/v1/...` route on this page) | `https://pay.api.pocket.network` | `https://pay.beta.api.pocket.network` |
| Paid relays for a service | `https://<service>.api.pocket.network` | `https://<service>.beta.api.pocket.network` |
| Free relays for a service (no token, rate limited) | `https://<service>.free.api.pocket.network` | `https://<service>.free.beta.api.pocket.network` |
| Web console | `https://console.api.pocket.network` | `https://console.beta.api.pocket.network` |

The examples below use the production names; on beta, add `beta.` before `api`.

Only the `pay.` host serves `/v1/`. A `/v1/` path sent to a service host is not
an API call: it is relayed to that service like any other path. On the `pay.`
host, a path outside `/v1/` answers a plain-text `404`, and a `/v1/` route that
is not for customers answers `404` with api code 78.

All money in this API is an **integer number of micro-USD** (1 USD =
1,000,000 micro-USD). Field names that end in `_micro_usd` carry it.

---

## Authentication

A request is authenticated in one of three ways, depending on the route:

| kind | used by | what you send |
|---|---|---|
| **none** | `GET /v1/prices`, `GET /v1/payments/options`, `GET /v1/payments/quote`, `POST /v1/sessions` | nothing |
| **signed** | every other account route | an RFC 9421 HTTP Message Signature made with your account key (below) |
| **session** | the same routes as **signed**, from a browser | a `Budget-Session` header carrying a token from `POST /v1/sessions` |

### Your account is a key

Your account identifier (the **uid**) is a **compressed secp256k1 public key**,
written as 66 hex characters (33 bytes; the first byte is `02` or `03`). For
example: `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`.

- You generate the key pair yourself. The private key never leaves your machine
  and there is no registration call: the account exists as soon as you sign
  something with the key.
- The uid is case-insensitive; the server always writes it in lowercase.
- secp256k1 is the curve Pocket (a Cosmos chain) and EVM chains both use, so the
  same key also has a Pocket address (`pokt1...`) and an EVM address (`0x...`).
  That is what lets the server tell that a payment came from you
  ([Paying](#paying)).

### Signed requests (RFC 9421)

The scheme is RFC 9421 (HTTP Message Signatures) with RFC 9530
(`Content-Digest`), restricted to one fixed profile named `budgetkit-v1`.

**Covered components** are always exactly these three, in this order:
`"@method"`, `"@path"`, `"content-digest"`. The server does not read the list
you send; it rebuilds the signature base from its own fixed list, so a signature
over any other set does not verify.

**Signature parameters** (all four required, no others accepted):

| parameter | format | meaning |
|---|---|---|
| `created` | integer, Unix seconds | when you signed. Accepted when `abs(server_now - created) <= 300` |
| `keyid` | quoted; your uid, `^0[23][0-9a-fA-F]{64}$` | who is signing. This is the account the request acts on |
| `nonce` | quoted; 16 random bytes as 32 **lowercase** hex characters | single use. A nonce already seen is refused |
| `tag` | quoted; `budgetkit-v1:<target>` | which deployment the request is for. A request signed for another deployment is refused (api code 5) |

The `<target>` is the deployment's target name: `beta` on beta, and the
production name is published when production opens. Its format is `^[a-z0-9][a-z0-9_-]{0,63}$`.

An `alg` parameter, any other parameter, a repeated parameter, a label other
than `sig1`, or more than one signature is refused (api codes 10 to 34).

**Headers you send:**

| header | value |
|---|---|
| `Content-Digest` | `sha-256=:<base64(sha256(body))>:`. For an empty body this is `sha-256=:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=:`. The server computes the digest from the body it receives and does not read this header, but sending it is recommended |
| `Signature-Input` | `sig1=("@method" "@path" "content-digest");created=<created>;keyid="<uid>";nonce="<nonce>";tag="budgetkit-v1:<target>"` |
| `Signature` | `sig1=:<base64 of the 64-byte signature>:` |

#### How to sign a request, step by step

1. Fix the exact body bytes you will send. For a request with no body the body
   is empty (zero bytes). Do not re-serialise the JSON after signing.
2. Compute the digest: `sha-256=:` + standard base64 (with padding) of
   `sha256(body)` + `:`.
3. Choose `created` (current Unix time in seconds) and a fresh `nonce`
   (16 random bytes, lowercase hex).
4. Build the parameters string, exactly:
   ```
   ("@method" "@path" "content-digest");created=<created>;keyid="<uid>";nonce="<nonce>";tag="budgetkit-v1:<target>"
   ```
5. Build the signature base: four lines joined by a single `\n`, **no trailing
   newline**:
   ```
   "@method": <METHOD>
   "@path": <PATH>
   "content-digest": <DIGEST>
   "@signature-params": <PARAMETERS>
   ```
   `<METHOD>` is upper case (`GET`, `POST`, `DELETE`). `<PATH>` is the URL path
   only, **without the query string** (for `GET /v1/events?limit=10` the path is
   `/v1/events`). The query string is not covered by the signature.
6. Sign: ECDSA on secp256k1 over `sha256(base)`, with deterministic nonces
   (RFC 6979). Encode the signature as the 64-byte `r || s` (32 bytes each,
   big-endian, no recovery byte, not DER), then standard base64.
7. Send the three headers with the request. `keyid` may be written in either
   case, but it must be byte for byte what you put into the signature base.

**Replay and retry rules**

- Each nonce is accepted once. The server keeps spent nonces for 600 seconds
  (twice the window). Re-sending the same signed request is refused with
  `401` (api code 4).
- If an answer is lost (timeout, dropped connection), the request may have been
  processed. Sign the request again with a **new nonce** before retrying. A
  token mint is not idempotent: list your tokens to check before retrying one.
- A body larger than 1 MiB is refused with `413` (api code 1).
- If your clock is more than 300 seconds off, every signature is refused with
  `401` (api code 4). Keep the clock synchronised.

When a signature does not verify, the answer does not say which check failed
(unknown key, bad signature, stale `created` and spent nonce all answer api code
4). A malformed header is told precisely (codes 10 to 34).

#### Test vectors

Use them to check your implementation byte for byte.

- private key (hex): `0000000000000000000000000000000000000000000000000000000000000001`
- uid: `0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798`
- request: `GET /v1/balance`, empty body
- `created`: `1752674625`, `nonce`: `00112233445566778899aabbccddeeff`, target: `local`

Signature base (shown with real line breaks; there is no newline after the last line):

```
"@method": GET
"@path": /v1/balance
"content-digest": sha-256=:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=:
"@signature-params": ("@method" "@path" "content-digest");created=1752674625;keyid="0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798";nonce="00112233445566778899aabbccddeeff";tag="budgetkit-v1:local"
```

Headers:

```
Content-Digest: sha-256=:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=:
Signature-Input: sig1=("@method" "@path" "content-digest");created=1752674625;keyid="0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798";nonce="00112233445566778899aabbccddeeff";tag="budgetkit-v1:local"
Signature: sig1=:82o6gDDINUqhA+aHJILeYrDE44UUttzZfjdw3/Rut4Es8TK0irYoarM/zCF8d8IByqA/8kexv0iSII4Cqukl1w==:
```

A second vector, `POST /v1/tokens` with body `{"name":"ci"}`, nonce
`ffeeddccbbaa99887766554433221100`, same key, `created` and target:

```
Content-Digest: sha-256=:64sPDUIfaJqvKyaOpU4LASgwwIQrYcjLxIIPhOXCnY4=:
Signature: sig1=:unKQ8pflLs0caIBr5HQzVuyufvbPP8HHh0eEdX3p1OhC5R0LkBjiPecoqTFglm3t/SF+UovhbWWNNDNdYEB4Yg==:
```

secp256k1 is not one of the algorithms registered in RFC 9421, so a generic
RFC 9421 library cannot produce these signatures without being given a
secp256k1 signer.

#### A minimal Go signer

This program depends only on the public
`github.com/decred/dcrd/dcrec/secp256k1/v4` module. It reproduces both vectors
above. It prints the three headers, one per line, ready for `curl -H @file`.

```go
// bksign prints the three headers that sign one budgetkit request,
// or with -keygen writes a new key file and prints its uid.
package main

import (
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"encoding/hex"
	"flag"
	"fmt"
	"os"
	"strings"
	"time"

	"github.com/decred/dcrd/dcrec/secp256k1/v4"
	"github.com/decred/dcrd/dcrec/secp256k1/v4/ecdsa"
)

func main() {
	keygen := flag.String("keygen", "", "write a new key to this file and print its uid, then exit")
	keyPath := flag.String("key", "", "key file: 32 bytes as hex")
	target := flag.String("target", os.Getenv("BUDGETKIT_TARGET"), "deployment name")
	method := flag.String("method", "GET", "HTTP method")
	path := flag.String("path", "", "request path WITHOUT the query string, e.g. /v1/balance")
	body := flag.String("body", "", "exact request body bytes (empty for GET)")
	created := flag.Int64("created", 0, "unix seconds (0 = now; set only to reproduce a test vector)")
	nonce := flag.String("nonce", "", "32 lowercase hex chars (empty = random; set only to reproduce a test vector)")
	flag.Parse()

	if *keygen != "" {
		priv, err := secp256k1.GeneratePrivateKey()
		must(err)
		must(os.WriteFile(*keygen, []byte(hex.EncodeToString(priv.Serialize())+"\n"), 0o600))
		fmt.Println(hex.EncodeToString(priv.PubKey().SerializeCompressed()))
		return
	}
	raw, err := os.ReadFile(*keyPath)
	must(err)
	kb, err := hex.DecodeString(strings.TrimSpace(string(raw)))
	if err != nil || len(kb) != 32 {
		must(fmt.Errorf("key file: want 32 bytes of hex"))
	}
	priv := secp256k1.PrivKeyFromBytes(kb)
	uid := hex.EncodeToString(priv.PubKey().SerializeCompressed())
	if *target == "" || *path == "" {
		must(fmt.Errorf("-target and -path are required"))
	}
	if *created == 0 {
		*created = time.Now().Unix()
	}
	if *nonce == "" {
		var n [16]byte
		_, err := rand.Read(n[:])
		must(err)
		*nonce = hex.EncodeToString(n[:])
	}
	sum := sha256.Sum256([]byte(*body))
	digest := "sha-256=:" + base64.StdEncoding.EncodeToString(sum[:]) + ":"
	params := fmt.Sprintf(`("@method" "@path" "content-digest");created=%d;keyid="%s";nonce="%s";tag="budgetkit-v1:%s"`,
		*created, uid, *nonce, *target)
	base := `"@method": ` + *method + "\n" +
		`"@path": ` + *path + "\n" +
		`"content-digest": ` + digest + "\n" +
		`"@signature-params": ` + params
	h := sha256.Sum256([]byte(base))
	compact := ecdsa.SignCompact(priv, h[:], true) // [recovery byte | r | s]
	fmt.Println("Content-Digest: " + digest)
	fmt.Println("Signature-Input: sig1=" + params)
	fmt.Println("Signature: sig1=:" + base64.StdEncoding.EncodeToString(compact[1:]) + ":")
}

func must(err error) {
	if err != nil {
		fmt.Fprintln(os.Stderr, "bksign:", err)
		os.Exit(1)
	}
}
```

To build it: put it in an empty directory as `main.go`, then run
`go mod init bksign && go get github.com/decred/dcrd/dcrec/secp256k1/v4@v4.4.1 && go build -o bksign .`
(tested with Go 1.27).

The CLI command `budgetkitctl tools sign` prints the same three headers without
any code ([cli.md](cli.md)).

### Browser sessions

A browser wallet cannot compute the signature above, so the web console signs
in once and receives a session token instead. You can use the same flow from
code.

**`POST /v1/sessions`**: no authentication; the body is the credential.

Request body (all strings):

| field | required | value |
|---|---|---|
| `scheme` | yes | `pocket-raw` (secp256k1 over `sha256(utf8(message))`, as Soothe Vault signs on Pocket), `adr36` (the ADR-36 sign document wrapping the message, as Cosmos wallets sign), or `eip191` (an EVM wallet's `personal_sign`) |
| `message` | yes | the sign-in text, exactly as signed (format below) |
| `signature` | yes | `pocket-raw`: hex of the 64-byte compact signature (65 bytes with a trailing recovery byte is also accepted). `adr36`: base64 of 64 bytes. `eip191`: `0x`-hex of the 65-byte `r || s || v` |
| `public_key` | for `pocket-raw` and `adr36` | the compressed secp256k1 public key: hex for `pocket-raw`, base64 for `adr36`. Omit it for `eip191` (the key is recovered) |

The sign-in text is 10 lines, or 11 with the optional last line, joined by `\n`:

```
<console-domain> wants you to sign in with your Pocket account:
<your pokt1... or 0x... address>

Sign in to the Pocket Network console

URI: https://<console-domain>
Version: 1
Chain ID: <chain id>
Nonce: <32 lowercase hex characters>
Issued At: <UTC time, e.g. 2026-11-04T14:47:00.000Z>
Expiration Time: <UTC time>
```

The server refuses the proof unless:

- `<console-domain>` is this deployment's console domain
  (`console.api.pocket.network`, or `console.beta.api.pocket.network` on beta);
- `Chain ID` is the `chain_id` of one of the deployment's payment options
  ([`GET /v1/payments/options`](#get-v1paymentsoptions-where-and-how-to-pay)):
  on beta, `pocket-lego-testnet` or `84532`;
- `Issued At` is within 300 seconds of the server's clock and `Expiration Time`,
  when present, has not passed;
- the nonce has not been used before;
- the signature is valid and the address on line 2 is the address of the
  signing key (bech32 `pokt...` or an EVM address, either case).

The console always sends the statement `Sign in to the Pocket Network console`
on line 4. Signing this text is not a transaction: nothing goes on chain and no
fee is paid. The number of sign-ins from one client IP address is limited; over
the limit the answer is `429` (api code 97) with `Retry-After`.

Response `200` (**shape**):

```json
{ "token": "v1.<claims>.<signature>", "uid": "<uid>", "expires_at": "<RFC 3339 time>" }
```

A session lasts 8 hours unless the deployment sets another length (between 10
minutes and 7 days). Send it as `Budget-Session: <token>` on any **signed**
route below; it works on no other route. If a request carries a
`Signature-Input` or `Signature` header, it is judged as a signed request and
the session header is ignored.

**Minting and revoking a token under a session need a fresh wallet proof.** A
token outlives the session, so `POST /v1/tokens` and `DELETE /v1/tokens/{id}`
under a session carry a `proof` object in their body: the same four fields as
`POST /v1/sessions`, over a new sign-in text whose statement line is
`Mint the API token "<name>"` or `Revoke the API token "<id>"`, signed by the
session's own key. A missing or refused proof is `403` (api codes 95 and 96);
the session itself stays valid. A signed request needs no proof.

**`DELETE /v1/sessions/current`**: authenticated by the `Budget-Session` header
only (a signature is not accepted). Ends that session and answers `204` with no
body. A session that has already ended is refused with `401` (api code 92).

**`GET /v1/sessions`**: signed. Lists the account's live sessions:
`{ "sessions": [ { "expires_at": "<time>", "current": true } ] }` (**shape**).

**`DELETE /v1/sessions`**: signed. Ends every session of the account, including
the console's, and answers `{ "revoked": <n> }`. A key signature always reaches
this route, so it closes a session somebody else opened.

Session errors: api codes 88 to 97 ([Errors](#errors)).

---

## Routes

Every route below is called on the `pay.` host. "Signed" means signed or with a
session. The account a signed route acts on is always **the signer**; no route
takes a uid.

### Tokens

API tokens are the bearer credentials your applications send with relays
([Relays](#relays)). A token is `pkt_` followed by up to 43 base62 characters.
Its plaintext is returned **once**, by the mint call; the server stores only its
SHA-256 hash. A token's id is `tok_` followed by 16 hex characters.

#### `POST /v1/tokens`: mint a token

- Auth: signed. Under a session, also a `proof` ([Browser sessions](#browser-sessions)).
- Body (optional; `{}` or an empty body is fine):

| field | type | meaning |
|---|---|---|
| `name` | string, up to 128 characters | your own label; not interpreted, not unique |
| `expires_at` | string, RFC 3339 time | when the token stops working. Omit for a token that never expires |

- Response `201` (**shape**):

```json
{ "id": "tok_<hex>", "token": "pkt_<...>", "uid": "<uid>", "name": "<name>", "expires_at": "<RFC 3339 time or null>" }
```

- Errors: `409` api 79 when the account already holds its maximum of live
  tokens (100 unless the deployment sets another number; revoke one to mint
  another), `400` api 3 or 45 for a bad body, `500` api 46.

```sh
BODY='{"name":"backend"}'
curl -sS -X POST "https://pay.api.pocket.network/v1/tokens" \
  -H 'Content-Type: application/json' \
  -H @headers.txt \
  --data-binary "$BODY"
```

`headers.txt` holds the three signature headers for `POST /v1/tokens` and this
exact `BODY`.

#### `GET /v1/tokens`: list tokens

- Auth: signed. No body.
- Response `200`: a JSON **array** (**shape**):

```json
[ { "id": "tok_<hex>", "name": "<name>", "created_at": "<time>", "expires_at": "<time>", "revoked_at": "<time>" } ]
```

`name`, `expires_at` and `revoked_at` are omitted when empty. It never contains
a token's plaintext or hash.

- Errors: `500` api 47.

#### `DELETE /v1/tokens/{id}`: revoke a token

- Auth: signed, with no body. Under a session, the body carries a `proof`.
  `{id}` is the `tok_...` id.
- Response `204`, no body. Relays carrying the token are refused once the
  revocation reaches the relay hosts, normally within seconds.
- Errors: `404` api 49 for an id that is unknown or not yours (the two are not
  told apart), `500` api 48.

### Account

#### `GET /v1/balance`: your balance and tier

- Auth: signed. No body.
- Response `200` (**shape**):

```json
{
  "uid": "<uid>",
  "trial_micro_usd": 0,
  "plan_micro_usd": 0,
  "credit_micro_usd": 0,
  "spendable_micro_usd": 0,
  "tier": "trial | paid | exhausted",
  "version": 0,
  "low_services": ["<service>"],
  "has_plan": false,
  "next_trial_reset_at": "<time, optional>",
  "next_plan_refill_at": "<time, optional>"
}
```

| field | meaning |
|---|---|
| `trial_micro_usd` | the daily trial pool; refills at `next_trial_reset_at` |
| `plan_micro_usd` | the plan pool; refills at `next_plan_refill_at` while `has_plan` is true |
| `credit_micro_usd` | money you paid in; never expires |
| `spendable_micro_usd` | the sum of the three pools |
| `tier` | `trial`, `paid` (plan or credit above zero) or `exhausted` (nothing left; relays are refused with `402`) |
| `low_services` | services for which your balance is below the low-balance warning threshold |
| `has_plan` | whether an active plan refills the plan pool. Plan money with `has_plan: false` is left over from a cancelled plan |
| `version` | an increasing counter of changes to the account |

A key that has never been used answers with the starting state: the daily trial
quota, if the deployment grants one.

- Errors: `503` api 50.

#### `GET /v1/usage`: what you spent it on

- Auth: signed. No body.
- Response `200`: `{ "uid": "<uid>", "rollups": [ ... ] }`, one row per service,
  hour and price (**shape**):

```json
{
  "service": "<service>",
  "hour": "<time>",
  "count": 0,
  "charges": 0,
  "cost_micro_usd": 0,
  "drained_trial_micro_usd": 0,
  "drained_plan_micro_usd": 0,
  "drained_credit_micro_usd": 0,
  "forgiven_micro_usd": 0,
  "cu_per_relay": 0,
  "cuttm": 0,
  "granularity": 0,
  "rate_micro_usd": 0,
  "markup_bps": 0,
  "rate_source": "<source>",
  "rate_level": "<level>",
  "quoted_at": "<time>"
}
```

`service` is the service's Pocket service id, which can differ from the name in
the relay address (on beta, `anvil` is billed as `pnf-anvil` and `pocket` as
`pnf-pocket-beta`). `count` is relays served; `cost_micro_usd` is what they
cost, and the three `drained_*` fields say which pool paid. The forgiven amount
is served traffic no pool could cover and that was not charged. The remaining
fields are the inputs the cost was computed from, so a row can be recomputed on
its own. Costs are whole micro-USD: each charge is the floor of the running
exact total, and the fraction is carried to the next charge rather than lost.

- Errors: `500` api 69.

#### `GET /v1/events`: account history

- Auth: signed. No body.
- Query: `limit` (non-negative integer; 0 or absent means 100; a larger value
  than 1000 is treated as 1000), `after` (the cursor from the previous page).
  The query is not signed: sign the path `/v1/events` only.
- Response `200`, newest first: `{ "uid": "<uid>", "events": [ ... ], "after": "<cursor>" }`.
  `after` is present only when there are more events. Event row (**shape**):

```json
{
  "kind": "genesis | reset | plan_grant | plan_reset | plan_cancel | refund | credit",
  "at": "<time>",
  "pool_micro_usd": 0,
  "prev_pool_micro_usd": 0,
  "version": 0,
  "actor": "<uid of the operator, or empty for automatic events>",
  "reference": "<optional>",
  "funding": "<optional: granted | sold>",
  "pool": "<optional: trial | plan | credit>",
  "reason": "<optional>",
  "asset": "<credit only>",
  "paid_micro_usd": 0,
  "bonus_bps": 0
}
```

| kind | meaning |
|---|---|
| `genesis` | the account was created and the trial granted |
| `reset` | the daily trial refilled |
| `plan_grant`, `plan_reset`, `plan_cancel` | a plan started, refilled, was cancelled |
| `refund` | an operator paid money back into a pool |
| `credit` | a payment was credited. `paid_micro_usd` is what the coins were worth and `bonus_bps` the bonus applied |

`pool_micro_usd` and `prev_pool_micro_usd` are the affected pool's value after
and before the event.

- Errors: `400` api 66 (bad `limit`), `400` api 67 (bad `after`), `500` api 68.

```sh
curl -sS "https://pay.api.pocket.network/v1/events?limit=50" -H @headers.txt
```

#### `GET /v1/prices`: what a relay costs

- Auth: none. No body. Every caller receives the same answer.
- Response `200` (**shape**):

```json
{
  "version": "<integer as string>",
  "markup_bps": 0,
  "settlement": "<chain/asset>",
  "params": { "cuttm": 0, "granularity": 0, "source": "<source>", "level": "<level>", "block_height": 0, "observed_at": "<time>" },
  "services": [
    {
      "service": "<service id>",
      "cu_per_relay": 0,
      "price_per_relay_micro_usd": { "numerator": "<integer>", "denominator": "<integer>" },
      "price_per_relay_micro_usd_decimal": "<decimal string>",
      "decimal_is_exact": true
    }
  ],
  "assets": [ { "asset": "<chain/asset>", "rate_micro_usd": 0, "decimals": 0, "source": "<source>", "level": "<level>", "quoted_at": "<time>" } ]
}
```

`services` lists every service the chain prices, by its Pocket service id. That
is more than a deployment serves, and the id can differ from the name in the
relay address: on beta, the relay services `anvil` and `pocket` are priced as
`pnf-anvil` and `pnf-pocket-beta`.

The per-relay price is a fraction of micro-USD, given as strings because the
denominator can exceed what a JSON number holds exactly. `numerator: "0"` means
free on purpose; empty strings (and the decimal `"unpriceable"`) mean the
service cannot be priced. The price is
`cu_per_relay x cuttm / granularity` upokt, valued at the POKT entry of
`assets` (`rate_micro_usd` per whole coin of `decimals` places), plus
`markup_bps`.

- Errors: `503` api 65 (no price loaded yet).

```sh
curl -sS "https://pay.api.pocket.network/v1/prices"
```

### Payments

#### `GET /v1/payments/options`: where and how to pay

- Auth: none. No body.
- Response `200` (**shape**): `{ "chains": [ ... ] }`, one row per coin:

```json
{
  "chain": "<chain name>",
  "asset": "<coin>",
  "wallet": "<receiving address>",
  "contract": "<token contract; only for a token>",
  "decimals": 0,
  "bonus_bps": 0,
  "min_confirmations": 0,
  "kind": "evm",
  "chain_id": "<the chain's id>",
  "denom": "<cosmos only>",
  "public_rpc_url": "<optional>",
  "display_name": "<optional>",
  "rate_micro_usd": 0,
  "quoted_at": "<time>"
}
```

| field | meaning |
|---|---|
| `chain` | the name you pass as `chain` when you notify |
| `asset` | the coin |
| `wallet` | where to send it |
| `contract` | for a token (e.g. an ERC-20), its contract address; absent for a chain's native coin |
| `decimals` | base units per whole coin, as a power of ten (6 for POKT's `upokt`, 18 for ETH's wei) |
| `bonus_bps` | extra credit in basis points: `2000` means a payment is credited at 120% of its value |
| `min_confirmations` | blocks the payment must be buried under before it is credited |
| `kind` | `evm` or `cosmos`: which kind of chain, and so which wallet |
| `chain_id` | the chain the payment must be sent on, as it names itself: decimal for EVM (`"84532"`), a string for cosmos (`"pocket-lego-testnet"`). Check your wallet or RPC is on it before sending |
| `denom` | cosmos only: the denomination your wallet sends (`upokt`); absent for EVM |
| `public_rpc_url` | optional: an RPC for this chain a wallet may use; absent when none is published |
| `display_name` | optional: the chain's name for a wallet to show |
| `rate_micro_usd` | the coin's price in force, in micro-USD per whole coin |
| `quoted_at` | when that price was quoted |

Only coins the server can currently price are listed.

- Errors: `503` api 80 or 77.

```sh
curl -sS "https://pay.api.pocket.network/v1/payments/options"
```

#### `GET /v1/payments/quote`: what an amount is credited as

- Auth: none. No body.
- Query: `chain` and `asset` (from the options), and exactly one of `amount`
  (a positive integer of base units) or `usd` (a positive decimal of at most 6
  places).
- Response `200` (**shape**):

```json
{ "chain": "<chain>", "asset": "<coin>", "amount": "<base units>", "decimals": 0,
  "amount_micro_usd": 0, "bonus_bps": 0, "credited_micro_usd": 0, "rate_micro_usd": 0, "quoted_at": "<time>" }
```

A quote is an estimate: a payment is valued at the price in force when it is
credited.

- Errors: `400` api 98 (bad query), `404` api 99 (that coin is not offered on
  that chain right now), `422` api 100 (the amount cannot be quoted exactly),
  `503` api 77 or 80.

```sh
curl -sS "https://pay.api.pocket.network/v1/payments/quote?chain=<CHAIN>&asset=<ASSET>&usd=10"
```

#### `POST /v1/payments/notify`: tell the server you paid

- Auth: signed. The payment is credited to **the signer**, and only if the
  chain shows the money was sent **from the signer's own address** on that
  chain ([Paying](#paying)).
- Body:

| field | required | value |
|---|---|---|
| `chain` | yes | a `chain` name from the options |
| `tx_hash` | one of the two | the transaction hash. EVM: `0x` + 64 hex. Pocket: 64 hex |
| `raw_tx` | one of the two | a signed transaction for the server to broadcast. EVM: hex (with or without `0x`). Pocket: standard base64 of the `TxRaw` bytes; it must include the signer's public key |
| `uid` | no; do **not** send it | refused for customers |

Sending both `tx_hash` and `raw_tx` is allowed; the hash must then match the
bytes.

- Response `200`: a **receipt** (shape below). Notifying the same transaction
  again is harmless and returns the current receipt.
- Errors: `400` with a payments code ([payments codespace](#payments-codespace)),
  `503` payments code 13 (send the same notice again), `400` api 3 (the body
  does not match the schema, e.g. neither hash nor raw).

Receipt (**shape**; the two read routes below return the same object):

```json
{
  "uid": "<uid>",
  "chain": "<chain>",
  "tx_hash": "<hash>",
  "notified_at": "<time>",
  "status": "to_deliver | verifying | waiting_confirmations | confirmed | credited | rejected | reorged | not_found",
  "reason": "<constant, only when there is one>",
  "reason_code": 0,
  "reason_message": "<fixed sentence>",
  "coins": [
    { "asset": "<coin>", "amount_asset": "<base units, string>", "credited_micro_usd": 0, "amount_micro_usd": 0, "bonus_bps": 0 }
  ]
}
```

| status | meaning | final |
|---|---|---|
| `to_deliver` | you sent `raw_tx`; the server has not broadcast it yet | no |
| `verifying` | the server is looking for the transaction on chain | no |
| `waiting_confirmations` | found in a block, paying us; waiting for `min_confirmations` | no |
| `confirmed` | enough confirmations; the credit is about to be written | no |
| `credited` | the credit landed in your account | **yes, success** |
| `rejected` | not a valid payment to us; see `reason` | yes, except the reasons a new notice reopens ([billing-rules.md](billing-rules.md)) |
| `reorged` | its block disappeared and the transaction did not come back | until notified again: a new notice reopens it |
| `not_found` | the chain never showed it within 24 hours of the notice (unless the deployment sets another time) | until notified again: a new notice reopens it |

`coins` appears once the chain has been read. `credited_micro_usd`,
`amount_micro_usd` and `bonus_bps` are zero until the coin is credited; then
`credited_micro_usd - amount_micro_usd` is exactly the bonus you earned.
`reason`, `reason_code` and `reason_message` use the payments codespace. If
somebody else's key sent the transaction, your receipt says `rejected` with
reason `not_the_sender`: a payment is credited only to whoever sent it.

```sh
BODY='{"chain":"<CHAIN>","tx_hash":"<TX-HASH>"}'
curl -sS -X POST "https://pay.api.pocket.network/v1/payments/notify" \
  -H 'Content-Type: application/json' \
  -H @headers.txt \
  --data-binary "$BODY"
```

#### `GET /v1/payments`: payments you notified

- Auth: signed. No body.
- Query: `limit` (non-negative integer; default 100; a larger value than 1000 is
  treated as 1000), `after` (the opaque cursor from the previous page; do not
  build one). Sign the path `/v1/payments` only.
- Response `200`: `{ "payments": [ <receipt>, ... ], "after": "<cursor>" }`,
  newest notice first; `after` is present only when there is more.
- Errors: `400` api 66, `400` api 67, `503` api 75, `503` api 77.

#### `GET /v1/payments/{chain}/{tx_hash}`: one payment

- Auth: signed. No body.
- Response `200`: a receipt. `404` api 76 when this account never notified that
  transaction (also when somebody else did: other accounts' payments are not
  disclosed).
- Errors: `503` api 75, `503` api 77.

---

## Response conventions

- Successful responses are JSON with `Content-Type: application/json`, except
  `204` (no body).
- Times are RFC 3339 strings in UTC.
- Fields documented as optional are omitted, not sent as `null`, except
  `expires_at` in the mint response.

---

## Errors

Every refusal from the account API has a JSON body with three fields:

```json
{ "codespace": "api", "code": 4, "message": "unauthorized" }
```

Branch on `codespace` and `code`, which never change meaning. The `message` is a
fixed sentence per code. There are two codespaces: `api` and `payments`.

### `api` codespace

Codes a customer can receive:

| code | HTTP | message | what to do |
|---|---|---|---|
| 1 | 413 | request entity too large: the body is larger than this API carries | send a body under 1 MiB |
| 2 | 400 | the request body could not be read | resend |
| 3 | 400 | the request body does not match this route's schema | fix the body (unknown field, missing required field, wrong type) |
| 4 | 401 | unauthorized | check your clock, sign again with a new nonce, sign the exact body and the path without the query |
| 5 | 401 | unauthorized: this request is signed for another deployment | set the right target |
| 6 | 503 | service unavailable: the replay gate cannot be consulted | retry later with a new signature |
| 7 | 502 | the control plane is not answering | retry later |
| 8 | 503 | service unavailable: the control plane's transport has not reported how large a message it carries | retry later |
| 9 | 500 | internal error | retry later |
| 10 to 34 | 401 | malformed signature header: ... (the message names the part) | fix the header format |
| 35 | 502 | the control plane gave an answer that is not an HTTP status | retry later |
| 41 | 402 | payment required (relays) | see [Relays](#relays) |
| 45 | 400 | malformed body | fix the mint body |
| 46 | 500 | the token could not be minted | list tokens, then retry |
| 47 | 500 | the tokens could not be listed | retry |
| 48 | 500 | the token could not be revoked | retry |
| 49 | 404 | token not found | check the id; it is unknown or not yours |
| 50 | 503 | service unavailable: the balance could not be read | retry |
| 65 | 503 | no price snapshot loaded: budgetd cannot price a relay yet | retry later |
| 66 | 400 | limit must be a non-negative integer | fix the query |
| 67 | 400 | after must be a positive integer cursor from a previous page | use the cursor from the previous page |
| 68 | 500 | history unavailable | retry |
| 69 | 500 | usage unavailable | retry |
| 75 | 503 | service unavailable: the payment record could not be read | retry |
| 76 | 404 | unknown payment | this account never notified it: notify it first, or check chain and hash |
| 77 | 503 | service unavailable: payments are not configured | this deployment takes no payments |
| 78 | 404 | not found: no such route | check the path and the host |
| 79 | 409 | conflict: this account already holds its maximum of live tokens; revoke one to mint another | revoke one, then mint |
| 80 | 503 | service unavailable: the payment options could not be read | retry |
| 83 | 503 | service unavailable: the admin roster in force could not be read | only reached when a notify body carries `uid`; remove it |
| 84 | 502 | bad gateway: the upstream did not answer | retry (WebSocket) |
| 85 | 502 | bad gateway: the upstream refused the WebSocket upgrade | retry later |
| 86 | 503 | service unavailable: the module is closing and takes no new session | retry; another replica answers |
| 88 | 400 | bad request: the sign-in proof is malformed | rebuild the message and body |
| 89 | 400 | bad request: the sign-in message names another site | use the deployment's console domain |
| 90 | 400 | bad request: the sign-in message names a chain this deployment has no rail on | use a `chain_id` from the payment options |
| 91 | 401 | unauthorized: the sign-in proof was refused | build a new message with a fresh nonce and time, sign again |
| 92 | 401 | unauthorized: the session is not valid | sign in again |
| 93 | 503 | service unavailable: this deployment issues no browser sessions | use signed requests |
| 94 | 503 | service unavailable: the session store could not answer | retry |
| 95 | 403 | forbidden: this action from a browser session needs a fresh signature from the wallet | add the `proof` ([Browser sessions](#browser-sessions)) |
| 96 | 403 | forbidden: the wallet's proof for this action was refused | sign a new proof |
| 97 | 429 | too many requests: too many sign-ins from this address; try again after Retry-After seconds | wait `Retry-After` seconds |
| 98 | 400 | bad request: a quote takes chain, asset and exactly one of amount (a positive integer of base units) or usd (a positive decimal of at most 6 places) | fix the query |
| 99 | 404 | not found: that coin is not offered on that chain right now | use a row of the payment options |
| 100 | 422 | unprocessable: that amount cannot be quoted exactly: it is past what can be represented, or no amount of the coin is worth exactly those dollars | quote another amount |

### payments codespace

Returned by `POST /v1/payments/notify` as the error body (`400`, or `503` for
code 13), and used in a receipt's `reason_code` and `reason_message` when a
payment is rejected. Codes 2 and 6 are not used.

| code | HTTP | message |
|---|---|---|
| 1 | 400 | malformed body: need chain and tx_hash or raw_tx |
| 3 | 400 | the uid comes from the signature; remove it from the body |
| 4 | 400 | the chain is not one payments are accepted on |
| 5 | 400 | the signing key derives no address on this chain |
| 7 | 400 | tx_hash is not a transaction hash this chain can have |
| 8 | 400 | raw_tx is not a signed transaction this chain can read |
| 9 | 400 | raw_tx is not a payment: it has to send the chain's coin to our wallet, or call a token's transfer to it |
| 10 | 400 | raw_tx pays an address that is not our wallet |
| 11 | 400 | raw_tx moves a coin this chain does not accept |
| 12 | 400 | raw_tx is not signed by the notifying account: a payment is notified by whoever pays it |
| 13 | 503 | service unavailable: the payment could not be registered; send the same notice again |
| 14 | 400 | raw_tx spends from more than one account; a payment has one payer |
| 24 | 400 | tx_hash does not match raw_tx: send one, or the hash of the bytes you send |
| 26 | 400 | raw_tx carries no public key for its signer, so it cannot be checked before it is sent; include the signer's public key |

These appear only in receipts, with the matching `reason` constant:

| code | `reason` | message |
|---|---|---|
| 15 | `not_the_sender` | the payment was sent by another account, and is credited to whoever sent it |
| 16 | `broadcast_refused` | the node refused raw_tx; once the transaction is on chain, notify its tx_hash |
| 17 | `not_ours` | the transaction pays nothing to our wallet |
| 18 | `coin_not_accepted` | the transaction pays our wallet in a coin this network does not accept |
| 19 | `execution_failed` | the transaction failed on chain, so nothing was paid |
| 20 | `two_senders` | the transaction pays our wallet from more than one account; a payment has one payer |
| 21 | `no_price` | the payment could not be priced; notify it again later |
| 22 | `credit_zero` | the payment is worth less than the smallest amount credited, so nothing was credited |
| 23 | `credit_overflow` | the payment is worth more than one credit can hold, so nothing was credited; it can be returned ([billing-rules.md](billing-rules.md#returns)) |
| 25 | `raw_too_large` | the chain refused raw_tx as too large; once the transaction is on chain, notify its tx_hash |
| 27 | `unattributable` | no single account funded the coin that reached our wallet, so nothing was credited; it can be returned ([billing-rules.md](billing-rules.md#returns)) |

---

## Relays

Send your application's requests to `https://<service>.api.pocket.network`
with your token in the standard bearer header:

```
Authorization: Bearer pkt_<...>
```

The scheme name is case-insensitive. Any other `Authorization` scheme, or a
token anywhere else (a query parameter, another header), is treated as no
token. The request body and path are whatever the service speaks; for an EVM
JSON-RPC service such as `anvil` on beta:

```sh
curl -sS -X POST "https://anvil.beta.api.pocket.network/" \
  -H "Authorization: Bearer <TOKEN>" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

Which pool pays: the trial pool first, then the plan pool, then credit. While
plan or credit money remains you are on the `paid` tier; on trial money alone,
the `trial` tier. Each tier has its own rate limits, set by the deployment.

### Headers on a served relay

| header | meaning |
|---|---|
| `Pocket-Tier` | the tier the request was served at: `trial` or `paid` |
| `X-RateLimit-Limit` | your tier's limit for this service, per window |
| `X-RateLimit-Remaining` | how many requests are left in the window |
| `X-RateLimit-Reset` | when the window ends, as a Unix time in seconds |
| `X-Request-Id` | an id for this request; quote it when you report a problem |

The `X-RateLimit-*` headers are on served relays and on the `429` of your
tier's limit. A `402`, a `413` and the per-address `429` below do not carry
them. The free endpoint sends the same three headers on the requests it serves
and on its own `429`.

### What is billed

- **HTTP**: one relay per request that the service answered with a `2xx`, or
  with one of `400 404 405 406 411 413 414 415 416 417 422 431`. Other `4xx`
  answers, every `5xx`, and refusals by the relay host itself are not billed.
  For an EVM JSON-RPC batch, each element is one relay, billed when the batch is
  answered with a `2xx`.
- **WebSocket**: one relay per data message the service sends you. Control
  frames, and the messages you send, are not billed. The token goes in the
  `Authorization` header of the upgrade request, the one way it is read, so a
  browser's WebSocket API, which cannot set that header, cannot open a paid
  socket. When your balance reaches zero during a socket, the message that
  spent it is billed and the socket is closed. Falling from the paid tier to the
  trial tier does not close it.
- **gRPC**: one relay per call, billed when it ends with status `OK`,
  `INVALID_ARGUMENT`, `NOT_FOUND`, `FAILED_PRECONDITION` or `OUT_OF_RANGE`.
- **Streamed HTTP responses** (server-sent events): one relay per request.

### `402 Payment Required`

When your account has nothing left in any pool, or the request carries no
token of a known account, relays on the paid domain are refused with `402`
(never `429`: waiting does not help). The response says how to pay and where the
free endpoint is.

Headers:

```
WWW-Authenticate: budget options="https://pay.api.pocket.network/v1/payments/options"
Link: <https://pay.api.pocket.network/v1/payments/options>; rel="payment"
Link: <https://<service>.free.api.pocket.network>; rel="alternate"; title="free endpoint"
```

Body (**shape**; the text is in `error`, not `message`):

```json
{
  "codespace": "api",
  "code": 41,
  "error": "payment required: this account has no balance left",
  "scheme": "budget",
  "accepts": [
    {
      "scheme": "budget",
      "network": "<chain>",
      "asset": "<coin>",
      "contract": "<token contract, optional>",
      "pay_to": "<receiving wallet>",
      "extra": { "bonus_bps": 0, "min_confirmations": 0 }
    }
  ],
  "notify": "https://pay.api.pocket.network/v1/payments/notify",
  "free_endpoint": "https://<service>.free.api.pocket.network"
}
```

`error` is one of:

| `error` | meaning |
|---|---|
| `payment required: this account has no balance left` | pay, or use the free endpoint |
| `payment required: this request carries no token of a known account` | no token, an unknown token, or a revoked one. A newly minted token takes a moment to reach the relay hosts: retry after a few seconds |
| `payment required: this replica cannot bill a WebSocket, gRPC or streamed session yet; retry (transport_not_billable)` | the relay host has just started; retry |

`accepts` is omitted when the relay host does not know the payment options yet;
read `GET /v1/payments/options` then.

### Other relay refusals

| answer | meaning |
|---|---|
| `429 Too Many Requests` with `Retry-After: <seconds>`, body `throttled` | you are over your tier's rate. Wait that long |
| `429` with `Retry-After`, body `too many requests from this address` | too many requests from your IP address, whatever the account. Wait that long |
| `503` with `Retry-After: 1` | the rate limiter could not decide. Retry |
| `413` | the body or a JSON-RPC batch is larger than the deployment allows |
| `404` with `{"error":"unknown service","service":"<name>"}` | the service is not served on this domain |
| `421 Misdirected Request` with `{"error":"misdirected request: this process does not serve that Host"}` | the host name is not one of the deployment's domains |
| `502` api 84 or 85 | the service did not answer, or refused the WebSocket upgrade |

### The free endpoint

`https://<service>.free.api.pocket.network` serves relays without a token and
without charging, at its own rate limit, counted per client IP address. It
serves no `/v1/` routes. A feature the free endpoint does not serve on a
service is answered with a `402` whose `error` names the feature and says to use
the paid endpoint. On beta the free limit is 100 requests per minute per client
IP address, and WebSocket is not served.

---

## Paying

1. **Read the options**: `GET /v1/payments/options`. Pick a row; note `chain`,
   `wallet`, `asset`, `contract` (if any), `decimals`, `bonus_bps`,
   `min_confirmations` and `chain_id`.
2. **Send the coins from your own key's address.** The server credits a payment
   only if it came from the address derived from your uid on that chain: the
   Pocket address (`pokt1...`) or the EVM address (`0x...`) of the same key.
   A transfer from any other address is not credited to you.
   - Native coin: a plain transfer of the amount to `wallet`.
   - Token: a call to `contract`'s `transfer(address,uint256)` with `wallet` as
     the recipient. Not `transferFrom`, and no value attached.
   - Amounts are in base units: `amount x 10^decimals`.
3. **Notify**: `POST /v1/payments/notify` with `{"chain": "<chain>", "tx_hash": "<hash>"}`,
   signed with the same key. Or, instead of broadcasting yourself, send the
   signed transaction as `raw_tx` and the server broadcasts it.
4. **Poll** `GET /v1/payments/{chain}/{tx_hash}` until `status` is final.
   `credited` is success; then `GET /v1/balance` shows the new
   `credit_micro_usd` and the tier `paid`.

**How a payment is valued.** At the moment of crediting, each coin is valued at
the price in force for that coin, in micro-USD (rounded down), then the bonus is
added: `credited = value x (10000 + bonus_bps) / 10000`. The receipt shows both
`amount_micro_usd` (the value) and `credited_micro_usd` (with the bonus). The
bonus in force at crediting time applies; the options route always shows the
current ones. `GET /v1/payments/quote` estimates it before you pay.

**Confirmations.** A payment is credited after `min_confirmations` blocks.

**Time limit.** If the server cannot find the transaction on chain within 24
hours of the notice (unless the deployment sets another time), it becomes
`not_found`. Notifying the same transaction again reopens a `not_found` or
`reorged` payment.
