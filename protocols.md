# One relay per protocol: HTTP, WebSocket, gRPC, SSE

One copy-paste example for each protocol a service can speak. Every example
needs the same two things first, and both are covered elsewhere:

- **An account and a balance**: [index.md](index.md), "Your account is a
  secp256k1 key" and "Your money: three pools".
- **An API token** (`pkt_...`): mint one in the console or with
  `budgetkitctl user mint-token` ([cli.md](cli.md)), then put it in a shell
  variable:

  ```sh
  export TOKEN=pkt_...
  ```

`<service>` is a service name. `GET /v1/prices` on the account API lists them
([api.md](api.md#get-v1prices-what-a-relay-costs)), and
[index.md](index.md#where-to-send-requests) lists the hosts for mainnet and beta.
Every example uses the `Authorization: Bearer` header, the place to prefer. The
other places a token can go are in [api.md](api.md#where-the-token-goes).

What each protocol is billed is the rule in
[api.md](api.md#what-is-billed), summarised under each example. The refusals
are described in [api.md](api.md#errors): `402` with no balance or an unknown
token, `403` from an allowlist, and `400` for two different tokens or a token
offered as a subprotocol without `pocket`.

## HTTP

Any HTTP request the service speaks. For an EVM JSON-RPC service:

```sh
curl -sS -X POST "https://<service>.pro.api.pocket.network/" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

**Billed:** one relay per request the service answers with a `2xx` (or one of
the client errors listed in [api.md](api.md#what-is-billed)). A JSON-RPC batch
of N calls is N relays.

## WebSocket

The token goes in the `Authorization` header of the upgrade request. This uses
[websocat](https://github.com/vi/websocat); any WebSocket client that can set a
header works. It subscribes to new blocks on an EVM service:

```sh
echo '{"jsonrpc":"2.0","id":1,"method":"eth_subscribe","params":["newHeads"]}' |
  websocat -n --text "wss://<service>.pro.api.pocket.network/" -H "Authorization: Bearer $TOKEN"
```

The URL goes before `-H`: websocat's `-H` takes several values, so a URL
after it is read as one more header. `-n` keeps the socket open after the
subscription is sent, so new blocks keep arriving until you press Ctrl-C.

A browser cannot set that header. It offers the token as a subprotocol beside
`pocket` instead: `new WebSocket(url, ["pocket", token])`
([api.md](api.md#where-the-token-goes)).

**Billed:** one relay per data message the service sends you. The messages
you send and control frames are free. When your balance reaches zero, the
message that spent it is billed and the socket is closed.

## gRPC

The token goes in the call's metadata as `authorization`. This uses
[grpcurl](https://github.com/fullstorydev/grpcurl) against a Cosmos service.
It needs server reflection; without it, pass the service's `.proto` files with
`-proto`:

```sh
grpcurl -H "authorization: Bearer $TOKEN" \
  -d '{}' <service>.pro.api.pocket.network:443 \
  cosmos.base.tendermint.v1beta1.Service/GetLatestBlock
```

A browser can call in gRPC-Web (`application/grpc-web+proto` over HTTP/1.1)
instead; the text form, `application/grpc-web-text`, is not served.

**Billed:** one relay per call, however many messages a server stream returns.
The call is billed when it ends with status `OK`, `INVALID_ARGUMENT`,
`NOT_FOUND`, `FAILED_PRECONDITION` or `OUT_OF_RANGE`; any other status is free.

**Refused:** a gRPC call the endpoint refuses ends with a gRPC status, never an
HTTP error, and its message names what to do. None of these is billed:

| Status | Why | The message names |
|---|---|---|
| `UNAUTHENTICATED` | no token, or one nobody knows | where to pay and the free endpoint |
| `RESOURCE_EXHAUSTED` | no balance left, or too many requests | where to pay, or how long to wait |
| `PERMISSION_DENIED` | gRPC is not served on the free endpoint, or not for this service or token | the Pro endpoint, or what is refused |
| `UNIMPLEMENTED` | unknown service, or gRPC-Web text | what is served |
| `UNAVAILABLE` | the service is down or busy | retry |
| `DEADLINE_EXCEEDED` | the service did not answer in time | |

## Server-sent events (SSE)

A request that asks for a stream with `Accept: text/event-stream`. `-N` makes
curl print each event as it arrives:

```sh
curl -sS -N "https://<service>.pro.api.pocket.network/<the service's stream path>" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Accept: text/event-stream'
```

**Billed:** one relay per request, however many events the stream sends. A
stream that goes silent is closed at the deployment's idle timeout and is
still billed once.

## Which services speak what

A service speaks the protocols its own chain or API speaks. On beta, `anvil`
(EVM) answers JSON-RPC over HTTP and WebSocket, and serves no gRPC. `pocket`
answers Cosmos REST, CometBFT RPC, WebSocket, gRPC and gRPC-Web on the Pro
domain. Neither serves SSE. The free endpoint serves no WebSocket on beta, and
no gRPC or SSE; it does serve JSON-RPC batches over HTTP, each call taking one
slot of the free rate limit ([api.md](api.md#the-free-endpoint)). A WebSocket,
gRPC or SSE session also needs that feature enabled for your tier on the
deployment.
