# Reference receiver examples

The reference receiver is domain-neutral. These examples show how to adapt it to
a specific vertical by injecting a **context mapper** — the function that turns
an inbound A2A message into the PortAuth `requestContext` the engine evaluates.

## `bill-payment-mapper.js`

The banking mapper that backs the bundled bill-payment demo
(`reference-presenter/`). It defaults `action`/`resource`/`currency`, derives a
`requestId`, and coerces `claimAmount` to a finite number (rejecting bad input
with HTTP 400 before the engine is called).

Wire it programmatically:

```js
import { createReferenceReceiverServer } from "portauth-a2a-reference-receiver";
import { billPaymentContextMapper } from "portauth-a2a-reference-receiver/examples/bill-payment-mapper.js";

const { server } = createReferenceReceiverServer({ contextMapper: billPaymentContextMapper });
```

…or via the CLI:

```bash
A2A_CONTEXT_MAPPER=bill-payment ENGINE_URL=http://localhost:3200 npm run receiver
```

Without `A2A_CONTEXT_MAPPER`, the receiver uses `defaultRequestContextMapper`,
which forwards the message's structured data part unchanged. The bill-payment
demo passes under the neutral default too, because the presenter already sends a
complete `requestContext`; the example mapper exists to show domain defaulting,
coercion, and validation.

## Writing your own

A context mapper receives `{ message, body, portauth }` and returns (or resolves
to) a `requestContext` object. It must include string `action` and `resource`;
numeric fields must be finite. Throw an `Error` (optionally with a `status`
property) to reject malformed input before evaluation.
