# The `budgetkitctl` CLI

`budgetkitctl` manages a Pocket API Console account from a terminal: it creates
your key, signs every request for you, and calls the account API. It is a single
binary with no runtime dependencies.

Commands are grouped by scope: `budgetkitctl <scope> <command> [flags]`. This
page covers the `user` scope (your own account) and the `tools` scope (key
generation and manual signing). The `admin` scope is for Pocket Network
operators and is not covered here.

---

## Install

Download the binary for your system from the release page,
https://github.com/pokt-network/pocket-api-cli/releases. `<version>` is the
release tag, for example `v0.1.0`.

| OS | architecture | file |
|---|---|---|
| Linux | x86-64 | `budgetkitctl_<version>_linux_amd64` |
| Linux | ARM64 | `budgetkitctl_<version>_linux_arm64` |
| macOS | Intel | `budgetkitctl_<version>_darwin_amd64` |
| macOS | Apple silicon | `budgetkitctl_<version>_darwin_arm64` |
| Windows | x86-64 | `budgetkitctl_<version>_windows_amd64.exe` |
| Windows | ARM64 | `budgetkitctl_<version>_windows_arm64.exe` |

Each release also carries a `SHA256SUMS` file and `SHA256SUMS.asc`, a detached
GPG signature of `SHA256SUMS` made with the Pocket Network release key. The
public key is [`release-key.asc`](release-key.asc), beside these pages, and its
fingerprint is:

```
03E7 C270 9090 327D F488  6C02 8B4B EE82 473C 7CB9
```

Import the key, check that the fingerprint matches, then check the signature,
then the binary, then install it:

```sh
gpg --import release-key.asc
gpg --fingerprint developer@pokt.foundation   # must show the fingerprint above
gpg --verify SHA256SUMS.asc SHA256SUMS        # must say "Good signature"
sha256sum -c SHA256SUMS --ignore-missing      # expect: budgetkitctl_..._linux_amd64: OK
install -m 0755 budgetkitctl_<version>_linux_amd64 /usr/local/bin/budgetkitctl
budgetkitctl version                          # prints: budgetkitctl <version>
```

On macOS use `shasum -a 256 -c SHA256SUMS --ignore-missing`.

---

## Where it talks to

Every `user` command takes these flags:

| flag | meaning |
|---|---|
| `-url` | **required**: base URL of the account API. Production: `https://pay.api.pocket.network`. Beta: `https://pay.beta.api.pocket.network` |
| `-key` | **required** (every command except `logout`): your private key file |
| `-host` | overrides the HTTP `Host` header sent to `-url`; defaults to `$CONTROL_HOST`. Leave it empty on the published deployments, whose names resolve |

There are no defaults for `-url` and `-key`: a command run without them stops
and says so. The CLI refuses a plain `http://` URL to any machine other than
your own, because a signed request and a minted token would travel in clear.

And one environment variable:

| variable | meaning |
|---|---|
| `BUDGETKIT_TARGET` | the target name of the deployment your requests are signed for. **It has no default.** Every command that signs refuses to run without it: `no target: a signed request names the deployment it is for (-target, or BUDGETKIT_TARGET). Without it the request is signed for nobody` |

Each deployment has its own target name: `beta` for beta
(`https://pay.beta.api.pocket.network`); the production name is published when
production opens. A request signed for another
deployment is refused (`401`, api code 5), so set `BUDGETKIT_TARGET` to match
`-url`.

```sh
export BUDGETKIT_TARGET=beta
export BK_URL=https://pay.beta.api.pocket.network
```

The examples below use `-url "$BK_URL"`.

---

## Your key

Your account is identified by a secp256k1 key pair. The **uid** (your account
id) is the compressed public key in hex: 66 characters, starting with `02` or
`03`.

**Key file format:** the 32-byte private key as 64 hex characters, with no `0x`
prefix. Leading and trailing whitespace (including a final newline) is ignored.
Anything else is refused with `key file <path>: want 32 bytes of hex`.

### `tools keygen`: create a key

```
budgetkitctl tools keygen [-out <file>]
```

| flag | default | meaning |
|---|---|---|
| `-out` | `local/keys/pocket.key` | where to write the private key. The directory is created; the file is written with mode `0600` |

Output:

```
key: <file>
uid: <66 hex characters>
```

`keygen` never overwrites a file. If one already exists at the path it stops
with `a key already exists at <file>: it is an account and is not overwritten;
pass -out with another path, or move that file away yourself`.

```sh
budgetkitctl tools keygen -out ~/.pocket/pocket.key
```

Keep the file secret: whoever has it controls the account and its money. There
is no recovery if you lose it.

You can also use an existing secp256k1 private key (for example one exported
from a wallet as raw hex) by writing its 64 hex characters to a file. That key's
Pocket address and EVM address are the addresses you pay from.

---

## `user` commands

Every `user` command acts on the account of the key that signs it; none takes a
uid. Several print a table by default and the raw JSON with `-json`.

When the server refuses a request, the command exits with status 1 and prints
the method, the path, the status and the server's error body, for example:

```
budgetkitctl: GET /v1/balance: 401 Unauthorized: {"codespace":"api","code":4,"message":"unauthorized"}
```

The codes are listed in [api.md](api.md#errors).

Every command's `-h` output also prints the matching `tools sign` and `curl`
pair for calling the same route by hand.

### `user status`: balance and tier

```
budgetkitctl user status -key <file> -url <url> [-json] [-usd]
```

| flag | meaning |
|---|---|
| `-json` | print the raw response of `GET /v1/balance` |
| `-usd` | show money in USD (unit `USD`) instead of micro-USD (unit `µUSD`) |

Output:

```
uid       <uid>
tier      <trial|paid|exhausted>
trial     <amount> µUSD
plan      <amount> µUSD  (an active refill contract)
credit    <amount> µUSD
spendable <amount> µUSD
short for <service,service>
```

The `(an active refill contract)` note appears only while a plan refills the
plan pool; the `short for` line only when some services are below the
low-balance threshold.

```sh
budgetkitctl user status -key ~/.pocket/pocket.key -url "$BK_URL" -usd
```

### `user usage`: what you spent it on

```
budgetkitctl user usage -key <file> -url <url> [-json] [-verify]
```

| flag | meaning |
|---|---|
| `-json` | print the raw response of `GET /v1/usage` |
| `-verify` | recompute each row's cost from its own inputs and show the difference |

### `user events`: account history

```
budgetkitctl user events -key <file> -url <url> [-limit <n>] [-all] [-json]
```

| flag | meaning |
|---|---|
| `-limit` | events per request; `0` uses the server's default (100) |
| `-all` | follow the cursor until the whole history has been read. Without it, one page is shown |
| `-json` | print the raw pages |

### `user prices`: what a request costs

```
budgetkitctl user prices -key <file> -url <url> [-json] [-verify]
```

| flag | meaning |
|---|---|
| `-json` | print the raw response of `GET /v1/prices` |
| `-verify` | recompute each service's price from the published rate and show the difference |

The route needs no signature, but the CLI signs it anyway, so a key file and
`BUDGETKIT_TARGET` are still required. Without the CLI: `curl -sS
"$BK_URL/v1/prices"`.

### `user mint-token`: create an API token

```
budgetkitctl user mint-token -key <file> -url <url> [-name <name>] [-expires <RFC 3339 time>]
```

| flag | meaning |
|---|---|
| `-name` | optional label, up to 128 characters |
| `-expires` | optional expiry, RFC 3339 (e.g. `2027-01-31T00:00:00Z`). Without it the token never expires |

Prints the server's JSON response (shape; the field order is not significant):

```json
{"expires_at":null,"id":"tok_<16 hex>","name":"<name>","token":"pkt_<...>","uid":"<uid>"}
```

**The `token` value is shown only this once.** Save it now; the server keeps
only its hash. If you lose it, mint another and revoke the old one.

On failure it prints `mint failed: <status> <status text>: <error body>`. `409`
with api code 79 means the account already holds its maximum of live tokens.

```sh
budgetkitctl user mint-token -key ~/.pocket/pocket.key -url "$BK_URL" -name production-backend
```

### `user tokens list`: your tokens

```
budgetkitctl user tokens list -key <file> -url <url> [-json]
```

Output: a table with `ID`, `NAME`, `CREATED`, `EXPIRES` (`never` for no
expiry) and `STATE` (`live`, or `revoked <time>`). With no tokens it prints
`no tokens — mint one with: user mint-token`.

### `user tokens revoke`: revoke a token

```
budgetkitctl user tokens revoke -id <token id> -key <file> -url <url>
```

| flag | meaning |
|---|---|
| `-id` | **required**: the `tok_...` id from `user tokens list` |

Prints `revoked <id> — relays carrying it are refused from the next token notice`.
An unknown id, or one that is not yours, is a `404` (api code 49).

### `user payments`: how to pay

```
budgetkitctl user payments -key <file> -url <url> [-json]
```

Prints one row per coin you can pay with: `CHAIN`, `WALLET` (where to send),
`ASSET`, `DEC` (decimals), `BONUS BPS` (extra credit in basis points; 2000 =
+20%), `CONF` (confirmations before credit), `KIND` (`evm` or `cosmos`),
`CHAIN ID` (the chain your wallet must send on) and `DENOM` (what a cosmos
wallet sends, `-` for an EVM coin), followed by:

```
pay the wallet on-chain, then: user notify-payment -chain <chain> -tx-hash <hash>
```

`-json` prints the raw `GET /v1/payments/options` response, which also carries
`contract` for token coins.

### `user payments quote`: what an amount is credited as

```
budgetkitctl user payments quote -chain <chain> -asset <asset> (-usd <dollars> | -amount <base units>) -key <file> -url <url> [-json]
```

| flag | meaning |
|---|---|
| `-chain` | **required**: a `CHAIN` from `user payments` |
| `-asset` | **required**: an `ASSET` from `user payments` |
| `-usd` | the dollars you want to pay, a decimal of at most 6 places |
| `-amount` | or the amount you will send, in base units (wei, upokt) |

Exactly one of `-usd` and `-amount`. Output:

```
send      <coins> <asset>  (<base units> base units) to the <chain> wallet in `user payments`
worth     <dollars> USD  at <rate> USD per <asset>, quoted <time>
bonus     +<percent>%  (<bps> bps)
credited  <dollars> USD

an estimate: a payment is valued at the price in force when it confirms
```

### `user notify-payment`: tell the server you paid

```
budgetkitctl user notify-payment -chain <chain> (-tx-hash <hash> | -raw-tx <signed tx>) -key <file> -url <url>
```

| flag | meaning |
|---|---|
| `-chain` | **required**: a `CHAIN` from `user payments` |
| `-tx-hash` | the hash of a transaction already broadcast. EVM: `0x` + 64 hex; Pocket: 64 hex |
| `-raw-tx` | instead, a signed transaction for the server to broadcast. EVM: hex; Pocket: base64 of the signed `TxRaw`, including the signer's public key |

One of `-tx-hash` or `-raw-tx` is required. The `-key` must be the key whose
address **sent** the payment: the payment is credited to the signer only if the
chain shows it came from the signer's own address.

Prints the receipt JSON ([api.md](api.md#post-v1paymentsnotify-tell-the-server-you-paid)),
whose `status` starts at `verifying` (hash) or `to_deliver` (raw).

```sh
budgetkitctl user notify-payment -key ~/.pocket/pocket.key -url "$BK_URL" -chain <CHAIN> -tx-hash <TX-HASH>
```

### `user payments status`: one payment

```
budgetkitctl user payments status -chain <chain> -tx <hash> -key <file> -url <url>
```

| flag | meaning |
|---|---|
| `-chain` | **required** |
| `-tx` | **required**: the transaction hash you notified |

Prints the receipt JSON. Repeat until `status` is `credited` (success),
`rejected`, `reorged` or `not_found`. A hash your account never notified is a
`404` (api code 76).

### `user payments list`: payments you notified

```
budgetkitctl user payments list -key <file> -url <url> [-limit <n>] [-after <cursor>]
```

| flag | meaning |
|---|---|
| `-limit` | page size; `0` uses the server's default (100) |
| `-after` | the `after` cursor from the previous page's JSON |

Prints the raw JSON page: `{"payments":[...],"after":"<cursor>"}`, newest first.

### `user login`: open a browser-style session with your key file

```
budgetkitctl user login -domain <console domain> [-chain <chain id>] -key <file> -url <url>
```

| flag | meaning |
|---|---|
| `-domain` | **required**: the console's domain, `console.api.pocket.network` (beta: `console.beta.api.pocket.network`) |
| `-chain` | the chain id the sign-in names; default `pocket`. It must be the `CHAIN ID` of one of the deployment's payment options: on beta pass `-chain pocket-lego-testnet` |

This does what the web console does with a wallet: it builds the sign-in text
for your key's Pocket address, signs it, and calls `POST /v1/sessions`. It
prints `{"token":"...","uid":"...","expires_at":"..."}`. The token can be sent
as a `Budget-Session` header instead of a signature
([api.md](api.md#browser-sessions)). Most CLI users do not need sessions: every
command signs on its own.

### `user logout`: end a session

```
BUDGETKIT_SESSION=<token> budgetkitctl user logout -url <url>
```

The session token goes in the `BUDGETKIT_SESSION` environment variable, never
on the command line. Prints
`signed out — the session's row is gone and every replica refuses its token`.

### `user sessions list` and `user sessions revoke-all`

```
budgetkitctl user sessions list -key <file> -url <url> [-json]
budgetkitctl user sessions revoke-all -key <file> -url <url>
```

`sessions list` prints each live session's expiry (`no live sessions` when there
is none). `sessions revoke-all` ends every session of the account, those the
console opened included, and prints `revoked <n> session(s): every replica
refuses their tokens now`. Your key always works: this is how you close a
session you did not open.

---

## `tools sign`: sign a request by hand

For calling the API with `curl` or another HTTP client. It prints the three
signature headers for one request.

```
budgetkitctl tools sign -key <file> -path <path> [-method <METHOD>] [-body '<json>'] [-target <target>]
```

| flag | default | meaning |
|---|---|---|
| `-key` | (required) | your key file |
| `-path` | (required) | the request path, **without the query string**, e.g. `/v1/events` |
| `-method` | `GET` | the HTTP method |
| `-body` | empty | the exact body you will send |
| `-target` | `$BUDGETKIT_TARGET` | the deployment's target name |

Output, three lines:

```
Content-Digest: sha-256=:<base64>:
Signature-Input: sig1=("@method" "@path" "content-digest");created=<unix>;keyid="<uid>";nonce="<hex>";tag="budgetkit-v1:<target>"
Signature: sig1=:<base64>:
```

The signature is valid for 300 seconds and only once (its nonce is single use):
sign immediately before each call.

```sh
budgetkitctl tools sign -key ~/.pocket/pocket.key -method GET -path /v1/balance > headers.txt
curl -sS "$BK_URL/v1/balance" -H @headers.txt

BODY='{"name":"ci"}'
budgetkitctl tools sign -key ~/.pocket/pocket.key -method POST -path /v1/tokens -body "$BODY" > headers.txt
curl -sS -X POST "$BK_URL/v1/tokens" -H @headers.txt -H 'Content-Type: application/json' --data-binary "$BODY"
```

---

## Other

- `budgetkitctl version` prints the build version.
- A command typed without its scope (for example `budgetkitctl status`) answers
  with where it lives: ``"status" now lives under a scope: run `budgetkitctl user status` ``.
