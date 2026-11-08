# Pocket API Console: CLI and documentation

Paid, metered access to the services Pocket Network serves: blockchain RPC and
the other APIs the deployment lists. Your account is a secp256k1 key, the same
kind Pocket and Ethereum wallets use, so there is no email or password; you pay
in POKT or in EVM coins verified on chain.

This repository publishes two things:

- **`budgetkitctl`**, the command-line tool, on the
  [Releases](https://github.com/pokt-network/pocket-api-cli/releases) page:
  Linux, macOS and Windows on x86-64 and ARM64, each release signed.
- **The documentation**, the pages below.

## Start here

| you are | read |
|---|---|
| asking a quick question | [faq.md](faq.md): short answers, each pointing to its page |
| new to the console | [index.md](index.md): accounts, balances, tokens, endpoints |
| using a browser | [console.md](console.md) |
| using a terminal | [cli.md](cli.md) |
| writing code against the API | [api.md](api.md): request signing, relays, error codes |
| an AI agent, or automating it | [agents.md](agents.md): one runbook, every command in order |
| paying | [billing-rules.md](billing-rules.md): which payments are credited |

## Endpoints

| what | production | beta |
|---|---|---|
| Web console | https://console.api.pocket.network | https://console.beta.api.pocket.network |
| Paid requests, with a token | `https://<service>.api.pocket.network` | `https://<service>.beta.api.pocket.network` |
| Free requests, no token | `https://<service>.free.api.pocket.network` | `https://<service>.free.beta.api.pocket.network` |

## Install the CLI

Download your binary and `SHA256SUMS` plus `SHA256SUMS.asc` from the release,
then check the signature before running anything. The release key is
[`release-key.asc`](release-key.asc), fingerprint
`03E7 C270 9090 327D F488  6C02 8B4B EE82 473C 7CB9`:

```sh
gpg --import release-key.asc
gpg --fingerprint developer@pokt.foundation   # must show the fingerprint above
gpg --verify SHA256SUMS.asc SHA256SUMS        # must say "Good signature"
sha256sum -c SHA256SUMS --ignore-missing
```

[cli.md](cli.md#install) has the full steps for every system.

## Help

Ask on the Pocket Network Discord, https://discord.gg/pocket-network.
[faq.md](faq.md) says what to include, and what never to share.
