# PortAuth A2A Extension v1

Status: Draft. Co-located with the engine while the spec stabilizes; will move
to its own publishable package once v1 is frozen.

| File | Purpose |
|---|---|
| [SPEC.md](SPEC.md) | Normative specification of the extension. |
| [poc/run.js](poc/run.js) | Minimal proof-of-concept exercising UC-03 against the live engine. |

## Quick start

```bash
# 1. Bring up the standard compose stack (postgres + issuer + engine).
podman compose -f portauth-docker-compose.yml up -d --build

# 2. Run the PoC.
node extensions/a2a/poc/run.js
```

Expected output:

```
=== 4/4 scenarios passed ===
```

The PoC:

1. Registers the issuer with the engine and trusts it for `payments:bill_pay`.
2. Issues a UC-03 VC at the issuer with three constraints (amount cap,
   recipient enum, geo enum).
3. Wraps the VC in an A2A-shaped `message/send` JSON-RPC payload under
   `message.metadata["<extension-URI>/vc"]`.
4. Simulates an A2A receiver: extracts the VC from metadata and calls
   `/evaluate` with a translated request context.
5. Runs four scenarios: one allow + three denies (over-cap amount,
   off-list recipient, off-list geo region).

## Roadmap

| Phase | Deliverable | Status |
|---|---|---|
| 0 | SPEC.md + PoC | done |
| 1 | `sdks/a2a/` caller SDK with `preflight()` and `sendWithVc()` | planned |
| 2 | `sdks/a2a-receiver/` Express/Fastify middleware | planned |
| 3 | `extensions/a2a/reference-presenter/` and `reference-receiver/` minimal agents wired to Google ADK | planned |
| 4 | Conformance test suite under `tests/conformance/a2a/` | planned |

## Related work

The metadata-keyed A2A extension pattern was pioneered by the Identity
Machines Iron Book extension (`https://github.com/identitymachines/a2a_ironbook`).
This extension adopts the same A2A compliance recipe and differs in credential
format, delegation semantics, and trust model. See SPEC.md §10 for a factual
comparison.
