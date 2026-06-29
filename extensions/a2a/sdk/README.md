# portauth-a2a

Caller-side (presenter) helpers for carrying [PortAuth](https://kyndryl-open-source.github.io/aiagent-portable-authorization/)
authorization credentials over the [A2A](https://github.com/google-a2a/A2A) HTTP+JSON
protocol. This is the SDK an **agent that initiates a task** uses to attach a Verifiable
Credential, build a conformant `message:send` request, and map engine decisions to A2A
errors.

> Status: pre-1.0. The **wire format** (extension URI, metadata keys, activation header)
> is frozen at the `v1` URI — see [VERSIONING.md](../VERSIONING.md). The JavaScript API may
> still change before `1.0.0`.

## Install

```sh
npm install portauth-a2a
```

Requires Node.js >= 20 (uses the global `fetch` and `node:crypto`).

## Wire constants (frozen)

```js
import { EXTENSION_URI, A2A_VERSION, A2A_MEDIA_TYPE, VC_FORMATS, metadataKeys } from "portauth-a2a";

EXTENSION_URI;  // "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1"
A2A_VERSION;    // "1.0"
A2A_MEDIA_TYPE; // "application/a2a+json"
VC_FORMATS;     // ["tier1-jwt", "tier2-x509-jwt", "tier3-vc-jose"]
metadataKeys;   // { vc, vcFormat, presenterId, proofOfPossession, idempotencyKey, delegationChain }
```

The receiver advertises support via the Agent Card and the `A2A-Extensions` activation
header set to `EXTENSION_URI`.

## Quick use

```js
import { sendWithVc } from "portauth-a2a";

const result = await sendWithVc({
  receiverUrl: "https://receiver.example",
  credential: vc,           // the Verifiable Credential (Tier1/2/3)
  vcFormat: "tier1-jwt",
  presenterId: "agent:customer-finance-1",
  text: "Pay my electric bill for 2400 USD.",
  requestContext: { action: "payments:bill_pay", resource: "urn:bank:utilities/electric" },
  authorization: "Bearer receiver-endpoint-token", // if the Agent Card advertises bearer auth
});
```

See the [extension SPEC](https://github.com/kyndryl-open-source/aiagent-portable-authorization/blob/main/extensions/a2a/SPEC.md)
and the [reference presenter](https://github.com/kyndryl-open-source/aiagent-portable-authorization/tree/main/extensions/a2a/reference-presenter)
for an end-to-end example.

## License

MIT — see [LICENSE](./LICENSE).
