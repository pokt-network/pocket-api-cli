# Pocket API

Pocket API gives access to the services Pocket Network serves: blockchain RPC
and the other APIs listed by the deployment. It comes in three parts:

- **Pocket API Free**: the free domain. No account and no token, at lower rate
  limits.
- **Pocket API Pro**: the Pro domain. Requests carry an API token and are paid
  from your account.
- **Pocket API Console**: the web console, where you manage that account.

Behind Pocket API Pro is a billing system that keeps your balance, charges each
request you send, and credits the payments you make on chain.

**Are you an AI agent, or automating this end to end?** Start with
[agents.md](agents.md): one runbook, every command in order, from creating a
key to sending requests and paying.

Other pages:

- [faq.md](faq.md): short answers to the questions newcomers ask
- [console.md](console.md): the web console in a browser
- [cli.md](cli.md): the `budgetkitctl` command-line tool
- [api.md](api.md): the HTTP API, request signing, relays and error codes
- [billing-rules.md](billing-rules.md): which payments are credited, and which are not

---

## Your account is a secp256k1 key

You do not sign up with an email or a password. Your account is a
**cryptographic key pair** (secp256k1, the same kind of key Pocket and Ethereum
wallets use):

- The **private key** stays with you, in your wallet or in a key file. It proves
  that requests come from you. Pocket Network never sees it.
- The **public key**, written as 66 hex characters, is your **account id**
  (the "uid").
- There is no registration step. Your account exists from the first time you
  use the key.

Because it is a normal wallet key, the same key also has a Pocket address
(`pokt1...`) and an EVM address (`0x...`). That is how a payment is linked to
your account: **a payment counts for you only if it is sent from your key's own
address.**

Keep the private key safe. Whoever holds it controls the account and its money,
and a lost key cannot be recovered.

---

## Your money: three pools

Balances are kept in US dollars, to the millionth of a dollar (one micro-USD).
Your balance is split into three pools:

| pool | what it is | does it expire? |
|---|---|---|
| **Trial** | a free allowance, the same for everybody, that refills every day | what you do not use is gone at the daily refill |
| **Plan** | an amount arranged with Pocket Network that refills on its own schedule (for example monthly) | it resets at each refill; it does not carry over |
| **Credit** | money you paid in | **never expires** |

Each request is paid from the pools **in this order: trial first, then plan,
then credit.** So the free allowance is always used before any money you paid.

The trial amount is set by each deployment. The console's Account page and
`GET /v1/balance` show your trial pool and when it refills.

---

## Tiers: what happens as you spend

| tier | when | what you get |
|---|---|---|
| **paid** | you have plan or credit money left | the paid rate limits |
| **trial** | only trial money is left | service continues, at the trial rate limits |
| **out of money** | nothing left in any pool | Pro requests are refused with HTTP `402 Payment Required` |

A `402` response tells you how to get going again: it lists where and how to
pay, and names the free endpoint you can use meanwhile. Running out of money is
never answered with `429 Too Many Requests`, because waiting does not help.

The rate limits of each tier are deployment configuration. Every request served
on the Pro domain carries `X-RateLimit-Limit`, `X-RateLimit-Remaining` and
`X-RateLimit-Reset` headers, and a `Pocket-Tier` header naming the tier it was
served at ([api.md](api.md#relays)).

---

## API tokens

Your applications do not use your private key. They use **API tokens**:

- You create ("mint") a token in the console or with the CLI. It looks like
  `pkt_...`.
- Your app sends it with every request, in the standard header
  `Authorization: Bearer <token>`. Prefer the header always. A client that
  cannot set one can put the token as the first segment of the path
  (`https://<service>.pro.api.pocket.network/pkt_<...>/...`): it is the easiest
  way and the least secure, because URLs end up in logs, browser history and
  screenshots. A browser's WebSocket offers it as a subprotocol instead:
  `new WebSocket(url, ["pocket", token])`. See [api.md](api.md) "Where the
  token goes".
- **The token is shown only once**, when you create it. The service stores only
  a fingerprint (a hash) of it. Lost it? Mint a new one.
- You can hold many tokens (one per app is a good habit), give each a name and
  an optional expiry date, and revoke any of them at any time. A revoked or
  expired token's open WebSocket, gRPC and streaming sessions are closed.
- A token can be limited to the browser origins and the client IP addresses
  it may be used from, and those lists can be edited later. An origin list
  only restricts browsers, since any server can send any `Origin`; an IP list
  binds a server too. See [api.md](api.md) "Per-token allowlists".

All tokens draw from the same account balance.

### Rotating a token

There is no single "rotate" action, on purpose: rotating is minting a new
token and revoking the old one, in that order, so your apps never stop.

1. Mint a new token (it can carry the same name as the old one).
2. Put the new token in every app that used the old one, and check that they
   get answers.
3. Revoke the old token. Relays carrying it are refused normally within
   seconds.

Both tokens work between steps 1 and 3. If the account already holds its
maximum of live tokens, the mint in step 1 is refused: revoke a token you no
longer use first.

---

## Where to send requests

Mainnet:

| what | address |
|---|---|
| Pocket API Pro: requests with a token | `https://<service>.pro.api.pocket.network` |
| Pocket API Free: requests with no token, lower limits | `https://<service>.api.pocket.network` |
| Account API (balance, tokens, payments) | `https://pay.api.pocket.network` |
| Pocket API Console | `https://console.api.pocket.network` |

Beta:

| what | address |
|---|---|
| Pocket API Pro: requests with a token | `https://<service>.pro.beta.api.pocket.network` |
| Pocket API Free: requests with no token | `https://<service>.beta.api.pocket.network` |
| Account API | `https://pay.beta.api.pocket.network` |
| Pocket API Console | `https://console.beta.api.pocket.network` |

`<service>` is the name of the service as the deployment publishes it. On beta
the services are:

| service | what it is |
|---|---|
| `anvil` | an EVM chain: JSON-RPC over HTTP and WebSocket |
| `pocket` | the Pocket beta chain (`pocket-lego-testnet`): Cosmos REST, CometBFT RPC, WebSocket, gRPC and gRPC-Web |

`GET /v1/prices` lists every service the Pocket chain prices, which is more
than a deployment serves: on beta, only the two above answer.

The free endpoint is rate-limited per client IP address and per service. A
request over the limit is answered `429` with `Retry-After`, the seconds to
wait. It serves JSON-RPC batches over HTTP; on beta it serves no WebSocket. See
[api.md](api.md#the-free-endpoint).

**Beta and mainnet are separate deployments** with separate databases. A key
works on both, but its beta balance, tokens and payments are not its mainnet
ones. Every signed request names the deployment it is for, so a request signed
for one is refused by the other.

The price of one request for each service is public: `GET /v1/prices` on the
account API ([api.md](api.md#get-v1prices-what-a-relay-costs)).

---

## How a request is priced

A request's price is its compute units valued at Pocket's CU peg, a fixed dollar
amount per billion compute units (`peg_micro_usd_per_billion_cu`), plus the
Pocket API Pro service fee (`markup_bps`, the service fee in basis points). The
compute units of one relay are the chain's own figure for that service. The
market price of POKT is not part of it. The peg, the service fee and every
service's price in force are published at
`https://pay.api.pocket.network/v1/prices`.

The peg reaches the price through the chain's multiplier from compute units to
POKT (`cuttm`): the POKT rate the service uses is the peg's,
`peg x granularity / (1000 x cuttm)` micro-USD per POKT, rounded down. So
`cuttm` cancels out, and a relay costs `cu_per_relay x peg / 10^9` micro-USD plus
the service fee, a hair under it where the rate was rounded down. `GET /v1/prices`
publishes every input (`peg_micro_usd_per_billion_cu`, `cu_per_relay`, `cuttm`,
`granularity`, the POKT rate and `markup_bps`), so a price can be recomputed by
hand:

```
price (micro-USD) = cu_per_relay x cuttm / granularity      (upokt per relay)
                    x rate_micro_usd / 10^decimals          (micro-USD per upokt)
                    x (10000 + markup_bps) / 10000
```

When the chain changes one of those parameters, the new value is charged from
the chain session in which the chain applies it. A change of `cuttm` moves the
POKT rate and leaves the price of a compute unit where the peg puts it.

---

## Paying

You top up your **credit** by sending coins on chain:

- in **POKT**, on Pocket Network, or
- in a coin on an EVM chain (its native coin, or a token such as an ERC-20).

Which coins are accepted, the receiving wallet for each, the bonus and the
number of confirmations needed are listed by the console's **Fund** page and by
the public route `https://pay.api.pocket.network/v1/payments/options`
(`https://pay.beta.api.pocket.network/v1/payments/options` on beta). Each row's
`min_confirmations` is the number of confirmations that coin waits for, in
force now.

A **bonus** is extra credit on top of what you paid, given in basis points as
`bonus_bps`: a payment is credited as `value x (10000 + bonus_bps) / 10000`. We
usually offer a bonus when you pay in POKT. The Fund page and the payment
options always show the figures in force.

How a payment gets credited:

1. Send the coins **from your key's own address** to the receiving wallet.
2. Tell the service about the transaction. The console does this for you; with
   the CLI or the API you send the transaction hash.
3. The service reads the transaction on the chain itself, waits for the
   required confirmations, values the coins at the price in force, adds the
   bonus, and adds the result to your credit. POKT is valued at the peg rate,
   the same rate requests are priced at; every other coin at its market price.

The chain is the proof: nothing you tell the service is taken on trust. The
rules in full are in [billing-rules.md](billing-rules.md).
