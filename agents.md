# Runbook for AI agents: use the Pocket API Console end to end

This page is written for an automated agent with a shell. Follow the steps in
order. Every placeholder in `<ANGLE_BRACKETS>` is named in the table below with
where its value comes from. Each step says what a successful response looks
like, which errors can come back, and what to do about each.

There are two paths:

- **Path A (primary): the `budgetkitctl` CLI.** It creates the key, signs every
  request and calls the API.
- **Path B (fallback): raw HTTP with `curl`**, signing each request yourself with
  the small Go program in [api.md](api.md#a-minimal-go-signer). Use it only if
  you cannot run the CLI.

Both paths reach the same API and give the same results.

---

## Values you need

Set these once. Values marked "given" must come from whoever asked you to do the
task, or from the published documentation; the rest are produced by the steps.

| placeholder | what it is | where it comes from |
|---|---|---|
| `<API_URL>` | base URL of the account API | given. Mainnet: `https://pay.api.pocket.network`. Beta: `https://pay.beta.api.pocket.network` |
| `<TARGET>` | the target name every signature names | `beta` for beta; the mainnet name is published when mainnet opens. It has no default, and it must match `API_URL`: a wrong value gives api code 5 |
| `<RELAY_DOMAIN>` | domain suffix for relays | `api.pocket.network` for mainnet, `beta.api.pocket.network` for beta |
| `<KEY_FILE>` | path to your private key file | step A1 |
| `<UID>` | your account id: 66 hex characters starting with `02` or `03` | step A1 |
| `<SERVICE>` | a service name to relay to, e.g. `anvil` on beta | given (the deployment's published services) |
| `<TOKEN>` | your API token, `pkt_...` | step A4 (`token`) |
| `<CHAIN>`, `<WALLET>`, `<ASSET>`, `<CONTRACT>`, `<DECIMALS>`, `<MIN_CONF>`, `<CHAIN_ID>` | the payment option you choose | step A6 |
| `<PAYER_ADDRESS>` | your key's address on that chain | step A7 |
| `<CHAIN_RPC>` | an RPC endpoint of the chain you pay on, for your payment tool | the payment option's `public_rpc_url` when it has one; otherwise given. Step A8 checks its chain id before any money moves |
| `<TX_HASH>` | the hash of your payment transaction | step A8 |

```sh
export API_URL='<API_URL>'
export BUDGETKIT_TARGET='<TARGET>'
export RELAY_DOMAIN='<RELAY_DOMAIN>'
umask 077   # every file this runbook writes (key, token) is readable by you only
```

Facts you can rely on:

- Money in the API is an **integer number of micro-USD** (1 USD = 1,000,000).
- Your account is your secp256k1 key. There is no sign-up call: the account exists the
  first time you use the key.
- Every signed request is valid for **300 seconds** from signing and **once**.
  The CLI signs fresh on every call. If you sign by hand, sign right before each
  call and never resend the same headers.
- Your machine's clock must be within 300 seconds of real time. Check with
  `date -u`.
- Beta and mainnet are separate deployments: a key's balance, tokens and
  payments on one are not on the other.

Tools used (install what your path needs):

| tool | needed for |
|---|---|
| `budgetkitctl` | Path A |
| `curl` | relays (step A5) and Path B |
| `jq` | reading JSON in the commands below |
| `go` | Path B only, to build the signer (tested with Go 1.27) |
| `cast` (Foundry) | paying on an EVM chain |
| `pocketd` | paying on Pocket |

---

## Path A: with the CLI

### A0. Install the CLI

Download the binary for your platform from
https://github.com/pokt-network/pocket-api-cli/releases. File names:
`budgetkitctl_<version>_<os>_<arch>`, with `<os>` one of `linux`, `darwin`,
`windows` (`.exe`) and `<arch>` one of `amd64`, `arm64`. A `SHA256SUMS` file
sits beside them ([cli.md](cli.md#install) also checks its signature).

```sh
sha256sum -c SHA256SUMS --ignore-missing     # expect: budgetkitctl_..._linux_amd64: OK
chmod +x budgetkitctl_<version>_linux_amd64
mv budgetkitctl_<version>_linux_amd64 ./budgetkitctl
./budgetkitctl version                       # expect: budgetkitctl <version>
```

The commands below write `budgetkitctl`; use `./budgetkitctl` if it is not on
your `PATH`.

**How the CLI reports errors:** on any refusal it exits with status 1 and prints
one line to stderr:

```
budgetkitctl: <METHOD> <PATH>: <HTTP status> <status text>: {"codespace":"...","code":N,"message":"..."}
```

Look up `codespace` and `code` in [Error codes](#error-codes-and-what-to-do).
Always check the exit status (`echo $?`) rather than only reading the output.

### A1. Create your key

```sh
mkdir -p ./bk
budgetkitctl tools keygen -out ./bk/agent.key | tee ./bk/keygen.out
```

Success (exit 0) prints exactly two lines:

```
key: ./bk/agent.key
uid: 02<64 hex characters>
```

Each run makes a new key, and `keygen` refuses to overwrite an existing file
(`a key already exists at ./bk/agent.key: ...`): keep the key you made. Capture
the uid from the saved output:

```sh
export KEY_FILE=./bk/agent.key
export UID_HEX=$(awk '/^uid:/{print $2}' ./bk/keygen.out)
echo "$UID_HEX"        # expect 66 hex characters starting with 02 or 03
```

**Using an existing key instead** (for example one that already holds funds):
write its private key as exactly 64 hex characters, **without a `0x` prefix**,
into a file, and use that file as `KEY_FILE`. The CLI refuses anything else
with `key file <path>: want 32 bytes of hex`. Its uid is the `keyid` printed by
`budgetkitctl tools sign -key "$KEY_FILE" -path /v1/balance` (the value inside
`keyid="..."`).

### A2. Read your balance (proves signing works)

```sh
budgetkitctl user status -key "$KEY_FILE" -url "$API_URL" -json
```

Success: exit 0 and a JSON object (shape):

```json
{"uid":"<UID>","trial_micro_usd":0,"plan_micro_usd":0,"credit_micro_usd":0,"spendable_micro_usd":0,
 "tier":"trial","version":0,"low_services":[],"has_plan":false,"next_trial_reset_at":"<time>"}
```

Check: `uid` equals `UID_HEX`. A brand-new key shows the starting state: the
daily trial allowance in `trial_micro_usd` (if the deployment grants one) and
`tier` `trial`; or `tier` `exhausted` with all zeros if there is no trial.

Without `-json` the same data prints as lines (`uid`, `tier`, `trial`, `plan`,
`credit`, `spendable`, amounts in `µUSD`).

Errors here:

| you see | meaning | do |
|---|---|---|
| `no target: a signed request names the deployment it is for ...` | `BUDGETKIT_TARGET` is empty | export it |
| `-url is required ...` or `-key is required ...` | a flag is missing | pass it |
| `401` api 5 | `BUDGETKIT_TARGET` does not match the deployment at `API_URL` | fix the target |
| `401` api 4 | signature refused: usually the clock is off by more than 300 s | fix the clock, retry |
| `404` api 78 | right host, but a route that is not published there | check the path |
| `402`, or `404` with `{"error":"unknown service",...}` | the request reached a service host, not the account API | use the `pay.` host as `API_URL` |
| connection error | wrong `API_URL` or no network | check `API_URL` |

### A3. Choose a service

The service is named in the relay address: `https://<SERVICE>.<RELAY_DOMAIN>/`.
On beta the services are `anvil` (an EVM chain, JSON-RPC) and `pocket` (the
Pocket beta chain: Cosmos REST and CometBFT RPC). For the relay example in step
A5, use `anvil`.

```sh
export SERVICE='<SERVICE>'
```

Prices are public: `budgetkitctl user prices -key "$KEY_FILE" -url "$API_URL"`
lists every service the chain prices, by its Pocket service id, which can
differ from the name in the address (on beta, `anvil` is priced and billed as
`pnf-anvil`, and `pocket` as `pnf-pocket-beta`). `503` api 65 means prices are
not loaded yet: wait 30 s and retry.

### A4. Mint an API token

```sh
budgetkitctl user mint-token -key "$KEY_FILE" -url "$API_URL" -name agent > ./bk/token.json
chmod 600 ./bk/token.json
export TOKEN=$(jq -r .token ./bk/token.json)
```

Success: exit 0 and one JSON object (shape):

```json
{"expires_at":null,"id":"tok_<hex>","name":"agent","token":"pkt_<...>","uid":"<UID>"}
```

Check: `TOKEN` starts with `pkt_`. **This is the only time the token is
shown.** It is saved in `./bk/token.json`; keep that file. To expire the token,
add `-expires 2027-01-31T00:00:00Z` (RFC 3339).

Errors: `mint failed: 409 Conflict: ...code":79...` means the account already
holds its maximum of live tokens: list them with
`budgetkitctl user tokens list -key "$KEY_FILE" -url "$API_URL"` and revoke one
with `budgetkitctl user tokens revoke -key "$KEY_FILE" -url "$API_URL" -id <tok_id>`.
A `500` (api 46) may still have minted a token: list tokens before minting again.

### A5. Send relays with the token

Relays do not use the CLI. They go to the service's own host:

```sh
curl -sS -i -X POST "https://$SERVICE.$RELAY_DOMAIN/" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

This body fits an EVM JSON-RPC service; other services take their own
protocol's request.

| status | meaning | do |
|---|---|---|
| `200` with a JSON-RPC `result` | **success**: the relay was served and charged to your account | continue |
| `402`, body `error` = `payment required: this request carries no token of a known account` | the token has not reached the relay hosts yet, or it is wrong or revoked (all three give the same answer) | if you minted it in the last minute, wait 5 s and retry, up to 1 minute; otherwise check `TOKEN`, and that it was not revoked |
| `402`, body `error` = `payment required: this account has no balance left` | your account is out of money | do steps A6 to A10 (pay), then retry |
| `429` | over your tier's rate limit, or too many requests from your address | wait the `Retry-After` seconds, retry |
| `503` with `Retry-After: 1` | the rate limiter could not decide | retry after 1 s |
| `404` `{"error":"unknown service","service":"..."}` | no such service on this domain | check `SERVICE` |
| `421 Misdirected Request` | the host name is not one of this deployment's domains | check `RELAY_DOMAIN` |

The response headers `X-RateLimit-Limit`, `X-RateLimit-Remaining` and
`X-RateLimit-Reset` (a Unix time) say how much of your tier's rate is left, and
`Pocket-Tier` says the tier you were served at.

To confirm the charge, run
`budgetkitctl user usage -key "$KEY_FILE" -url "$API_URL" -json`: a row for the
service (under its Pocket service id) with `count` > 0 appears once the relays
have been reported, normally within seconds; if it is not there yet, wait and
retry. `cost_micro_usd` is a whole number of micro-USD: each charge is the floor
of the running exact total, and the fraction is carried to the next charge
rather than lost. One request can cost less than one micro-USD, so after a few
requests `cost_micro_usd` can still read `0`, and a trial balance unchanged,
while `count` grows: that is not a failure.

A `402` body carries everything needed to pay (shape):

```json
{"codespace":"api","code":41,"error":"payment required: ...","scheme":"budget",
 "accepts":[{"scheme":"budget","network":"<CHAIN>","asset":"<ASSET>","contract":"<CONTRACT>","pay_to":"<WALLET>",
             "extra":{"bonus_bps":0,"min_confirmations":0}}],
 "notify":"<API_URL>/v1/payments/notify","free_endpoint":"https://<SERVICE>.free.<RELAY_DOMAIN>"}
```

### A6. List payment options and choose one

```sh
budgetkitctl user payments -key "$KEY_FILE" -url "$API_URL" -json | tee ./bk/options.json
```

Success: `{"chains":[ ... ]}`, one entry per coin (shape):

```json
{"chain":"<CHAIN>","asset":"<ASSET>","wallet":"<WALLET>","contract":"<CONTRACT, only for a token>",
 "decimals":18,"bonus_bps":0,"min_confirmations":3,
 "kind":"evm","chain_id":"<CHAIN_ID>","denom":"<DENOM, only for a cosmos coin>",
 "public_rpc_url":"<optional>","display_name":"<optional>","rate_micro_usd":0,"quoted_at":"<time>"}
```

- `kind` is `evm` or `cosmos`; `chain_id` is the chain the payment must be sent
  on, as that chain names itself (`"84532"` in decimal for an EVM chain, a
  string such as `"pocket-lego-testnet"` for a cosmos one).
- `denom` (cosmos only) is what your wallet sends, e.g. `upokt`; `asset` is how
  the service names the coin.
- `public_rpc_url` and `display_name`, when present, are an RPC you may use for
  this chain and its name; absent means none is published.

Choose ONE entry: the chain you can pay on and hold funds for, and the coin you
will send. A chain can have several entries, one per coin: match on `chain` AND
`asset`; an entry with no `contract` is the chain's own coin (ETH, POKT), an
entry with a `contract` is a token sent through that contract. Set:

```sh
export CHAIN='<chain>' WALLET='<wallet>' ASSET='<asset>' DECIMALS='<decimals>' MIN_CONF='<min_confirmations>'
export CONTRACT='<contract, or empty for a native coin>' CHAIN_ID='<chain_id>'
export CHAIN_RPC='<public_rpc_url if the entry has one, otherwise the RPC you were given>'
```

- A `wallet` starting with `0x` is an EVM chain; `pokt1...` is Pocket.
- `bonus_bps` is extra credit: `2000` means the payment is credited at 120% of
  its value.
- An empty `chains` list means no coin can be priced right now: wait a few
  minutes and retry.

To see what an amount is credited as before paying:

```sh
budgetkitctl user payments quote -key "$KEY_FILE" -url "$API_URL" -chain "$CHAIN" -asset "$ASSET" -usd 1 -json
```

Its `amount` is the base units to send for that many dollars, and
`credited_micro_usd` what they are credited as. It is an estimate: the price in
force when the payment is credited decides.

### A7. Find your paying address and fund it

**A payment is credited only if it is sent from your key's own address.** Paying
from any other address credits nothing to you (the receipt says `rejected`,
reason `not_the_sender`).

EVM address of your key:

```sh
export PAYER_ADDRESS=$(cast wallet address --private-key 0x$(cat "$KEY_FILE"))
```

Pocket address of your key (the standard cosmos-sdk key commands):

```sh
pocketd keys import-hex agent "$(cat "$KEY_FILE")" --keyring-backend test
export PAYER_ADDRESS=$(pocketd keys show agent -a --keyring-backend test)
```

`PAYER_ADDRESS` must hold the amount you will pay plus the network fee. A new
key holds nothing: transfer funds to `PAYER_ADDRESS` from a funded wallet first,
or use a funded existing key (step A1). Where those funds come from is outside
this API. On a test network, use that network's faucet to fund
`PAYER_ADDRESS` itself; paying from an unrelated wallet credits nothing.

### A8. Pay

Amounts on chain are in base units: `amount x 10^DECIMALS`.

Before paying on EVM, check the RPC is on the RIGHT CHAIN, answers, and that
your address holds funds. **Stop if the chain id differs from the option's
`chain_id`**: a payment sent on another chain never reaches the wallet the
option names and is not credited.

```sh
test "$(cast chain-id --rpc-url "$CHAIN_RPC")" = "$CHAIN_ID" || { echo "wrong chain"; exit 1; }
cast block-number --rpc-url "$CHAIN_RPC"
cast balance "$PAYER_ADDRESS" --rpc-url "$CHAIN_RPC"
```

**EVM, native coin** (`CONTRACT` empty). `cast` accepts a unit suffix such as
`0.01ether`:

```sh
cast send "$WALLET" --value 0.01ether --private-key 0x$(cat "$KEY_FILE") --rpc-url "$CHAIN_RPC" --json | tee ./bk/pay.json
export TX_HASH=$(jq -r .transactionHash ./bk/pay.json)
```

**EVM, token** (`CONTRACT` set): call `transfer(address,uint256)`; no value.

```sh
cast send "$CONTRACT" 'transfer(address,uint256)' "$WALLET" <AMOUNT_IN_BASE_UNITS> \
  --private-key 0x$(cat "$KEY_FILE") --rpc-url "$CHAIN_RPC" --json | tee ./bk/pay.json
export TX_HASH=$(jq -r .transactionHash ./bk/pay.json)
```

Success: `status` in `./bk/pay.json` is `"0x1"`, and `TX_HASH` is `0x` plus 64
hex characters. A `status` of `"0x0"` means the transaction failed on chain and
nothing was paid.

**Pocket (POKT).** The amount is in `upokt` (1 POKT = 1,000,000 upokt), and the
chain id is the option's `chain_id` (`pocket-lego-testnet` on beta).
`<POCKET_NODE>` is a Pocket node RPC endpoint you were given:

```sh
pocketd tx bank send agent "$WALLET" <AMOUNT>upokt \
  --chain-id "$CHAIN_ID" --node '<POCKET_NODE>' --keyring-backend test \
  --fees <FEE>upokt -y -o json | tee ./bk/pay.json
export TX_HASH=$(jq -r .txhash ./bk/pay.json)
```

Success: `code` is `0` and `TX_HASH` is 64 hex characters.

### A9. Notify the payment

```sh
budgetkitctl user notify-payment -key "$KEY_FILE" -url "$API_URL" -chain "$CHAIN" -tx-hash "$TX_HASH"
```

Use the **same key** that sent the payment. Success: exit 0 and a receipt (shape):

```json
{"uid":"<UID>","chain":"<CHAIN>","tx_hash":"<TX_HASH>","notified_at":"<time>","status":"verifying"}
```

Notifying the same transaction again is harmless: it returns the current
receipt. If the command fails with payments code 13 (`503`), send the same
notice again.

### A10. Poll until credited

```sh
for i in $(seq 1 120); do
  R=$(budgetkitctl user payments status -key "$KEY_FILE" -url "$API_URL" -chain "$CHAIN" -tx "$TX_HASH" 2>&1) || { echo "$R"; break; }
  S=$(echo "$R" | jq -r .status); echo "$(date -u +%T) $S"
  case "$S" in credited|rejected|reorged|not_found) echo "$R"; break;; esac
  sleep 10
done
```

This waits up to 20 minutes. A chain may need several minutes for `MIN_CONF`
confirmations.

| status | meaning | do |
|---|---|---|
| `to_deliver`, `verifying` | looking for the transaction on chain | keep polling |
| `waiting_confirmations` | found; waiting for `MIN_CONF` blocks | keep polling |
| `confirmed` | the credit is being written | keep polling |
| **`credited`** | **success** | go to the success check |
| `rejected` | not a valid payment; `reason` and `reason_message` say why | see the table below |
| `reorged` | its block vanished and it did not come back | pay again, or notify again if it re-appears on chain |
| `not_found` | the chain never showed it (24 hours after the notice, unless the deployment sets another time) | check `TX_HASH` and `CHAIN`; notifying again reopens it |

A credited receipt (shape):

```json
{"uid":"<UID>","chain":"<CHAIN>","tx_hash":"<TX_HASH>","notified_at":"<time>","status":"credited",
 "coins":[{"asset":"<ASSET>","amount_asset":"<base units>","credited_micro_usd":0,"amount_micro_usd":0,"bonus_bps":0}]}
```

`credited_micro_usd` is what was added to your credit; `credited_micro_usd -
amount_micro_usd` is the bonus.

Rejection reasons (`reason`):

| `reason` | do |
|---|---|
| `not_the_sender` | you paid from an address that is not your key's. Pay again from `PAYER_ADDRESS` |
| `not_ours` | the recipient was not `WALLET`. Pay again to `WALLET` |
| `coin_not_accepted` | wrong coin or token contract. Use the chosen option's coin |
| `execution_failed` | the transaction failed on chain. Pay again |
| `two_senders` | more than one sender in one transaction. Use one plain transfer |
| `no_price` | temporary; notify the same hash again later |
| `credit_zero` | too small to credit. Pay a larger amount |
| `credit_overflow` | too large for one credit. It can be returned: billing-rules.md, "Returns" |
| `unattributable` | ETH reached `WALLET` funded by two or more addresses in one transaction, or a `SELFDESTRUCT` ran. Send plain ETH from `PAYER_ADDRESS` itself; it can be returned: billing-rules.md, "Returns" |
| `broadcast_refused`, `raw_too_large` | only for `raw_tx`; broadcast yourself and notify `tx_hash` |

**Success check:**

```sh
budgetkitctl user status -key "$KEY_FILE" -url "$API_URL" -json | jq '{tier, credit_micro_usd, spendable_micro_usd}'
```

`credit_micro_usd` has grown by the receipt's `credited_micro_usd`, and `tier`
is `paid`. Relays (step A5) now succeed even when the trial is used up.

---

### A11. Revoke the token when you are done

```sh
budgetkitctl user tokens revoke -key "$KEY_FILE" -url "$API_URL" -id "$(jq -r .id ./bk/token.json)"
```

Success: `revoked tok_<hex> — relays carrying it are refused from the next token
notice`. A token notice is the message that tells every relay host about the
change; it normally arrives within seconds. After it, a relay with `TOKEN` gets
`402` with `payment required: this request carries no token of a known account`.

## Path B: raw HTTP, signing by hand

Use this when `budgetkitctl` cannot run. You sign each request with the program
from [api.md](api.md#a-minimal-go-signer), which prints the three headers in a
file that `curl -H @file` reads.

### B0. Build the signer

Needs a Go toolchain (tested with Go 1.27) and network access to the Go module
proxy.

```sh
mkdir -p ./bksign && cd ./bksign
# save the program from api.md ("A minimal Go signer") here as main.go, then:
go mod init bksign
go get github.com/decred/dcrd/dcrec/secp256k1/v4@v4.4.1
go build -o ../bksign-bin .
cd ..
```

**Self-test (do this once).** It must print exactly the `Signature` line shown;
if it does not, the signer is broken and nothing else will work:

```sh
printf '0000000000000000000000000000000000000000000000000000000000000001\n' > ./bk-vector.key
./bksign-bin -key ./bk-vector.key -target local -method GET -path /v1/balance \
  -created 1752674625 -nonce 00112233445566778899aabbccddeeff | tail -1
# expect:
# Signature: sig1=:82o6gDDINUqhA+aHJILeYrDE44UUttzZfjdw3/Rut4Es8TK0irYoarM/zCF8d8IByqA/8kexv0iSII4Cqukl1w==:
```

If you have `budgetkitctl` but prefer `curl`, `budgetkitctl tools sign -key
<file> -method <M> -path <P> [-body '<json>']` prints the same three lines and
can replace `./bksign-bin` below.

### B1. Create your key

```sh
mkdir -p ./bk
export UID_HEX=$(./bksign-bin -keygen ./bk/agent.key)
export KEY_FILE=./bk/agent.key
echo "$UID_HEX"        # expect 66 hex characters starting with 02 or 03
```

To use an existing key, write its 64 hex characters (no `0x`) to a file.

### How to call a signed route

For every signed call: (1) fix the exact body, (2) sign the method, the path
**without the query string**, and that body, (3) send the headers with exactly
that body. Sign again for every call.

```sh
sign() { ./bksign-bin -key "$KEY_FILE" -method "$1" -path "$2" -body "${3:-}" > ./bk/headers.txt; }
```

### B2. Read your balance

```sh
sign GET /v1/balance
curl -sS -w '\nHTTP %{http_code}\n' "$API_URL/v1/balance" -H @./bk/headers.txt
```

Success: `HTTP 200` and the same JSON as step A2.

### B3. Choose a service

As step A3. Prices need no signature: `curl -sS "$API_URL/v1/prices"`.

### B4. Mint a token

```sh
BODY='{"name":"agent"}'
sign POST /v1/tokens "$BODY"
curl -sS -w '\nHTTP %{http_code}\n' -X POST "$API_URL/v1/tokens" \
  -H @./bk/headers.txt -H 'Content-Type: application/json' --data-binary "$BODY" > ./bk/token.out
export TOKEN=$(head -1 ./bk/token.out | jq -r .token)
```

Success: `HTTP 201` and the same JSON as step A4. The body you send must be
byte-identical to the one you signed: use `--data-binary` with the same
variable.

To list tokens: `sign GET /v1/tokens` then `curl -sS "$API_URL/v1/tokens" -H @./bk/headers.txt`.
To revoke: `sign DELETE /v1/tokens/<tok_id>` then
`curl -sS -X DELETE "$API_URL/v1/tokens/<tok_id>" -H @./bk/headers.txt` (`204` = done).

### B5. Send relays

Exactly as step A5 (relays never need a signature, only the token).

### B6. List payment options

No signature:

```sh
curl -sS "$API_URL/v1/payments/options" | tee ./bk/options.json
```

Same shape and choices as step A6. A quote needs no signature either:
`curl -sS "$API_URL/v1/payments/quote?chain=$CHAIN&asset=$ASSET&usd=1"`.

### B7, B8. Find your address, fund it, pay

Exactly as steps A7 and A8.

### B9. Notify

```sh
BODY=$(jq -cn --arg c "$CHAIN" --arg h "$TX_HASH" '{chain:$c, tx_hash:$h}')
sign POST /v1/payments/notify "$BODY"
curl -sS -w '\nHTTP %{http_code}\n' -X POST "$API_URL/v1/payments/notify" \
  -H @./bk/headers.txt -H 'Content-Type: application/json' --data-binary "$BODY"
```

Success: `HTTP 200` and a receipt as in step A9. Do not add a `uid` field.

### B10. Poll

```sh
for i in $(seq 1 120); do
  sign GET "/v1/payments/$CHAIN/$TX_HASH"
  R=$(curl -sS "$API_URL/v1/payments/$CHAIN/$TX_HASH" -H @./bk/headers.txt)
  S=$(echo "$R" | jq -r '.status // empty'); echo "$(date -u +%T) ${S:-$R}"
  case "$S" in credited|rejected|reorged|not_found) echo "$R"; break;; esac
  sleep 10
done
```

Same statuses, reasons and success check as step A10 (for the success check,
`sign GET /v1/balance` and call `/v1/balance`).

---

## Error codes and what to do

Every refusal from the account API is JSON:
`{"codespace":"<api|payments>","code":<number>,"message":"<fixed sentence>"}`.
Branch on `codespace` and `code`. The full tables are in
[api.md](api.md#errors).

### `api` codespace

| code | HTTP | meaning | do next |
|---|---|---|---|
| 1 | 413 | body over 1 MiB | send a smaller body |
| 2 | 400 | body could not be read | resend |
| 3 | 400 | body does not match the route (unknown or missing field, wrong type) | fix the body; for notify, send `chain` plus `tx_hash` or `raw_tx` |
| 4 | 401 | signature not accepted: clock off by more than 300 s, nonce reused, body or path differs from what was signed, query string included in the signed path, or wrong key file | check `date -u`; sign again right before the call; sign the path without `?...`; send the exact signed body |
| 5 | 401 | signed for another deployment | fix `BUDGETKIT_TARGET` |
| 6, 7, 8, 9, 35 | 5xx | a service problem | wait 10 s, sign again, retry (at most 5 times) |
| 10 to 34 | 401 | a signature header is malformed; the message names the part | your signer is broken: run the Path B self-test, or use the CLI |
| 41 | 402 | payment required (relays) | see step A5 |
| 45 | 400 | malformed mint body | send `{}` or `{"name":"..."}` |
| 46 | 500 | mint failed | list tokens before minting again |
| 47, 48, 50, 68, 69, 75, 80 | 5xx | a read or write failed on the server | retry after 10 s |
| 49 | 404 | token not found (unknown or not yours) | check the `tok_` id |
| 65 | 503 | no prices loaded yet | wait 30 s, retry |
| 66 | 400 | bad `limit` | use a non-negative integer |
| 67 | 400 | bad `after` cursor | use the `after` value from the previous page |
| 76 | 404 | this account never notified that payment | notify it (step A9 or B9), check `CHAIN` and `TX_HASH` |
| 77 | 503 | payments are not configured on this deployment | stop; report it |
| 78 | 404 | no such route here | you are calling the wrong host or path; use `API_URL` |
| 79 | 409 | too many live tokens | revoke one, then mint |
| 98, 99, 100 | 400, 404, 422 | the quote's query is wrong, the coin is not offered, or the amount cannot be quoted | fix the query; use a row of the options |

### `payments` codespace (from notify)

| code | HTTP | meaning | do next |
|---|---|---|---|
| 1 | 400 | need `chain` and `tx_hash` or `raw_tx` | fix the body |
| 3 | 400 | `uid` in the body | remove it |
| 4 | 400 | unknown chain | use a `chain` from the options |
| 5 | 400 | your key has no address on this chain | choose another option |
| 7 | 400 | not a valid hash for this chain | EVM: `0x` + 64 hex; Pocket: 64 hex |
| 13 | 503 | could not register the payment | send the same notice again |
| 8 to 12, 14, 24, 26 | 400 | `raw_tx` problems | notify the `tx_hash` instead |

Receipt `reason_code` values 15 to 27 are the rejection reasons in step A10.

---

## Checklist: how to tell the whole run succeeded

1. `tools keygen` (or `bksign -keygen`) printed a uid of 66 hex characters.
2. The balance call returned `200` with your `uid`.
3. The mint returned `201` with a `token` starting `pkt_`.
4. A relay to `https://<SERVICE>.<RELAY_DOMAIN>/` with `Authorization: Bearer <TOKEN>` returned `200`.
5. The payment receipt reached `"status":"credited"`, and the balance afterwards
   shows `credit_micro_usd` increased by `credited_micro_usd` and `tier` `paid`.
6. After A11, a relay with the revoked token returned `402`.
