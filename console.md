# The web console

The Pocket API Console (`https://console.api.pocket.network`, and
`https://console.beta.api.pocket.network` on beta) is where you manage your API
account in a browser: see your balance, create API tokens for your apps, and top
up from your wallet. There is no password and no email. You sign in with a
crypto wallet, and the key in that wallet *is* your account.

The top bar has four pages: **Account**, **Tokens**, **Fund** and **Sessions**.
It also shows the address you signed in with (hover it to see your account id),
a light/dark theme switch, and the sign-out button.

---

## Signing in

1. Open the console. It looks for wallet extensions installed in your browser.
2. Pick your wallet. This release has been tested with:
   - **Soothe Vault**, for a Pocket account (and its EVM account);
   - **MetaMask**, for an EVM account.

   The console also lists other wallets it detects in the browser; those have
   not been tested. A wallet that offers both a Pocket and an EVM account asks
   **Which account?** with a **Pocket** and an **EVM** button. If no wallet is
   found, the console says so and offers to look again after you install one.
3. If the wallet shares more than one account with the console, choose the one
   to sign in with. The one you used last is marked **last used**. The account
   you choose is the account the console manages and pays from.
4. Your wallet asks you to sign a message. It is plain text, like this:

   ```
   console.api.pocket.network wants you to sign in with your Pocket account:
   pokt1...your address...

   Sign in to the Pocket Network console

   URI: https://console.api.pocket.network
   Version: 1
   Chain ID: <the chain id>
   Nonce: <a one-time random value>
   Issued At: <now>
   Expiration Time: <five minutes from now>
   ```

   It names the site, your address, a one-time random value and the time.
   **Signing it is not a transaction: nothing is sent on chain and no fee is
   paid.** The message is valid for five minutes, so sign it promptly.
5. You land on the Account page. How long you stay signed in is set by the
   deployment. When the session ends, the console takes you back to the sign-in
   page with **Your session ended. Sign in again.**

Signing in does not ask your wallet to switch networks.

**Your account is your secp256k1 key.** The same key has a Pocket address (`pokt1...`)
and an EVM address (`0x...`), and both lead to the same account. Signing in with
the Pocket account of a key or with its EVM account opens the same console.

If something goes wrong, the console explains it in a short note:

| you see | what happened |
|---|---|
| You cancelled the request in your wallet. | you rejected the signature; nothing was signed or sent |
| Your wallet already has a request open. | open the extension and approve or reject the pending request |
| Your wallet did not answer in time. | the wallet may be locked; unlock it and try again |
| Your wallet has not allowed this site to use that account or network. | approve the site for that account or network in the wallet |
| a message from the server, with "api code" and a number | the server refused the request; the code is listed in [api.md](api.md#errors) |

---

## Account

**Spendable balance.** The large figure at the top is what you can spend now,
in US dollars, with your current tier ("spendable on the trial tier" or
"spendable on the paid tier"). The thin bar under it shows how the three pools
make up that total. This part of the page refreshes every 15 seconds.

**The three pools.** Your money sits in up to three pools, and requests are paid
from them in this order: **trial first, then plan, then credit**.

| card | what it is | what the card says |
|---|---|---|
| **Trial** | a free daily allowance, the same for everybody | when it refills ("refills in 5h") |
| **Plan** | an amount arranged with Pocket Network that refills on its own schedule | the next refill date, or "no refill contract" |
| **Credit** | money you paid in | "paid in; never expires" |

If your balance is getting low for some services, a yellow note names them:
"The balance is below the low watermark for ... Fund the account to keep those
services at full speed."

**Spend.** A chart of what you spent, by service, in your local time. Choose the
range (**24h**, **48h**, **7d**, **30d**) and the chart type (**Bars** or
**Lines**). Bars are per hour for 24h and 48h, per 6 hours for 7 days and per
day for 30 days. Up to five services get their own colour; the rest are grouped
as "Other". Hover or tab to a bar to see the amounts.

**Usage by service.** A table of each service with the number of **Relays**
and their **Cost**. It starts empty: "No relays yet. Send one with a token and
it shows here as soon as it is charged." Services are named here by their
Pocket service id, which can differ from the name in the request's address (on
beta, `anvil` is billed as `pnf-anvil` and `pocket` as `pnf-pocket-beta`).

**Activity.** Everything that changed your balance, newest first, with
**When**, **What**, **Change** (the amount added or removed) and **By**.

| what | meaning |
|---|---|
| Account created, trial granted | your first use of the account |
| Daily trial refilled | the daily trial came back |
| Plan started / Plan refilled / Plan cancelled | changes to a plan |
| Refund | Pocket Network returned money to one of your pools |
| Payment credited | a payment you made was added to your credit (with the bonus, when there is one) |

**By** is "automatic" for things the system does on its own, and "Pocket
support" for actions taken by Pocket Network staff. Click **More** to load older
entries. The usage and activity tables load when you open the page.

---

## Tokens

API tokens are what your applications send with each request. Each token
belongs to your account, and you can hold several (one per app is a good
habit).

**Mint a token.** Type a name (for example "production backend"), choose how
long it lives (**Never expires**, **Expires in 7 days**, **Expires in 30
days**, **Expires in 90 days** or **Expires in 1 year**), and click **Mint
token**. Your wallet asks you to sign a short message naming the token: a token
outlives your session, so minting needs a fresh signature. A dialog then shows
the token:

- **Copy it now.** It is shown once. The server keeps only a fingerprint (hash)
  of it and cannot show it again. If you lose it, mint another.
- Use it in your app as the header `Authorization: Bearer <token>` on the paid
  domain, `https://<service>.api.pocket.network`.
- Click **I have saved it** to close the dialog.

An account holds at most a fixed number of live tokens (100 unless the
deployment sets another number). At the limit the console says "This account
already holds its maximum of live tokens. Revoke one to mint another."

**Your tokens** are listed with **Id**, **Name**, **Created**, **Expires** and
**State**: **live**, **expired** or **revoked**.

**Revoke a token.** Click **Revoke** on a live token, confirm, and sign the
message your wallet shows. Requests carrying that token stop being accepted
within seconds. This cannot be undone; mint a new token to replace it.

---

## Fund

Top up your Credit pool, and follow the payments you made.

**Where to pay.** A table of every coin you can pay with: **Chain**, **Coin**
(marked "token" for a token such as an ERC-20), **Receiving wallet**, **Rate**,
**Bonus** and **Confirmations** (how many blocks must pass before it is
credited). Only coins the service can price right now are listed. Each
receiving wallet has a copy button and a QR code of the address, for paying
from another app.

**Bonus.** Some coins earn extra credit. When one does, the page says so at the
top, for example "Pay in POKT and get +20% more credit." A +20% bonus means a
payment worth $10 is credited as $12. The page always shows the bonuses in
force.

**Paying from the connected wallet:**

1. Click a row in **Where to pay**.
2. Connect the wallet account you signed in with. Only wallets that can pay that
   coin are offered: a Pocket coin needs a Pocket wallet, an EVM coin an EVM
   wallet.
3. **The paying account must be the one you signed in with.** Only a transfer
   sent from your key's own address is credited to you. If the wallet connects a
   different account, the console stops and tells you which address to select.
4. Enter the amount, in US dollars or in the coin (for a token whose decimals are
   not known, also enter the **Token decimals**). The console asks the service
   for a quote and shows what the amount is credited as, bonus included. The
   quote is an estimate: a payment is valued at the price in force when it is
   credited.
5. Click **Send and notify**. Your wallet may ask to switch to the payment's
   network, then asks you to approve the transfer. This one is a real
   transaction and pays the network fee. The wallet broadcasts it and the
   console tells the service its hash.
6. A status row appears and updates every 5 seconds until it is **credited**,
   **rejected** or **reorged**. Once credited, the Account page shows the new
   Credit.

**Paying from elsewhere.** You can send the transfer from any wallet or app, as
long as it is sent **from your key's own address** to the receiving wallet
shown. Then, under **Already paid from another app?**, choose the chain, paste
the transaction hash and click **Track payment**: the console notifies it for
you. The CLI (`budgetkitctl user notify-payment`, see [cli.md](cli.md)) and the
API (`POST /v1/payments/notify`, see [api.md](api.md)) do the same.

**Your payments.** Below, every payment your account notified, newest first,
refreshed every 15 seconds. Columns: **Date** (when it was notified),
**Chain**, **Hash**, **State**, **Reason** and **Credited** (the dollar amount
added, plus the bonus you earned, when there was one). Payments notified by
other accounts are never shown to you.

| state | meaning |
|---|---|
| to deliver | a signed transaction was handed to the service and has not been broadcast yet |
| verifying | the service is looking for your transaction on the chain |
| waiting confirmations | found; waiting for the chain to confirm it |
| confirmed | confirmed; the credit is being added |
| **credited** | done: the money is in your Credit pool |
| rejected | not a valid payment; the **Reason** column says why |
| reorged | the block holding it disappeared from the chain |
| not found | the chain never showed the transaction (24 hours after the notice, unless the deployment sets another time) |

The **Reason** column shows the sentence the service gives, for example:

| reason | what to do |
|---|---|
| The payment was sent by another account, and is credited to whoever sent it. | pay from the wallet account you signed in with |
| The transaction pays nothing to our wallet. | send to the wallet shown in **Where to pay** |
| The transaction pays our wallet in a coin this network does not accept. | pay with a listed coin |
| The transaction failed on chain, so nothing was paid. | pay again |
| The payment could not be priced; notify it again later. | track the same hash again later |
| The payment is worth less than the smallest amount credited, so nothing was credited. | pay a larger amount |

The full list is in [api.md](api.md#payments-codespace), and what each rule
means is in [billing-rules.md](billing-rules.md).

---

## Sessions

The browser sessions open on your account: **This browser** and any other
sign-in, each with when it expires. **Sign out everywhere** ends all of them
after you confirm. Your API tokens are not affected: revoke a token on the
Tokens page.
