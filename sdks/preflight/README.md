# @portauth/preflight

Sender-side discovery client for the portable authorization governance
manifest. Implements Plan 8 of `portauth_refimpl_features.md`.

## API

```js
import { preflightCheck } from "@portauth/preflight";

const result = await preflightCheck({
  manifestUrl: "https://engine.example.com/.well-known/agent-governance",
  signingKeyUrl: "https://engine.example.com/.well-known/agent-governance/signing-key",
  credential: "<compact JWS / vc+jwt>",
  requestContext: { action: "claims:read", resource: "/claims/123" },
});

if (!result.compatible) {
  for (const i of result.incompatibilities) {
    console.warn(`[${i.severity}] ${i.rule}: ${i.detail}`);
  }
}
```

## CLI

```bash
portauth-preflight \
  --manifest https://engine.example.com/.well-known/agent-governance \
  --signing-key-url https://engine.example.com/.well-known/agent-governance/signing-key \
  --credential ./cred.jwt
```

Exit codes:

| Code | Meaning              |
|------|----------------------|
| 0    | compatible           |
| 2    | incompatible         |
| 1    | operational failure  |

## Rules

* `profileMatch` (error)         credential profile must appear in `manifest.credentialProfiles`
* `constraintTypes` (error)      every constraint type must appear in `manifest.constraintTypes`
* `vocabulary` (error)           every reserved-prefix field must be in `manifest.recognizedFields` or a `manifest.mappingProfiles[*].fields` entry
* `trustAnchors` (warning)       credential issuer should be pre-registered for its tier
* `requiredContextFields` (error) the caller-supplied `requestContext` must include every `manifest.requiredContextFields` entry

## Tests

```bash
cd sdks/preflight
node --test test/*.test.js
```
