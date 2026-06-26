# Portable Authorization Engine (PortAuth)

![GitHub issues](https://img.shields.io/github/issues/kyndryl-open-source/aiagent-portable-authorization)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/kyndryl-open-source/aiagent-portable-authorization)
![GitHub top language](https://img.shields.io/github/languages/top/kyndryl-open-source/aiagent-portable-authorization)
![GitHub contributors](https://img.shields.io/github/contributors/kyndryl-open-source/aiagent-portable-authorization)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

**Policy-embedded credential authorization for AI agents.** Issue signed credentials with machine-evaluable constraints, verify them at runtime, and enforce allow/deny decisions with a signed audit trail.

PortAuth is the reference implementation of the three-layer authorization architecture described in the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)): a Container layer (JWT, X.509 JWT, VC-JOSE), a Payload layer (permissions plus typed constraints), and an Engine layer (conjunctive, total, fail-closed evaluation, most-restrictive-wins local policy merge, signed audit).

---

## Quick Start

```bash
git clone https://github.com/kyndryl-open-source/aiagent-portable-authorization.git
cd aiagent-portable-authorization && ./quickstart.sh
```

`quickstart.sh` brings up the full stack (PostgreSQL, issuer, engine, OPA sidecar) with `docker compose` or `podman compose`, waits for health, then runs the end-to-end smoke flow and prints a PASS/FAIL summary. Requires Docker Desktop or Podman 5+.

Detailed setup, manual compose commands, and local-dev (no compose) options live in [docs/operations.md](docs/operations.md#running-locally-without-compose).

---

## Use case

A claims authority issues a credential to a **Damage Assessment Agent** authorizing it to read claim data under specific conditions:

- **Claim amount** ≤ $5,000 (numeric limit)
- **Claim type** must be `rear_end` or `collision` (enumerated list)
- **Resource path** must match `/claims/CLM-2026-*` (string pattern)
- **Time window** weekdays 8 AM to 6 PM CT (temporal window)

If the agent requests access to a $7,000 claim, outside business hours, or for an unsupported claim type, the request is denied, even if the receiving system would otherwise allow it. The credential defines the ceiling.

---

## Documentation

| Document | What's inside |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System diagram, component layout, full database schema (11 migrations) |
| [docs/api-reference.md](docs/api-reference.md) | All issuer and engine HTTP endpoints, status semantics, error codes, full end-to-end `curl` walkthrough |
| [docs/credentials-and-constraints.md](docs/credentials-and-constraints.md) | Tier 1 / Tier 2 / Tier 3 credential profiles, normalization, all six constraint types, attenuation rules |
| [docs/evaluation.md](docs/evaluation.md) | Six-step evaluation pipeline, audience and proof-of-possession, workflow combinators, governance manifest, preflight SDK, state authority |
| [docs/operations.md](docs/operations.md) | Logging, testing (unit + conformance + e2e + black-box), configuration reference, feature status, known limitations |
| [docs/paper-traceability.md](docs/paper-traceability.md) | Every paper requirement (BR-1 through BR-10 plus Sections 7.5.1 and 8.5) traced to source and tests |

---

## Extensions

| Extension | What's inside |
|---|---|
| [extensions/a2a](extensions/a2a/README.md) | Carry PortAuth credentials over Google A2A (Agent2Agent). Spec, caller SDK, framework-neutral receiver guard, and reference HTTP+JSON agents. |
| [extensions/a2a/SPEC.md](extensions/a2a/SPEC.md) | Normative A2A extension specification (metadata placement, delegation chains, error mapping). |
| [extensions/a2a/OPERATIONS.md](extensions/a2a/OPERATIONS.md) | A2A receiver production runbook: secret handling, rotation, monitoring, alerts, reconciliation. |
| [extensions/a2a/TLS.md](extensions/a2a/TLS.md) | Local nginx TLS reverse-proxy example for the A2A receiver. |

---

## Project hygiene

- [CONTRIBUTING.md](CONTRIBUTING.md): development setup, PR checklist, DCO sign-off (embedded text).
- [DCO](DCO): canonical Developer Certificate of Origin 1.1 text.
- [SECURITY.md](SECURITY.md): vulnerability reporting, supported versions, hardening checklist.
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md): Contributor Covenant 2.1.
- [CHANGELOG.md](CHANGELOG.md): release notes.

Before deploying beyond a local laptop, replace every value flagged in [SECURITY.md](SECURITY.md), terminate TLS in front of both services, and rotate the audit and governance signing keys.

---

## License

MIT. See [LICENSE.md](LICENSE.md).
