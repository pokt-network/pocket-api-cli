# Questions newcomers ask

Short answers, each with the page that has the details.

## The account

**What does "my account is a secp256k1 key" mean, and how do I get one?**
Your account is a secp256k1 key pair, the same kind Pocket and Ethereum
wallets use. The public key is your account id; there is no sign-up, no email
and no password, and the account exists the first time you use the key. Sign in
to the console with a Pocket or EVM wallet, or run `budgetkitctl tools keygen`.
See [index.md](index.md) "Your account is a secp256k1 key" and [cli.md](cli.md).

**Can I reuse my MetaMask or Pocket wallet key?**
Yes. In the console you sign in with the wallet and the key never leaves it.
The CLI and an agent can use an existing key too ([cli.md](cli.md) shows how),
but they read it from a plain file, so a key that also holds other funds puts
those funds on disk; a key made for this account alone is safer.

**If I lose the key, is my balance gone?**
Yes. Nobody, Pocket Network included, can recover a lost key, and whoever holds
the key controls the account. Keep a backup of it somewhere safe.

**Does my private key leave my machine when I use the console?**
No. The console only asks your wallet to sign a plain-text message; the
private key stays in the wallet. See [console.md](console.md) "Signing in".

**How does a team share one account?**
There are no team roles or shared logins. One person holds the key and mints a
separate API token for each teammate or application, each with its own name and
expiry; every token draws from the same balance, and any of them can be revoked
alone. Minting, revoking and paying need the key.

## Money

**What does a request cost?**
Each service has its own price per request: the compute units the Pocket chain
assigns to one of its requests, valued at Pocket's CU peg, plus the Pocket API
Pro service fee. The
market price of POKT does not change it. The method and the size of the
response do not change it either; a JSON-RPC batch is billed per element and a
WebSocket per message the service sends. A request of c compute units costs
c x peg / 10^9 micro-USD, where the peg is `peg_micro_usd_per_billion_cu`,
times (10000 + `markup_bps`) / 10000, where `markup_bps` is the service fee in
basis points. The peg, the service fee and every service's
price in force are published at `https://pay.api.pocket.network/v1/prices`.
See [index.md](index.md) "How a request is priced" and [api.md](api.md) "What
is billed".

**Is my balance in dollars or in tokens?**
Dollars, to the millionth of a dollar. A payment is valued once, when it is
credited, so a later change in a coin's price does not shrink your balance. POKT
is valued at the peg rate, the rate requests are priced at, not at its exchange
price.

**Is there a free option?**
Two. Every account gets a small **trial** allowance that refills every day, on
the Pocket API Pro endpoints. Separately, **Pocket API Free**, the free
endpoints (`https://<service>.api.pocket.network`), needs no account and no
token, at a lower rate limit counted per IP address and per service. A request
over it is answered `429` with `Retry-After`, the seconds to wait. JSON-RPC
batches are served there too; see [api.md](api.md#the-free-endpoint).

**How do I pay, and how long does it take?**
Send an accepted coin (POKT, or a native or ERC-20 coin on an EVM chain) from
your own key's address to the receiving wallet the console's Fund page shows,
then notify the transaction. It is credited once the network reaches its
confirmations, which takes seconds to minutes depending on the chain. There is
no card payment. See [index.md](index.md) "Paying" and
[billing-rules.md](billing-rules.md).

**I sent the wrong coin, on the wrong network, or from an exchange.**
None of these is credited. A payment from an exchange or a contract wallet is
not credited because it does not come from your key's address; withdraw to
your own wallet first. [billing-rules.md](billing-rules.md) lists every case.
Returns are manual: nothing is returned automatically. Find what went wrong
(the payment's status and reason on the Fund page), and gather the chain, the
transaction hash, your account id, what you sent and the address you sent it
from. Open a ticket on the Pocket Network Discord,
https://discord.gg/pocket-network, with that information
([billing-rules.md](billing-rules.md#returns), "Returns").

**Can I get a refund or withdraw my balance?**
There is no withdrawal, and on-chain payments are not refunded through the
console or the API: credit stays on the account and never expires. For money
that reached our wallet and was not credited, gather what went wrong, the
chain, the transaction hash, your account id, what you sent and the address you
sent it from. Open a ticket on the Pocket Network Discord,
https://discord.gg/pocket-network, with that information
([billing-rules.md](billing-rules.md#returns), "Returns").

**Do I get an invoice?**
No. The record of a payment is its transaction on the chain, and the credit it
produced shows in your account's Activity.

**How do I know the payment address is really Pocket Network's?**
Compare two sources before paying: the console's Fund page and
`GET /v1/payments/options` on `https://pay.api.pocket.network` show the same
wallets. Check that your browser is on `console.api.pocket.network`, and never
pay an address that reached you any other way, such as a message or an email.

**Where do I see my balance and spending?**
The console's Account page, `budgetkitctl user status` and `user usage`, or
`GET /v1/balance` and `GET /v1/usage`.

## Using it

**What is the fastest way to make one Pocket API Pro request?**
Sign in to the console, mint a token on the Tokens page, and send
`Authorization: Bearer <token>` to `https://<service>.pro.api.pocket.network`. The
daily trial pays for it. [agents.md](agents.md) has the same path for a
terminal, step by step.

**What is the difference between my key and a token?**
The key is the account: it signs account operations and never goes in a
request to a service. A `pkt_...` token is what your applications send, in the
`Authorization: Bearer` header, to the Pro endpoints.

**My client cannot set a header. Where does the token go?**
In the first segment of the path:
`https://<service>.pro.api.pocket.network/pkt_<...>/<path>`. It is removed before
the request reaches the service. It is the easiest way and the least secure:
URLs end up in proxy and server logs, browser history and screenshots, so a
header is always preferred. A browser WebSocket offers the token as a
subprotocol instead, `new WebSocket(url, ["pocket", token])`, and the token is
never echoed back. See [api.md](api.md) "Where the token goes".

**When do I need a signed request?**
Only for the account API (`https://pay.api.pocket.network/v1/...`): tokens,
balance, payments. Requests to services use the token, never a signature. The
console and `budgetkitctl` sign for you. See [api.md](api.md) "Authentication".

**Console, CLI or API?**
The console for a browser, `budgetkitctl` for a terminal or a script, the HTTP
API for your own code. [README.md](README.md) "Start here" picks the page.

**Can an AI agent do everything without a human?**
Yes, given coins to pay with. It also sets the deployment's target name,
which every signed request carries: `beta` for beta, and the mainnet name
once mainnet opens. [agents.md](agents.md) is its
runbook.

## Security

**What can somebody do with a leaked token?**
Send requests to services billed to your account, until you revoke it or it
expires; either one also closes its open WebSocket, gRPC and streaming
sessions. An allowlist of IPs limits where it works. A token cannot read your balance, mint or revoke tokens, pay, or
change anything on the account: those need the key. A revoked token is refused
normally within seconds. Do not put a token in a public web page or app.

**How do I rotate a token?**
Mint a new one, move your apps to it, then revoke the old one: both work in
between, so nothing stops. See [index.md](index.md) "Rotating a token".

**Can I limit a token?**
By expiry, and by the browser origins and client IP addresses it may be used
from: set them when you mint it and edit them later, in the console or with
`budgetkitctl user tokens allowlists`. An origin list only stops another web
site from using the token in a browser; a server can send any `Origin`, so
restrict a server's token by IP. An edit closes the token's open sessions that
the new lists no longer allow. There is no per-service token and no spending
cap per token; the account's balance is the limit. See [api.md](api.md)
"Per-token allowlists".

**Do you store my token?**
Only its SHA-256 hash. The token is shown once, when you mint it.

**Can a captured signed request be replayed?**
No. Each signature is used once and is valid for 300 seconds, so keep your
clock synchronised. See [api.md](api.md) "Replay and retry rules".

**How do I check that the CLI I downloaded is genuine?**
Verify `SHA256SUMS.asc` with the release key; [cli.md](cli.md) "Install" has the
commands and the fingerprint.

## When something goes wrong

**My requests return 402.**
Your account has nothing left in any pool, or the token is unknown or revoked.
Pay, or use the free endpoint. A token minted seconds ago may need a few
seconds more. See [api.md](api.md) "402 Payment Required".

**What do 401, 403 and 429 mean?**
`401`: the account API refused your signature (clock, reused signature, wrong
key or wrong deployment) or your console session has ended. `403`: the console needs a fresh wallet signature to
mint or revoke. `429`: you are over your rate limit; wait the `Retry-After`
seconds. A Pro endpoint answers a missing or unknown token with `402`, not
`401`. The full lists are in [api.md](api.md) "Errors" and
[agents.md](agents.md).

**I paid but my balance did not change.**
Check the payment's status on the console's Fund page or with
`budgetkitctl user payments status`. If you never notified the transaction,
notify it now: notifying the same hash twice is harmless and credits once.
Do not pay again before reading the reason: [billing-rules.md](billing-rules.md)
says, for each `rejected` reason, whether to pay again, notify again later, or
send more.

**Where do I get help?**
On the Pocket Network Discord, https://discord.gg/pocket-network. Say what you
did and what came back, and include whichever of these apply: your account id
(the public key, 66 hex characters), the `X-Request-Id` header of the failing
response, and for a payment the chain, the transaction hash, what you sent and
the address you sent it from. For a payment that was not credited, open a
ticket on the Pocket Network Discord, https://discord.gg/pocket-network, with
that information ([billing-rules.md](billing-rules.md#returns), "Returns").
Never share your private key or a token; nobody from Pocket Network will ask
for them.
