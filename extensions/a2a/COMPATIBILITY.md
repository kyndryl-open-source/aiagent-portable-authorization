# PortAuth A2A Compatibility Matrix

This extension targets the A2A HTTP+JSON binding. It does not claim universal
A2A transport coverage.

## A2A transport bindings

| A2A binding | Status | What is covered |
|---|---|---|
| HTTP+JSON | Supported and tested | Agent Card discovery, `POST /message:send`, `A2A-Extensions`, PortAuth metadata carriage, allow/deny error mapping, bearer endpoint auth discovery |
| JSON-RPC | Not implemented | Use an adapter that translates JSON-RPC `message/send` calls to the documented HTTP+JSON receiver surface, or contribute a JSON-RPC receiver binding |
| gRPC | Not implemented | Use a service-side adapter or implement a gRPC binding against `portauth-a2a-receiver` |
| Streaming | Not implemented | Agent Card advertises `streaming: false`; requests are evaluated synchronously before task acceptance |
| Push notifications | Not implemented | Agent Card advertises `pushNotifications: false` |

## Wire compatibility

The frozen PortAuth extension URI is:

```text
https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1
```

Any implementation that uses the same URI, metadata keys, VC formats, activation
header, and error mapping can interoperate over the supported HTTP+JSON binding.
See [VERSIONING.md](VERSIONING.md) for the package-version matrix and wire-format
change rules.

## Seller-safe wording

Use:

> PortAuth A2A v1 supports the A2A HTTP+JSON binding and provides reusable
> JavaScript SDK and receiver guard packages for carrying and enforcing portable
> authorization credentials.

Avoid:

> PortAuth A2A supports every A2A transport binding.

> PortAuth A2A is certified against all A2A clients.

