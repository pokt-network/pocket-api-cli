# Pocket API Console

The Pocket API Console is how you buy and manage paid access to the services
Pocket Network serves: blockchain RPC and the other APIs listed by the
deployment. Behind it is a billing system that keeps your balance, charges each
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

## Your account is a key

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

The trial amount is set by each deployment. The console's Account page shows
your trial pool and when it refills.

---

## Tiers: what happens as you spend

| tier | when | what you get |
|---|---|---|
| **paid** | you have plan or credit money left | the paid rate limits |
| **trial** | only trial money is left | service continues, at the trial rate limits |
| **out of money** | nothing left in any pool | paid requests are refused with HTTP `402 Payment Required` |

A `402` response tells you how to get going again: it lists where and how to
pay, and names the free endpoint you can use meanwhile. Running out of money is
never answered with `429 Too Many Requests`, because waiting does not help.

The rate limits of each tier are deployment configuration. Every request served
on the paid domain carries `X-RateLimit-Limit`, `X-RateLimit-Remaining` and
`X-RateLimit-Reset` headers, and a `Pocket-Tier` header naming the tier it was
served at ([api.md](api.md#relays)).

---

## API tokens

Your applications do not use your private key. They use **API tokens**:

- You create ("mint") a token in the console or with the CLI. It looks like
  `pkt_...`.
- Your app sends it with every request, in the standard header
  `Authorization: Bearer <token>`.
- **The token is shown only once**, when you create it. The service stores only
  a fingerprint (a hash) of it. Lost it? Mint a new one.
- You can hold many tokens (one per app is a good habit), give each a name and
  an optional expiry date, and revoke any of them at any time.

All tokens draw from the same account balance.

---

## Where to send requests

Production:

| what | address |
|---|---|
| Paid requests, with a token | `https://<service>.api.pocket.network` |
| Free requests, no token, lower limits | `https://<service>.free.api.pocket.network` |
| Account API (balance, tokens, payments) | `https://pay.api.pocket.network` |
| Web console | `https://console.api.pocket.network` |

Beta:

| what | address |
|---|---|
| Paid requests, with a token | `https://<service>.beta.api.pocket.network` |
| Free requests, no token | `https://<service>.free.beta.api.pocket.network` |
| Account API | `https://pay.beta.api.pocket.network` |
| Web console | `https://console.beta.api.pocket.network` |

`<service>` is the name of the service as the deployment publishes it. On beta
the services are:

| service | what it is |
|---|---|
| `anvil` | an EVM chain: JSON-RPC over HTTP and WebSocket |
| `pocket` | the Pocket beta chain (`pocket-lego-testnet`): Cosmos REST, CometBFT RPC and WebSocket |

The free tier on beta allows 100 requests per minute per client IP address, and
serves no WebSocket.

**Beta and production are separate deployments** with separate databases. A key
works on both, but its beta balance, tokens and payments are not its production
ones. Every signed request names the deployment it is for, so a request signed
for one is refused by the other.

The price of one request for each service is public: `GET /v1/prices` on the
account API ([api.md](api.md#get-v1prices-what-a-relay-costs)).

---

## How a request is priced

A request's price comes from the Pocket chain's own parameters: the compute
units the chain assigns to one relay of that service, and the chain's multiplier
from compute units to POKT. That amount of POKT is valued at the live POKT
price, and a markup is added. `GET /v1/prices` publishes every input
(`cu_per_relay`, `cuttm`, `granularity`, the POKT rate and `markup_bps`), so a
price can be recomputed by hand:

```
price (micro-USD) = cu_per_relay x cuttm / granularity      (upokt per relay)
                    x rate_micro_usd / 10^decimals          (micro-USD per upokt)
                    x (10000 + markup_bps) / 10000
```

When the chain changes one of those parameters, the new value is charged from
the chain session in which the chain applies it.

---

## Paying

You top up your **credit** by sending coins on chain:

- in **POKT**, on Pocket Network, or
- in a coin on an EVM chain (its native coin, or a token such as an ERC-20).

Which coins are accepted, the receiving wallet for each, the bonus and the
number of confirmations needed are listed by the console's **Fund** page and by
the public route `GET /v1/payments/options`. On beta, measured on 2026-11-08:

| chain | coin | confirmations | bonus |
|---|---|---|---|
| `base-sepolia` (Base Sepolia, chain id 84532) | ETH | 3 | none |
| `base-sepolia` | USDC (ERC-20) | 3 | none |
| `pocket-beta` (chain id `pocket-lego-testnet`) | POKT | 1 | +20% |

A **bonus** is extra credit on top of what you paid: with +20%, a payment worth
$10 is credited as $12. The Fund page and the payment options always show the
figures in force.

How a payment gets credited:

1. Send the coins **from your key's own address** to the receiving wallet.
2. Tell the service about the transaction. The console does this for you; with
   the CLI or the API you send the transaction hash.
3. The service reads the transaction on the chain itself, waits for the
   required confirmations, values the coins at the price in force, adds the
   bonus, and adds the result to your credit.

The chain is the proof: nothing you tell the service is taken on trust. The
rules in full are in [billing-rules.md](billing-rules.md).
