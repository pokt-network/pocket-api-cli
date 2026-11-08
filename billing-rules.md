# Payment rules: what is credited and what is not

Your account is funded by sending coins on chain to the receiving wallet the
deployment publishes, and telling the API the transaction's hash. This page is
every rule that decides whether that transfer becomes credit, in plain terms.
The receiving wallets, the coins accepted, their bonus and the confirmations
each network waits for are listed by `GET /v1/payments/options`, by the Fund
page of the console, and by `budgetkitctl user payments`.

## The one rule behind all the others

**A payment is credited only to the account whose key sent it.** Your account
IS your key: its Pocket address and its EVM address are both derived from it.
A transfer from any other address is not yours to claim, even if you notify it.

## Credited

| You do this | You get |
|---|---|
| Send an accepted coin from your own address to the receiving wallet, then notify the hash | Credit at the coin's price once the transaction has the network's confirmations |
| Send POKT on Pocket (a bank `MsgSend`) | Credit for the POKT, plus the POKT bonus the options list (+20% on beta) |
| Send several transfers to us in one Pocket transaction, or a `MsgMultiSend` whose outputs include us | The sum of everything that reached the receiving wallet |
| Send an accepted ERC-20 token (its contract is in the options) | Credit for the tokens the transfer moved to us |
| Send ETH (or the network's native coin) through a contract, for example a batching wallet, funded only by your own account | Credit for everything that reached the receiving wallet |
| Pay from a wallet that pays your gas for you (MetaMask's smart account) | Credited: what counts is whose balance paid, not who signed the transaction |
| Notify a hash late, hours or days after paying | Credited all the same: the chain decides, not the time you notified |
| Notify the same hash twice | Credited once |

**How much:** the amount times the coin's price when the transaction is
credited, plus its bonus, rounded **down** to the micro-dollar. A transaction is
decided whole: if any coin in it cannot be credited, none of it is.

## Not credited

| What happened | Status you see | What to do |
|---|---|---|
| The coins came from an address that is not your key's | `rejected`, `not_the_sender` | Pay from the wallet holding your key |
| Nothing in the transaction reached the receiving wallet | `rejected`, `not_ours` | Check the address you paid; notifying the hash again reads it again |
| The transaction pays us in a coin the network does not accept (alone or beside an accepted one) | `rejected`, `coin_not_accepted` | Pay in a listed coin, in its own transaction |
| The transaction failed or reverted on chain | `rejected`, `execution_failed` | Nothing moved; pay again |
| Payments to us in one transaction come from two different senders | `rejected`, `two_senders` | One sender per transaction |
| The coin has no price at the moment of the credit | `rejected`, `no_price` | Notify the hash again later: your notice reopens it |
| The amount is worth less than one micro-dollar | `rejected`, `credit_zero` | Send more than that |
| The amount is worth more than one credit can hold | `rejected`, `credit_overflow` | Pay in smaller amounts; this one can be returned, see [Returns](#returns) |
| You sent on a network the Fund page does not list for that coin | nothing is recorded: we read only the listed networks | Pay on a listed network; for the coins already sent, see [Returns](#returns) |
| ETH reached us from a balance other than your account's: a Safe or other contract wallet, a vault or exchange withdrawal, a faucet, a swap paying out ETH | `rejected`, `not_the_sender` | Send ETH from your own account. The money stays in our wallet; it can be returned, see [Returns](#returns) |
| ETH reached us in a transaction funded by two or more addresses (your value sent elsewhere while somebody else paid us, a third party paying anybody), or in which a `SELFDESTRUCT` runs | `rejected`, `unattributable` | Send plain ETH from your own account, or through a contract funded only by that send, or pay in a token. It can be returned, see [Returns](#returns) |
| A message of another module that only looks like a transfer (not the bank module's) | not counted | Use a plain bank send |
| The transaction was never in a block within the waiting time (24 hours after the notice, unless the deployment sets another time) | `not_found` | Notify the hash again once it is mined: that reopens it |
| Its block was replaced by a chain reorganisation | `reorged` | Notify again; if it landed in a new block, it is read again |
| You paid and never notified | nothing is recorded | Notify the hash; nothing is credited until you do |

## Paying from an exchange or a contract wallet

Pay from the wallet you sign in to the console with: its key is your account,
and only what leaves that address is credited to you. An exchange withdrawal
leaves the exchange's shared hot wallet, often in one transaction with other
customers' withdrawals, so nothing on chain says it was yours and it is never
credited to anyone. Withdraw to your own wallet first, then pay from it and
notify the hash. The same goes for a Safe or any other contract wallet: move the
funds to your signing wallet, then pay.

## Returns

Money that reached our wallet and was not credited is never returned
automatically. Ask on the Pocket Network Discord,
https://discord.gg/pocket-network, with the chain and the transaction hash.
Every return is reviewed and approved by Pocket Network case by case; an
approved return goes to the address the chain shows the money came from, minus
the network fee of sending it back.

## Good to know

- **Notifying is how a payment is matched to you.** The console notifies for you
  when you pay from the Fund page; from any other wallet, paste the hash under
  **Already paid from another app?** on the Fund page, or use
  `budgetkitctl user notify-payment`.
- **There are no refunds of on-chain payments through the API**: credit stays
  on the account. Overpaying is credit you keep.
- **Confirmations protect you and us from reorganisations.** A payment is
  credited only after the network's confirmations; until then it shows
  `verifying` or `waiting_confirmations`.
- **Every status is visible**: `GET /v1/payments/{chain}/{tx_hash}`, the Fund
  page of the console, or `budgetkitctl user payments status`.
