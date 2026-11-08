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
Each service has its own price per request, set from the Pocket chain's own
parameters at the live POKT price plus a markup, and `GET /v1/prices` publishes
it with every input. The method and the size of the response do not change it;
a JSON-RPC batch is billed per element and a WebSocket per message the service
sends. As an example, on beta on 2026-11-08 a request cost about 0.31 micro-USD,
about $0.31 per million. See [index.md](index.md) "How a request is priced" and
[api.md](api.md) "What is billed".

**Is my balance in dollars or in tokens?**
Dollars, to the millionth of a dollar. A payment is valued once, when it is
credited, so a later drop in the POKT price does not shrink your balance.

**Is there a free option?**
Two. Every account gets a small **trial** allowance that refills every day, on
the paid endpoints. Separately, the **free endpoints**
(`https://<service>.free.api.pocket.network`) need no account and no token, at a
lower rate limit counted per IP address (100 requests per minute on beta).

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
Nothing is returned automatically: ask on the Discord (below), and Pocket
Network reviews each case; an approved return is sent back minus the network
fee ([billing-rules.md](billing-rules.md#returns), "Returns").

**Can I get a refund or withdraw my balance?**
There is no withdrawal, and nothing in the console or the API asks for a
refund: credit stays on the account and never expires. Any return is reviewed
and approved by Pocket Network case by case and is sent minus the network fee
([billing-rules.md](billing-rules.md#returns), "Returns"); a refund it posts to
your account shows in your Activity.

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

**What is the fastest way to make one paid request?**
Sign in to the console, mint a token on the Tokens page, and send
`Authorization: Bearer <token>` to `https://<service>.api.pocket.network`. The
daily trial pays for it. [agents.md](agents.md) has the same path for a
terminal, step by step.

**What is the difference between my key and a token?**
The key is the account: it signs account operations and never goes in a
request to a service. A `pkt_...` token is what your applications send, in the
`Authorization: Bearer` header, to the paid endpoints.

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
expires. A token cannot read your balance, mint or revoke tokens, pay, or
change anything on the account: those need the key. A revoked token is refused
normally within seconds. Do not put a token in a public web page or app.

**Can I limit a token?**
By name and expiry only. There is no per-service token and no spending cap per
token; the account's balance is the limit.

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
seconds. A paid endpoint answers a missing or unknown token with `402`, not
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
response, and for a payment the chain and the transaction hash. Never share
your private key or a token; nobody from Pocket Network will ask for them.
