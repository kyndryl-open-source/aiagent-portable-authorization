# Operations

## Logging and observability

PortAuth uses **Pino** for structured JSON logging. This allows the "story" of the authorization process to be traced step-by-step from issuance to enforcement.

### Log locations

By default, logs are streamed to **stdout** (the console). This follows cloud-native best practices for log aggregation (Splunk, ELK, etc.).

- **Local Development:** Logs are pretty-printed to the terminal in real-time.
- **Docker:** Use `docker compose logs -f` to view combined logs from all services.
- **Persistent Files:** To capture logs to a file locally, use shell redirection:
  ```bash
  npm start | tee ./logs/portauth.log
  ```

### Tracing the "story"

The logs are instrumented with markers to map directly to the arXiv paper:

1. `STORY STEP 1`: Normative Core creation (Issuer)
2. `STORY STEP 2`: Container receipt and Payload extraction (Engine)
3. `STORY STEP 3`: Constraint evaluation logic (Engine)

## Testing

### Unit tests (Node.js test runner)

```bash
# Engine
( cd portauth-engine && npm install && npm test )

# Issuer
( cd portauth-issuer && npm install && npm test )

# Preflight SDK
( cd sdks/preflight && npm install && npm test )
```

### Conformance corpus

```bash
# From the engine package
( cd portauth-engine && npm run conformance )

# Or run the standalone runner directly
node conformance/run-conformance.js
```

### End-to-end smoke (requires running services)

```bash
# Bring up the compose stack first (docker or podman compose)
docker compose -f portauth-docker-compose.yml up -d --build

# Then drive the smoke flow
ISSUER_URL=http://localhost:3100 \
ENGINE_URL=http://localhost:3200 \
ISSUER_ID_FOR_ENGINE=http://portauth-issuer:3100 \
ISSUER_JWKS_URI=http://portauth-issuer:3100/.well-known/jwks.json \
  ./portauth-engine/test/e2e-smoke.sh
```

Why the override matters: the issuer mints credentials with `iss = http://portauth-issuer:3100` (its container hostname). The smoke script registers that exact value with the engine and points the engine at the in-network JWKS URI so the engine resolves the issuer's keys without going back through `localhost`.

The smoke flow covers: health checks, issuer registration, issuance, happy-path allow, numeric limit deny, permission deny, enumerated list deny, string pattern deny, subject binding mismatch, audit retrieval, revocation plus re-evaluation, local policy enforcement, stateful usage limit, governance manifest, and the optional OPA advisory policy check.

### Black-box system tests (Python pytest)

```bash
# Requires running portauth services
pytest tests/automated_system_tests.py -v
```

Covers: BB-01 register issuer, BB-05 create local policy, BB-10 issue credential, BB-20 happy path, BB-22 numeric limit denied (most-restrictive-wins), BB-40 audit record retrieval.

### Test results summary

| Suite | Tests | Pass | Fail | Notes |
|-------|-------|------|------|-------|
| `portauth-engine` `npm test` | 106 | 106 | 0 | Includes constraint, stateful, X.509, VC-JOSE, DID, status list, audit, glob attenuation, workflow, semantic resolver, and conformance-wrapper tests |
| `portauth-engine` `npm run conformance` | 9 vectors | 9 | 0 | Machine-readable corpus in `conformance/vectors/reference-core.json` |
| `portauth-issuer` `npm test` | 3 | 3 | 0 | Status List Publisher |
| `sdks/preflight` `npm test` | unit | pass | 0 | Covers profile / constraint / vocabulary / trust anchor / context field rules |
| `e2e-smoke.sh` (against compose) | 20 steps | 20 | 0 | Verified end-to-end against `podman compose` |
| `automated_system_tests.py` | 6 | not exercised this pass |  | Requires live services |

## Configuration reference

### portauth-issuer

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3100` | HTTP listen port |
| `DATABASE_URL` |  | PostgreSQL connection string |
| `ADMIN_API_KEY` |  | Bearer token for admin endpoints |
| `ISSUER_ID` |  | Issuer identifier (URL for Tier 1) |
| `ISSUER_PUBLIC_URL` |  | Optional HTTPS base URL used to derive the default status list URL |
| `VC_ISSUER_DID` |  | DID for VC-JOSE issuance (must start with `did:`) |
| `X509_PRIVATE_KEY_PEM` |  | PKCS8 private key for Tier 2 X.509 JWT issuance |
| `X509_CERT_CHAIN_PEM` |  | PEM certificate chain for Tier 2 X.509 JWT issuance |
| `X509_ISSUER_ID` | `ISSUER_ID` | Optional issuer identifier override for Tier 2 |
| `X509_SIGNING_ALGORITHM` | `SIGNING_ALGORITHM` | JWS algorithm for Tier 2 signing |
| `SIGNING_ALGORITHM` | `ES256` | Key algorithm |
| `DEFAULT_TTL_HOURS` | `24` | Default credential lifetime |
| `MAX_TTL_HOURS` | `720` | Maximum credential lifetime |
| `VC_STATUS_LIST_URL` |  | Bitstring Status List URL (enables revocation for VCs) |

### portauth-engine

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3200` | HTTP listen port |
| `DATABASE_URL` |  | PostgreSQL connection string |
| `ADMIN_API_KEY` |  | Bearer token for admin endpoints |
| `PORTAUTH_EVALUATOR_ID` / `EVALUATOR_ID` |  | Receiver identity used for `aud` checks |
| `PORTAUTH_ENGINE_AUDIENCE` / `ENGINE_AUDIENCE` |  | Alternate receiver audience identifiers |
| `PORTAUTH_REQUIRE_PROOF_OF_POSSESSION` | `false` | Require request-level PoP even when the credential has no `cnf.jwk` |
| `PORTAUTH_PROOF_MAX_AGE_SECONDS` | `300` | Accepted PoP freshness window |
| `PORTAUTH_X509_TRUST_ANCHOR_PEM` / `PORTAUTH_X509_TRUST_ANCHORS_PEM` |  | Optional env trust anchor fallback for Tier 2 verification |
| `PORTAUTH_X509_TRUST_ANCHOR_SHA256` |  | Optional env trust anchor fingerprint fallback for Tier 2 verification |
| `PORTAUTH_X509_REVOCATION_POLICY` | `soft-fail` | Tier 2 revocation behavior if the revoked-certificate table is unavailable |
| `JWKS_CACHE_TTL_SECONDS` | `300` | JWKS fetch cache TTL |
| `DID_CACHE_TTL_SECONDS` | `300` | DID document cache TTL |
| `STATUS_LIST_CACHE_TTL_SECONDS` | `60` | Bitstring Status List cache TTL |
| `STATUS_LIST_FETCH_TIMEOUT_MS` | `5000` | Status list HTTP fetch timeout |
| `MAX_DELEGATION_DEPTH` | `5` | Max delegation chain depth |
| `REQUIRE_ATTENUATION_PROOF` | `true` | Require attenuation in delegation |
| `AUDIT_SIGNING_KEY_ID` |  | Key ID for audit record signing |

### portauth-db

| Variable | Default | Description |
|----------|---------|-------------|
| `POSTGRES_DB` | `portauth` | Database name |
| `POSTGRES_USER` | `postgres` | Database user |
| `POSTGRES_PASSWORD` | `postgres` | Database password |

## Running locally without compose

```bash
# Terminal 1: PostgreSQL
docker run -d --name portauth-db \
  -e POSTGRES_DB=portauth \
  -e POSTGRES_PASSWORD=postgres \
  -p 5433:5432 \
  postgres:16-alpine

# Terminal 2: Issuer
cd portauth-issuer
npm install
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/portauth" \
ADMIN_API_KEY="dev-issuer-key-change-me" \
PORT=3100 \
node src/index.js

# Terminal 3: Engine
cd portauth-engine
npm install
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/portauth" \
ADMIN_API_KEY="dev-engine-key-change-me" \
PORT=3200 \
node src/index.js
```

## Feature Status

This table is the single source of truth for what is implemented, what is partially implemented, and what is roadmap. Anything not listed is not in this repository.

| Capability | Paper section | Status | Where it lives | Notes |
|---|---|---|---|---|
| Tier 1 JWT issuance + verification | 7.1, 7.7 | Implemented | `portauth-issuer/src/services/credentialIssuer.js`, `portauth-engine/src/services/credentialVerifier.js` | JWKS discovery, ES256, audience checks, optional PoP |
| Tier 2 X.509-backed JWT issuance + verification | 7.1, 7.7 | Implemented | `portauth-issuer/src/services/x509CredentialIssuer.js`, `portauth-engine/src/services/x509Verifier.js` | Chain build, trust-anchor check, identity binding, soft/hard-fail revocation against the local revoked-cert table |
| Tier 3 VC-JOSE issuance + verification (did:web, did:key) | 7.1, 7.7 | Implemented | `vcCredentialIssuer.js`, `vcJoseVerifier.js`, `didResolver.js` | did:web + did:key only; other DID methods fail-closed with `unsupported_did_method` |
| Bitstring Status List publication + verification | 7.9 | Implemented | `portauth-issuer/src/routes/statusList.js`, `portauth-engine/src/services/statusListChecker.js` | Issuer publishes `/.well-known/status-list/:purpose`; engine fetches, GZIP-decompresses, reads the bit |
| Tier 1 revocation hardening (timeout, retries, failure mode) | 7.9, 7.16.4 | Implemented | `revocationClient.js`, migration `009` | Env defaults: `PORTAUTH_TIER1_REVOCATION_*`; per-issuer overrides on `portauth_trusted_issuers` |
| Constraint engine (numeric, temporal, enumerated, string pattern, cumulative, usage) | 7.3, 7.4 | Implemented | `constraintEvaluator.js`, `statefulAuthorizer.js` | Conjunctive, fail-closed on unknown type or missing field |
| Restricted-glob attenuation (segment-based subset) | 7.3.4, 7.7.3 | Implemented | `globAttenuation.js` | `**` absorbs segments; `*` matches `[^/]*`; literal subset check |
| Delegation chain verification + attenuation | 7.7 | Implemented | `delegationVerifier.js` | Subject/issuer linkage, per-link verification, depth cap |
| Most-restrictive-wins local policy merge | 7.4 | Implemented | `localPolicyMerge.js` | Local policy can only narrow the credential's grant |
| Stateful cumulative + usage counters | 7.5.2 | Implemented | `statefulAuthorizer.js`, migration `007` | Transactional reservation, idempotency key, per-period scoping |
| Synchronous signed audit + startup re-sign sweep + `/audit/health` | 7.10, 7.16.7 | Implemented | `auditLogger.js`, `audit.js`, migration `008` | Sign-then-INSERT in one transaction; HTTP 500 + `audit_signing_failed` on signer failure |
| MVV semantic identifier coverage | 7.12.2 | Implemented | `semanticResolver.js`, `contextEnricher.js` | Built-in identifiers: `core.amount`, `core.currency_code`, `core.currency`, `core.request_time`, `core.timestamp`, `core.resource_id`, `core.resource_type`, `core.action`, `core.workflow_id`, `core.workflow_role`, `core.workflow_step_id`, `core.recipient_id`, `core.count`, `core.quantity`, `core.total_budget`, `core.geo_region`, `core.issuer_id`, `core.subject_id`, `core.presenter_id`, `core.audience_id`, `core.permission`, `core.valid_from`, `core.valid_until`, `core.delegator_id`, `core.request_id`, `core.ip_address` |
| Governance manifest assembled from system state + signing key endpoint | 7.13, 7.14 | Implemented | `wellknown.js`, `governanceState.js`, `engineKeyStore.js` | Revision counter bumped on admin issuer writes |
| Multi-principal workflow composition (`requireAll`, `requireAny`, `threshold`) | 7.5.1 | Implemented | `workflowEvaluator.js`, migration `011` | One wrapping audit row with `workflow_policy_id` + `workflow_sub_audit_ids` |
| Verifiable state authority client | 8.5.1, 8.5.2 | Implemented | `stateAuthorityClient.js`, migration `010` | JWS proof verification, allow-listed authorities, consumed proofs retained |
| Reference verifiable state authority server | 8.5 | Implemented (standalone) | `state-authority-ref/` | Not wired into `portauth-docker-compose.yml`; see *Known limitations* |
| Sender pre-flight discovery SDK (`@portauth/preflight` + CLI) | 7.13 | Implemented | `sdks/preflight/` | Five rules; CLI exit codes `0`/`1`/`2` |
| Conformance corpus runner | 7.16 | Implemented | `conformance/run-conformance.js`, `conformance/vectors/reference-core.json` | 9 vectors; runner exits non-zero on any failure |

### Known limitations (deferred; not in this release)

These are recognized gaps. They are out of scope here and are tracked in [`../CHANGELOG.md`](../CHANGELOG.md) under *Known limitations*.

- **OCSP / CRL Tier 2 wire clients**: schema tables exist (migration 009), but no ASN.1 parsing or HTTP retrieval client is implemented. Tier 2 revocation today falls back to the local revoked-certificate registry plus the `PORTAUTH_X509_REVOCATION_POLICY` soft/hard-fail switch.
- **Tier 3 Bitstring Status List in e2e smoke**: end-to-end pieces all exist; the smoke script does not yet issue then revoke then deny via the status list.
- **State authority in compose**: the reference authority is not started by `portauth-docker-compose.yml` and there is no two-engine race integration test.
- **Workflow and preflight in smoke**: covered by unit tests only.
- **Conformance vectors** for revocation hardening, state authority, glob attenuation, and workflow combinators are not yet in the corpus.
- **Migration runner**: each migration file is one `pool.query(sql)` call. A partial failure can leave the database in an inconsistent state. A real migrator with a `schema_migrations` ledger and per-file transactions is recommended before any production use.
