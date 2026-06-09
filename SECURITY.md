# Security Policy

PortAuth is a reference implementation of the policy-embedded authorization architecture described in *Digital Identity for Agentic Systems*. Because the codebase implements credential issuance, verification, and access decisions, we take security reports seriously and we ask reporters to follow the coordinated disclosure process below.

## Supported versions

| Version | Status | Notes |
|---------|--------|-------|
| `0.x`   | Active | Pre-1.0 reference implementation. Security fixes land on `main`. |

Tagged releases predate `1.0.0` and are not considered production-ready. Operators MUST treat the included development credentials, signing keys, and database passwords as compromised and replace them before any non-toy deployment.

## Reporting a vulnerability

If you believe you have found a security issue, please **do not open a public GitHub issue**. Instead, email the maintainers privately:

- `kyndryl-portauth-security@kyndryl.com` (preferred)
- Optionally, encrypt the report with the maintainer PGP key published in this repository's GitHub Security advisories page.

Please include:

1. A description of the issue and the impact you believe it has.
2. The component affected (`portauth-engine`, `portauth-issuer`, `state-authority-ref`, `sdks/preflight`, schema migrations, compose file, etc.).
3. Steps to reproduce, ideally with a minimal failing test case.
4. Any relevant logs, payloads, or proof-of-concept code.

We aim to acknowledge new reports within **5 business days** and to publish a fix or mitigation within **90 days** of the initial report, in line with standard coordinated disclosure timelines. We will credit reporters in the release notes unless they ask to remain anonymous.

## Scope

In scope:

- Authorization bypass, signature forgery, or replay in any of the three credential tiers (`tier1-jwt`, `tier2-x509-jwt`, `tier3-vc-jose`).
- Audit-record forgery, tampering, or suppression.
- Constraint evaluation soundness bugs (a constraint that should deny but allows).
- Attenuation widening bugs in delegation chains.
- Stateful-counter under-counting that lets a cumulative or usage limit be exceeded.
- Injection, deserialization, or path traversal in the issuer, engine, state-authority reference, or preflight SDK.

Out of scope:

- Vulnerabilities in third-party dependencies that are already published with a CVE and a fix; please open a PR bumping the version instead.
- Issues that require physical access to a running PortAuth host or root access to its database.
- Defects in the included OPA advisory policy that affect only the advisory decision, not the engine's authoritative decision.
- DoS via unauthenticated traffic against ports `3100`/`3200`/`3300`/`8181`; the reference compose file is for local development and is not hardened for hostile networks.

## Hardening guidance for operators

The compose file `portauth-docker-compose.yml` ships with development defaults that MUST be replaced before any deployment beyond a local laptop:

- `ADMIN_API_KEY` values (`dev-issuer-key-change-me`, `dev-engine-key-change-me`).
- PostgreSQL credentials (`postgres/postgres`).
- The default Tier 1 signing key generated on first start; rotate before issuing real credentials.
- Plain HTTP between services; terminate TLS in front of every component.

Additional recommendations live in the `Configuration Reference` section of [`README.md`](README.md#configuration-reference).
