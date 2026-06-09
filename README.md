# Portable Authorization Engine (PortAuth)

![GitHub issues](https://img.shields.io/github/issues/kyndryl-open-source/aiagent-portable-authorization)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/kyndryl-open-source/aiagent-portable-authorization)
![GitHub top language](https://img.shields.io/github/languages/top/kyndryl-open-source/aiagent-portable-authorization)
![GitHub contributors](https://img.shields.io/github/contributors/kyndryl-open-source/aiagent-portable-authorization)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

**Policy-embedded credential authorization for AI agents**: issue signed credentials with machine-evaluable constraints, verify them at runtime, and enforce allow/deny decisions with a full audit trail.

---

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [Database Schema](#database-schema)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Credential Profiles](#credential-profiles)
- [Constraint Types](#constraint-types)
- [Evaluation Pipeline](#evaluation-pipeline)
- [Workflow Composition](#workflow-composition)
- [Governance Manifest](#governance-manifest)
- [Preflight SDK](#preflight-sdk)
- [State Authority](#state-authority)
- [Testing](#testing)
- [Configuration Reference](#configuration-reference)
- [Feature Status](#feature-status)
- [Paper Traceability](#paper-traceability)
- [Security and Hardening](#security-and-hardening)
- [License](#license)

---

## Overview

PortAuth implements the three-layer authorization architecture described in the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)):

| Layer | Responsibility | What varies | What is constant |
|-------|---------------|-------------|-----------------|
| **Container** | Cryptographic envelope + transport | JWT, VC-JOSE, PKI |  |
| **Payload** | Authorization semantics |  | Agent ID, issuer ID, permissions, typed constraints |
| **Engine** | Evaluation + enforcement |  | Conjunctive, total, fail-closed processing model |

A credential issuer creates a signed authorization credential containing a complete authorization policy, who the agent is, what it may do, and under what conditions. The receiving engine verifies the credential, evaluates every embedded constraint against the actual request context, merges with receiver-local policy (most-restrictive-wins), and produces a cryptographically signed audit record of the decision.

### Use Case: Automated Insurance Claim Processing

A claims authority issues a credential to a **Damage Assessment Agent** authorizing it to read claim data under specific conditions:

- **Claim amount** ≤ $5,000 (numeric limit)
- **Claim type** must be `rear_end` or `collision` (enumerated list)
- **Resource path** must match `/claims/CLM-2026-*` (string pattern)
- **Time window** weekdays 8 AM to 6 PM CT (temporal window)

If the agent requests access to a $7,000 claim, outside business hours, or for an unsupported claim type, the request is denied, even if the receiving system would otherwise allow it. The credential defines the ceiling.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        PORTABLE AUTHORIZATION ENGINE                         │
│                                                                              │
│  ISSUANCE                        ENFORCEMENT                  ADMINISTRATION │
│  ────────                        ───────────                  ────────────── │
│                                                                              │
│  ┌───────────────────┐          ┌──────────────────────────┐                 │
│  │ portauth-issuer   │          │ portauth-engine           │                │
│  │ :3100             │          │ :3200                     │                │
│  │                   │          │                           │                │
│  │ POST /issue       │          │ POST /evaluate            │                │
│  │ POST /issue-vc    │   JWT or │ POST /evaluate/batch      │                │
│  │ POST /revoke      │◄─vc+jwt─┤                           │                │
│  │ POST /introspect  │────────►│ 1. Credential Verification│                │
│  │                   │ revoke   │ 2. Permission Match       │                │
│  │ GET /.well-known/ │ check    │ 3. Constraint Evaluation  │                │
│  │   jwks.json       │          │ 4. Local Policy Merge     │                │
│  └────────┬──────────┘          │ 5. Audit + Decision       │                │
│           │                     │                           │                │
│           │                     │ GET /audit/records/:id    │                │
│           │                     │ GET /audit/verify/:id     │                │
│           │                     │                           │                │
│           │                     │ GET  /admin/issuers       │                │
│           │                     │ POST /admin/issuers       │                │
│           │                     │ PUT  /admin/local-policy  │                │
│           │                     │ PUT  /admin/config/       │                │
│           │                     │       delegation          │                │
│           │                     │                           │                │
│           │                     │ GET /.well-known/         │                │
│           │                     │   agent-governance        │                │
│           │                     └────────────┬──────────────┘                │
│           │                                  │                               │
│           └──────────────┬───────────────────┘                               │
│                          │                                                   │
│                 ┌────────▼────────┐                                          │
│                 │  portauth-db    │                                          │
│                 │  PostgreSQL 16  │                                          │
│                 │  :5433          │                                          │
│                 └─────────────────┘                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

| Directory | Purpose | Stack |
|-----------|---------|-------|
| [`portauth-engine/`](portauth-engine/) | Credential verification, constraint evaluation, stateful authorization, local policy merge, audit, admin APIs, governance manifest, workflow composition | Node.js 20, Express, jose, pg |
| [`portauth-issuer/`](portauth-issuer/) | Credential issuance (JWT, X.509 JWT, VC-JOSE), revocation, introspection, JWKS, Bitstring Status List | Node.js 20, Express, jose, pg |
| [`portauth-db/`](portauth-db/) | PostgreSQL schema migrations (11 files, applied in order at startup) | PostgreSQL 16 Alpine |
| [`state-authority-ref/`](state-authority-ref/) | Reference verifiable state authority server (paper Section 8.5) | Node.js 20, Express, jose |
| [`sdks/preflight/`](sdks/preflight/) | `@portauth/preflight` sender-side discovery client and `portauth-preflight` CLI | Node.js 20, jose |
| [`conformance/`](conformance/) | Machine-readable conformance vectors and runner | JSON, Node.js |
| [`opa/`](opa/) | Advisory compatibility policy used by the local compose stack | OPA/Rego |

### portauth-engine source layout

```
src/
├── index.js                      Express app entry point
├── routes/
│   ├── evaluate.js               POST /evaluate, POST /evaluate/batch, POST /evaluate/workflow
│   ├── admin.js                  Trusted issuer CRUD, local policy, delegation config
│   ├── audit.js                  GET /audit/records/:id, /audit/verify/:id, /audit/health
│   └── wellknown.js              GET /.well-known/agent-governance (state-derived)
├── services/
│   ├── credentialVerifier.js     JWT/X.509/VC-JOSE verification, trust check, PoP, normalization
│   ├── constraintEvaluator.js    Stateless constraint engine
│   ├── statefulAuthorizer.js     Cumulative/usage counters with idempotent reservations
│   ├── stateAuthorityClient.js   Verifiable state authority proof verification (Plan 2)
│   ├── x509Verifier.js           X.509 chain, trust anchor, identity checks
│   ├── delegationVerifier.js     Delegation chain verification + attenuation enforcement
│   ├── globAttenuation.js        Segment-based restricted-glob subset check (Plan 5)
│   ├── localPolicyMerge.js       Most-restrictive-wins merge
│   ├── auditLogger.js            Synchronous sign-then-INSERT, startup re-sign sweep (Plan 6)
│   ├── contextEnricher.js        Populates __credential.* and __request.* MVV identifiers (Plan 4)
│   ├── didResolver.js            did:web + did:key resolution with DB cache
│   ├── statusListChecker.js      W3C Bitstring Status List revocation checking
│   ├── revocationClient.js       Tier 1 introspection with timeout/retries/failure-mode (Plan 1)
│   ├── governanceState.js        Loads issuers + mapping profiles for state-derived manifest (Plan 3)
│   ├── workflowEvaluator.js      requireAll / requireAny / threshold combinators (Plan 7)
│   ├── engineKeyStore.js         Cached audit + manifest signing key (Plan 6)
│   └── semanticResolver.js       MVV + industry profile field resolution
└── db/
    ├── pool.js                   PostgreSQL connection pool + migration runner
    └── migrate.js                Standalone migration runner
```

### portauth-issuer source layout

```
src/
├── index.js                      Express app entry point
├── routes/
│   ├── credentials.js            POST /issue, /issue-x509, /issue-vc, /revoke, /introspect
│   └── wellknown.js              GET /.well-known/jwks.json, /.well-known/status-list/:purpose
├── services/
│   ├── credentialIssuer.js       Tier 1 JWT issuance, revocation, introspection
│   ├── x509CredentialIssuer.js   Tier 2 X.509-backed JWT issuance
│   ├── vcCredentialIssuer.js     Tier 3 VC-JOSE (vc+jwt) issuance
│   ├── statusListPublisher.js    Bitstring Status List publication from issuer state
│   └── keyManager.js             ES256 key generation, rotation, JWKS export
├── middleware/
│   ├── auth.js                   Bearer token admin authentication (timing-safe)
│   └── validation.js             JSON Schema body validation (Ajv)
└── schemas/
    └── issue-request.json        JSON Schema for credential issuance requests
```

---

## Database Schema

Eleven migration files are applied at startup in order. Both the issuer and engine call `runMigrations()` against the shared database; all migrations are idempotent (`CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`).

### Migration 001: Issuer Tables

| Table | Purpose |
|-------|---------|
| `portauth_signing_keys` | Signing key pairs (kid, algorithm, JWKs, active flag, rotation timestamp) |
| `portauth_issued_credentials` | Issued credential metadata (jti, issuer, agent, permissions, constraints, validity, revocation state) |

### Migration 002: Engine Tables

| Table | Purpose |
|-------|---------|
| `portauth_trusted_issuers` | Trusted issuer registry (issuer ID, trust tier 1/2/3, permitted permissions, JWKS URI) |
| `portauth_local_policies` | Receiver-local policy constraints (same typed constraint format as credentials) |
| `portauth_delegation_config` | Delegation chain limits (max_depth, require_attenuation_proof) |
| `portauth_audit_records` | Signed audit records (decision, constraint results, verification state, cryptographic signature) |

### Migration 003: Semantic Mapping Profiles

| Table | Purpose |
|-------|---------|
| `portauth_mapping_profiles` | Semantic field → local field mappings. Pre-seeded with `mvv-core` and `insurance-claims-v1` profiles |

### Migration 004: VC/DID Support (Tier 3)

| Table / Change | Purpose |
|----------------|---------|
| `portauth_trusted_issuers` (ALTER) | Adds `did_method`, `governance_domain`, `governance_status`, `status_list_url`, `metadata` (JSONB) |
| `portauth_did_cache` (NEW) | DID document cache with TTL (did:web, did:key) |
| `portauth_credential_status` (NEW) | Bitstring Status List entries per credential |
| `portauth_status_list_cache` (NEW) | Receiver-side status list response cache |
| `portauth_issued_credentials` (ALTER) | Adds `format` (jwt/vc-jose), `status_list_index`, `did_issuer_id` |

### Migration 005: Engine Signing Keys

| Table | Purpose |
|-------|---------|
| `portauth_engine_signing_keys` | Persisted signing keys for audit records and governance manifests |

### Migration 006: Mapping Profile Governance

| Change | Purpose |
|--------|---------|
| `portauth_mapping_profiles` (ALTER) | Adds `trusted` and `valid_until` for stale/conflict handling |

### Migration 007: Reference Extensions

| Table / Change | Purpose |
|----------------|---------|
| `portauth_trusted_issuers` (ALTER) | Adds X.509 subject/SAN/trust-anchor/revocation metadata |
| `portauth_x509_revoked_certificates` (NEW) | Receiver-side revoked certificate registry |
| `portauth_issued_credentials` (ALTER) | Adds `x509-jwt` to the credential format check |
| `portauth_authorization_counters` (NEW) | Period-scoped counters for cumulative/usage constraints |
| `portauth_authorization_counter_events` (NEW) | Idempotency records to avoid duplicate counter increments |

### Migration 008: Audit `signed_at` (Plan 6)

| Change | Purpose |
|--------|---------|
| `portauth_audit_records` (ALTER) | Adds `signed_at` timestamp populated by the synchronous sign-then-INSERT path, plus a partial index on rows where `signature IS NULL` so the startup re-sign sweep is cheap |

### Migration 009: Revocation Hardening (Plan 1)

| Table / Change | Purpose |
|----------------|---------|
| `portauth_trusted_issuers` (ALTER) | Adds `revocation_mechanism`, `revocation_failure_mode`, `revocation_timeout_ms`, `revocation_retries` for per-issuer Tier 1 introspection policy |
| `portauth_x509_ocsp_cache` (NEW) | Cache slot for OCSP responses keyed by certificate fingerprint (table only; OCSP client is on the roadmap) |
| `portauth_x509_crl_cache` (NEW) | Cache slot for CRL bodies (table only; CRL client is on the roadmap) |
| `portauth_x509_crl_entries` (NEW) | Per-serial CRL entries (table only; populated when the CRL client lands) |

### Migration 010: State Authority (Plan 2)

| Table | Purpose |
|-------|---------|
| `portauth_state_authorities` | Allow-list of verifiable state authorities (authority id, base URL, signing JWK, governance status) |
| `portauth_state_proofs` | Consumed state-authority proofs retained for audit and replay defence |

### Migration 011: Workflow Policies (Plan 7)

| Table / Change | Purpose |
|----------------|---------|
| `portauth_workflow_policies` | Versioned multi-principal workflow policies with `requireAll` / `requireAny` / `threshold` combinators and required-role definitions |
| `portauth_audit_records` (ALTER) | Adds `workflow_policy_id` and `workflow_sub_audit_ids` columns plus a partial index for workflow rows |

---

## Quick Start

### Try it in 2 steps (~2-4 minutes)

```bash
git clone https://github.com/kyndryl-open-source/aiagent-portable-authorization.git
cd aiagent-portable-authorization && ./quickstart.sh
```

`quickstart.sh` brings up the full stack (PostgreSQL, issuer, engine, OPA sidecar) with `docker compose` or `podman compose`, waits for health, then runs the end-to-end smoke flow and prints a PASS/FAIL summary. Requires only Docker Desktop or Podman 5+ on the host.

### Prerequisites

- Docker and Docker Compose, or Podman 5+
- Node.js 20+ (only for local development without compose)

### Run with Docker or Podman Compose (manual)

```bash
cd aiagent-portable-authorization

# Docker
docker compose -f portauth-docker-compose.yml up -d --build

# Podman (verified equivalent)
podman compose -f portauth-docker-compose.yml up -d --build

# Verify health
curl -s http://localhost:3100/health   # issuer
curl -s http://localhost:3200/health   # engine
```

The compose file sets `DB_SSL: "false"` on the issuer and engine because the bundled `postgres:16-alpine` image does not enable TLS. Do not change that flag for local runs. The compose file is dev-only and uses placeholder credentials; see [SECURITY.md](SECURITY.md) for hardening.

### Run locally (development, no compose)

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

### End-to-end walkthrough

```bash
ISSUER=http://localhost:3100
ENGINE=http://localhost:3200
ISSUER_KEY="dev-issuer-key-change-me"
ENGINE_KEY="dev-engine-key-change-me"

# 1. Register the issuer as trusted
curl -X POST "$ENGINE/admin/issuers" \
  -H "Authorization: Bearer $ENGINE_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "issuerId": "'"$ISSUER"'",
    "trustTier": 1,
    "trustedForPermissions": ["damage_assessment:read"],
    "label": "Claims Authority",
    "jwksUri": "'"$ISSUER/.well-known/jwks.json"'"
  }'

# 2. Issue a credential (Tier 1 JWT)
CRED=$(curl -s -X POST "$ISSUER/credentials/issue" \
  -H "Authorization: Bearer $ISSUER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent:damage-assessment-001",
    "permissions": ["damage_assessment:read"],
    "constraints": [
      {"type": "NumericLimitConstraint", "field": "claimAmount", "operator": "lte", "value": 5000}
    ]
  }')
JWT=$(echo "$CRED" | jq -r '.credential')

# 3. Evaluate, happy path (allow)
curl -s -X POST "$ENGINE/evaluate" \
  -H "Content-Type: application/json" \
  -d '{
    "credential": "'"$JWT"'",
    "presenterId": "agent:damage-assessment-001",
    "requestContext": {
      "action": "damage_assessment:read",
      "resource": "/claims/CLM-2026-0001",
      "claimAmount": 4200
    }
  }' | jq .decision
# → "allow"

# 4. Evaluate, numeric limit exceeded (deny)
curl -s -X POST "$ENGINE/evaluate" \
  -H "Content-Type: application/json" \
  -d '{
    "credential": "'"$JWT"'",
    "presenterId": "agent:damage-assessment-001",
    "requestContext": {
      "action": "damage_assessment:read",
      "resource": "/claims/CLM-2026-0001",
      "claimAmount": 7500
    }
  }' | jq .decision
# → "deny"

# 5. Issue a VC-JOSE credential (Tier 3)
VC_CRED=$(curl -s -X POST "$ISSUER/credentials/issue-vc" \
  -H "Authorization: Bearer $ISSUER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
    "permissions": ["damage_assessment:read"],
    "constraints": [
      {"type": "NumericLimitConstraint", "field": "claimAmount", "operator": "lte", "value": 5000}
    ]
  }')
echo "$VC_CRED" | jq .format
# → "vc-jose"
```

---

## API Reference

### Issuer Service (`portauth-issuer`, port 3100)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/health` |  | Liveness check |
| `POST` | `/credentials/issue` | Bearer | Issue a Tier 1 JWT credential |
| `POST` | `/credentials/issue-x509` | Bearer | Issue a Tier 2 X.509-backed JWT credential |
| `POST` | `/credentials/issue-vc` | Bearer | Issue a Tier 3 VC-JOSE (`vc+jwt`) credential |
| `POST` | `/credentials/revoke` | Bearer | Revoke a credential by ID |
| `POST` | `/credentials/introspect` |  | Token introspection (active/inactive) |
| `GET` | `/.well-known/jwks.json` |  | Issuer's public signing keys |
| `GET` | `/.well-known/status-list/:purpose` |  | Published Bitstring Status List credential (`revocation` supported) |

### Engine Service (`portauth-engine`, port 3200)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/health` |  | Liveness check |
| `POST` | `/evaluate` |  | Evaluate a single credential against request context |
| `POST` | `/evaluate/batch` |  | Evaluate a delegation chain |
| `POST` | `/evaluate/workflow` |  | Multi-principal workflow evaluation (`requireAll`, `requireAny`, `threshold`) |
| `GET` | `/audit/records/:id` |  | Retrieve a signed audit record |
| `GET` | `/audit/verify/:id` |  | Verify audit record integrity |
| `GET` | `/audit/signing-key` |  | Audit signing public key (JWKS) |
| `GET` | `/audit/health` |  | Audit subsystem health (unsigned-record backlog, last re-sign sweep) |
| `GET` | `/admin/issuers` | Bearer | List trusted issuers |
| `POST` | `/admin/issuers` | Bearer | Register a trusted issuer |
| `PUT` | `/admin/issuers/:id` | Bearer | Update a trusted issuer |
| `DELETE` | `/admin/issuers/:id` | Bearer | Remove a trusted issuer |
| `GET` | `/admin/local-policy` |  | List receiver-local policies |
| `POST` | `/admin/local-policy` | Bearer | Create a local policy |
| `PUT` | `/admin/local-policy/:id` | Bearer | Update a local policy |
| `DELETE` | `/admin/local-policy/:id` | Bearer | Remove a local policy |
| `GET` | `/admin/config/delegation` | Bearer | Get delegation chain config |
| `PUT` | `/admin/config/delegation` | Bearer | Update delegation limits |
| `GET` | `/.well-known/agent-governance` |  | Signed governance manifest (assembled from current system state) |
| `GET` | `/.well-known/agent-governance/signing-key` |  | Governance manifest signing public key (JWKS) |

### HTTP Status Semantics

| Status | Meaning |
|--------|---------|
| `200 OK` | Credential verified, authorization decision computed (both `allow` and `deny`) |
| `201 Created` | Credential or resource created successfully |
| `400 Bad Request` | Request body malformed, never reached credential processing |
| `422 Unprocessable Entity` | Credential parsed but unusable (invalid signature, untrusted issuer, expired, revoked, subject mismatch) |
| `500 Internal Server Error` | Unexpected failure |

### Error Codes

| Code | Category | Description |
|------|----------|-------------|
| `credential_incomplete` | Structure | Missing authorization policy component (iss, sub, permissions, or constraints) |
| `signature_invalid` | Verification | JWS/VC signature does not verify against issuer's key |
| `issuer_untrusted` | Trust | Issuer not in trusted registry or governance status invalid |
| `subject_binding_mismatch` | Verification | Presenter identity is not the credential subject |
| `credential_expired` | Temporal | Credential outside its validity window |
| `credential_revoked` | Status | Credential revoked (introspection or Bitstring Status List) |
| `permission_denied` | Authorization | Requested action not in declared permissions |
| `constraint_failed` | Authorization | One or more typed constraints failed evaluation |
| `constraint_unknown` | Authorization | Unrecognized constraint type (fail-closed) |
| `context_field_missing` | Authorization | Request context missing a field required by the policy |
| `delegation_widened` | Delegation | Delegate widens permissions or loosens constraints |
| `delegation_chain_broken` | Delegation | A link in the chain failed verification |
| `delegation_depth_exceeded` | Delegation | Chain exceeds configured max depth |
| `local_policy_denied` | Local policy | Receiver-local policy further restricts the grant |
| `semantic_identifier_unknown` | Semantic | Signed field not in MVV or governed profile |
| `semantic_alias_conflict` | Semantic | More than one active governed mapping defines incompatible aliases |
| `mapping_profile_invalid` | Semantic | Mapping profile is untrusted or expired |
| `semantic_alias_missing` | Semantic | Governed semantic identifier has no local alias |
| `semantic_type_mismatch` | Semantic | Mapped field type incompatible with constraint type |
| `unsupported_did_method` | VC-JOSE | DID method not did:web or did:key |
| `audience_mismatch` | Verification | Credential audience does not include this receiver |
| `proof_of_possession_failed` | Verification | Request-bound PoP JWS is missing or invalid |
| `x509_chain_invalid` / `x509_chain_untrusted` | Tier 2 | Certificate chain failed or does not reach a trusted anchor |
| `x509_subject_mismatch` | Tier 2 | Signing certificate identity is not bound to the trusted issuer |
| `x509_revocation_unavailable` | Tier 2 | Revocation status is unavailable under hard-fail policy |
| `tier1_revocation_unavailable` | Tier 1 | Introspection unreachable under the configured hard-fail policy |
| `stateful_limit_exceeded` | Authorization | Stateful cumulative/usage counter would exceed its limit |
| `state_authority_untrusted` | State authority | Proof issuer not in the allow-list |
| `state_authority_unreachable` | State authority | Reservation request to the authority failed under hard-fail policy |
| `state_authority_proof_invalid` | State authority | JWS proof signature, audience, or binding failed verification |
| `workflow_policy_unknown` | Workflow | Workflow policy id is not registered |
| `workflow_role_mismatch` | Workflow | A submitted credential's role does not match a required workflow role |
| `workflow_denied` | Workflow | Combinator (`requireAll`, `requireAny`, `threshold`) did not produce an allow |
| `audit_signing_failed` | Audit | Synchronous audit signing failed; no row was written |

---

## Credential Profiles

### Tier 1: JWS/JWT (Enterprise-Native)

Standard JWT with authorization claims. Issuer resolved via JWKS/OIDC discovery.

```
Header:  { "typ": "JWT", "alg": "ES256", "kid": "..." }
Payload: { "iss": "https://issuer.example.com",
           "sub": "agent:damage-001",
           "permissions": ["damage_assessment:read"],
           "constraints": [{ "type": "NumericLimitConstraint", ... }],
           "nbf": ..., "exp": ... }
```

Revocation: short TTL + token introspection.

### Tier 2: X.509-backed JWT

JWT with the same authorization claims as Tier 1, but the signing key is bound to an `x5c` certificate chain in the JOSE header. The engine verifies the chain, checks the configured trust anchor, validates certificate time and identity binding, then verifies the JWT signature with the leaf certificate public key.

```
Header:  { "typ": "JWT", "alg": "ES256", "x5c": ["leaf", "issuer-or-root"], "x5t#S256": "..." }
Payload: { "iss": "https://issuer.example.com",
           "sub": "agent:damage-001",
           "permissions": ["damage_assessment:read"],
           "constraints": [...],
           "nbf": ..., "exp": ... }
```

Trust metadata is registered through `/admin/issuers` with `trustTier: 2` plus `x509TrustAnchorPem` or `x509TrustAnchorSha256`; `x509SanUri` or `x509Subject` binds the leaf certificate to the issuer. Revocation is checked against `portauth_x509_revoked_certificates`, with `soft-fail` or `hard-fail` policy.

### Tier 3: VC-JOSE (Cross-Domain DID/VC)

W3C Verifiable Credential Data Model 2.0 payload secured with JOSE/JWS (`vc+jwt` media type). Issuer is a DID resolved via did:web or did:key.

```
Header:  { "typ": "vc+jwt", "alg": "ES256", "kid": "did:web:issuer.example#key-1" }
Payload: { "iss": "did:web:issuer.example",
           "sub": "did:key:z6Mk...",
           "jti": "urn:uuid:...",
           "vc": {
             "@context": ["https://www.w3.org/ns/credentials/v2"],
             "type": ["VerifiableCredential", "AgentAuthorizationCredential"],
             "credentialSubject": {
               "id": "did:key:z6Mk...",
               "permissions": ["damage_assessment:read"],
               "constraints": [...]
             },
             "credentialStatus": { "type": "BitstringStatusListEntry", ... }
           }}
```

Revocation: W3C Bitstring Status List (GZIP-compressed bitstring, base64url-encoded).

### Normalization

All profiles normalize to the same canonical payload before constraint evaluation:

```json
{
  "issuerId": "...",
  "agentId": "...",
  "credentialId": "...",
  "permissions": ["damage_assessment:read"],
  "constraints": [{ "type": "NumericLimitConstraint", ... }],
  "issuedAt": "...",
  "expiresAt": "...",
  "credentialProfile": "tier1-jwt | tier2-x509-jwt | vc-jose"
}
```

The constraint evaluator, local policy merge, and delegation verifier operate on this normalized payload, they are container-agnostic.

---

## Constraint Types

| Type | Properties | Evaluation | Attenuation Rule |
|------|-----------|------------|-----------------|
| **NumericLimitConstraint** | `field`, `operator` (eq/lt/lte/gt/gte), `value`, `currency?` | Extract field, apply operator | Delegate must be equal or more restrictive |
| **TemporalWindowConstraint** | `field`, `validFrom`, `validUntil`, `timezone`, `daysOfWeek?` | Convert to TZ, check time + day | Delegate window must be subset |
| **EnumeratedListConstraint** | `field`, `allowed?`, `denied?` | Member check; deny takes precedence | Delegate allowed ⊆ parent allowed |
| **StringPatternConstraint** | `field`, `pattern`, `matchType` (exact/prefix/suffix/restricted_glob) | Pattern match per matchType | Delegate pattern matches subset of parent |
| **CumulativeLimitConstraint** | `id`, `field`, `maxAmount`, `period`, `scope` | Transactional counter reservation | Delegate must preserve or lower the cumulative limit |
| **UsageLimitConstraint** | `id`, `maxUses`, `period`, `scope` | Transactional use counter reservation | Delegate must preserve or lower the use limit |

**Processing model:** conjunctive (AND), total (every constraint evaluated), fail-closed (unknown types, missing fields → deny). Stateful constraints are deferred during stateless evaluation and enforced transactionally after receiver-local policy passes. Use `idempotencyKey` or the `Idempotency-Key` header to avoid duplicate reservations on retries.

---

## Evaluation Pipeline

```
Credential + Request Context
        │
        ▼
┌─────────────────────────────┐
│ 1. Credential Verification  │   Decode → resolve key (JWKS, X.509, or DID) → verify JWS
│    - Structural validation  │   → temporal validity → issuer trust → audience
│    - Signature verification │   → revocation → subject binding → optional PoP
│    - Trust + revocation     │   → normalize payload
└──────────────┬──────────────┘
               │ (422 on failure)
               ▼
┌─────────────────────────────┐
│ 2. Permission Match         │   Requested action ∈ declared permissions?
└──────────────┬──────────────┘
               │ (200 deny: permission_denied)
               ▼
┌─────────────────────────────┐
│ 3. Constraint Evaluation    │   Each typed constraint vs. request context
│    - NumericLimit           │   AND logic, all must pass
│    - TemporalWindow         │   Fail-closed on unknown type or missing field
│    - EnumeratedList         │
│    - StringPattern          │
└──────────────┬──────────────┘
               │ (200 deny: constraint_failed)
               ▼
┌─────────────────────────────┐
│ 4. Local Policy Merge       │   Receiver constraints ∩ credential constraints
│    (most-restrictive-wins)  │   Credential ceiling can never be exceeded
└──────────────┬──────────────┘
               │ (200 deny: local_policy_denied)
               ▼
┌─────────────────────────────┐
│ 5. Stateful Reservation     │   Cumulative/usage counters update transactionally
└──────────────┬──────────────┘
               │ (200 deny: stateful_limit_exceeded)
               ▼
┌─────────────────────────────┐
│ 6. Audit + Decision         │   Signed audit record → PostgreSQL
│    → allow / deny           │   auditRecordId in every response
└─────────────────────────────┘
```

For delegation chains (`POST /evaluate/batch`), step 1 is expanded to verify every link in the chain, require issuer-subject continuity (`child.issuerId == parent.agentId`), require `delegatedFrom` linkage, validate attenuation at each step, and then evaluate the leaf credential's effective constraints.

### Audience and Proof of Possession

If a credential includes `aud`, the engine rejects it unless the value includes `PORTAUTH_EVALUATOR_ID`, `EVALUATOR_ID`, `PORTAUTH_ENGINE_AUDIENCE`, or `ENGINE_AUDIENCE`. If a credential includes `cnf.jwk`, the request must include `proofOfPossession.jws`.

The PoP JWS payload is signed by the private key corresponding to `cnf.jwk` and must include:

```json
{
  "credentialId": "urn:uuid:...",
  "presenterId": "agent:...",
  "requestHash": "sha256-base64url-of-canonical-requestContext",
  "iat": 1780670000
}
```

---

## Workflow Composition

`POST /evaluate/workflow` evaluates a multi-principal decision against a registered workflow policy. The body carries the policy id, the request context, and a list of presented credentials (each with its `presenterId`, optional `proofOfPossession`, and a `role` that maps to a `requiredRole` in the policy).

Supported combinators:

- `requireAll`, every required role must produce an allow.
- `requireAny`, at least one required role must produce an allow.
- `threshold`, at least `n` of `m` required roles must produce an allow.

Each per-credential sub-decision is recorded and a single wrapping audit row captures the policy id and the child audit ids in `workflow_sub_audit_ids`. Role mismatch surfaces as `workflow_role_mismatch`; an unknown policy id surfaces as `workflow_policy_unknown`.

## Governance Manifest

`GET /.well-known/agent-governance` returns a signed JSON manifest assembled from current system state:

- **Credential profiles** supported (tier1-jwt, tier2-x509-jwt, tier3-vc-jose) with algorithms, trust anchors, DID methods, status mechanisms.
- **Trust anchors** per tier, including the active set of trusted issuers loaded from `portauth_trusted_issuers`.
- **Constraint types** accepted, including stateful cumulative and usage profiles.
- **Vocabulary version** and the active set of governed mapping profiles loaded from `portauth_mapping_profiles`.
- **Required context fields** that the receiver enforces.
- **Audience and proof-of-possession metadata** for senders' pre-flight checks.

The manifest carries a JWS signature. Its signing key is published at `GET /.well-known/agent-governance/signing-key`. A revision counter is bumped on every admin issuer write so the manifest invalidates after configuration changes.

Senders may perform advisory pre-flight compatibility checks against this manifest, but the receiver's engine remains authoritative.

## Preflight SDK

`@portauth/preflight` (`sdks/preflight/`) is a sender-side discovery and compatibility checker. It fetches the receiver's governance manifest, verifies its JWS against the published signing key, and runs five rules against a candidate credential:

1. **Profile match**: credential profile is in the receiver's `credentialProfiles`.
2. **Constraint types**: every constraint type in the credential is in the receiver's `constraintTypes`.
3. **Vocabulary**: every signed semantic identifier is in the receiver's `vocabularyVersion` (or a governed profile listed there).
4. **Trust anchors**: issuer is reachable through one of the manifest's trust anchors for the credential's tier.
5. **Required context fields**: the proposed request context covers the manifest's `requiredContextFields`.

The package exposes a `preflightCheck()` function and a `portauth-preflight` CLI:

```bash
npx --package=@portauth/preflight portauth-preflight \
  --manifest https://receiver.example.com/.well-known/agent-governance \
  --credential ./agent.jwt \
  --signing-key-url https://receiver.example.com/.well-known/agent-governance/signing-key \
  --request-context '{"action":"damage_assessment:read","claimAmount":4200}' \
  --json
```

Exit codes: `0` compatible, `2` incompatible, `1` operational failure (network, parse, signature).

## State Authority

A verifiable state authority (paper Section 8.5) provides cross-engine stateful reservations. `stateAuthorityClient.js` requests a reservation from an allow-listed authority, verifies the returned JWS-signed proof against the authority's published JWK, and binds the proof to the credential, action, and request id. Consumed proofs are stored in `portauth_state_proofs` for audit and replay defence.

Allow-listed authorities live in `portauth_state_authorities` (one row per authority with base URL, JWK, governance status). Per-issuer failure mode is controlled in the same row. Error codes: `state_authority_untrusted`, `state_authority_unreachable`, `state_authority_proof_invalid`.

A reference authority server lives under `state-authority-ref/` (Node 20 / Express / jose). It is not started by `portauth-docker-compose.yml`; see [Feature Status](#feature-status).

---

## Logging & Observability

PortAuth uses **Pino** for structured JSON logging. This allows the "story" of the authorization process to be traced step-by-step from issuance to enforcement.

### Log Locations
By default, logs are streamed to **stdout** (the console). This follows cloud-native best practices for log aggregation (Splunk, ELK, etc.).

- **Local Development:** Logs are pretty-printed to the terminal in real-time.
- **Docker:** Use `docker compose logs -f` to view combined logs from all services.
- **Persistent Files:** To capture logs to a file locally, use shell redirection:
  ```bash
  npm start | tee ./logs/portauth.log
  ```

### Tracing the "Story"
The logs are instrumented with markers to map directly to the arXiv paper:
1. `STORY STEP 1`: Normative Core creation (Issuer)
2. `STORY STEP 2`: Container receipt and Payload extraction (Engine)
3. `STORY STEP 3`: Constraint evaluation logic (Engine)

---

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
cd kyndrylresearch-aiagent-portauth

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

---

## Configuration Reference

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

---

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

These are recognized gaps. They are out of scope here and are tracked in [`CHANGELOG.md`](CHANGELOG.md) under *Known limitations*.

- **OCSP / CRL Tier 2 wire clients**: schema tables exist (migration 009), but no ASN.1 parsing or HTTP retrieval client is implemented. Tier 2 revocation today falls back to the local revoked-certificate registry plus the `PORTAUTH_X509_REVOCATION_POLICY` soft/hard-fail switch.
- **Tier 3 Bitstring Status List in e2e smoke**: end-to-end pieces all exist; the smoke script does not yet issue then revoke then deny via the status list.
- **State authority in compose**: the reference authority is not started by `portauth-docker-compose.yml` and there is no two-engine race integration test.
- **Workflow and preflight in smoke**: covered by unit tests only.
- **Conformance vectors** for revocation hardening, state authority, glob attenuation, and workflow combinators are not yet in the corpus.
- **Migration runner**: each migration file is one `pool.query(sql)` call. A partial failure can leave the database in an inconsistent state. A real migrator with a `schema_migrations` ledger and per-file transactions is recommended before any production use.

---

## Paper Traceability

Requirements traced from the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)) to implementation and test evidence.

### BR-1: Credential Issuance with Embedded Authorization Policy

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Credential contains all 4 authorization policy components | `credentialIssuer.js` builds JWT with `iss`, `sub`, `permissions`, `constraints` | e2e-smoke step "Issue credential"; BB-10 | Pass |
| JWS/JWT profile carries permissions and constraints as JWT claims | `credentialIssuer.js` signs with `jose.SignJWT` (ES256) | e2e-smoke + BB-10 | Pass |
| VC profile uses W3C VC Data Model 2.0 | `vcCredentialIssuer.js` builds `vc` claim with `credentials/v2` context | `vcJoseVerifier.test.js` | Pass |
| VC profile places permissions in `credentialSubject` | `vcCredentialIssuer.js` | `vcJoseVerifier.test.js` | Pass |
| Credential cryptographically signed by issuer | `keyManager.js` + `jose.SignJWT` | e2e-smoke verification step | Pass |
| Credential includes validity window | `nbf` / `exp` on Tier 1; `vc.validFrom` / `vc.validUntil` on Tier 3 | `vcJoseVerifier.test.js` expired-credential test | Pass |
| VC profile includes `credentialStatus` for revocation | `vcCredentialIssuer.js` adds `BitstringStatusListEntry` when configured | `vcJoseVerifier.test.js` status list suite | Pass |

### BR-2: Constraint Evaluation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Each typed constraint evaluated correctly | `constraintEvaluator.js` | `constraintEvaluator.test.js`; e2e-smoke deny steps; BB-22 | Pass |
| AND logic across constraints | `evaluateConstraints()` | `constraintEvaluator.test.js` | Pass |
| Unknown constraint types fail-closed | Default branch returns `{pass:false, error:"constraint_unknown"}` | `constraintEvaluator.test.js` | Pass |
| Missing context fields fail-closed | Field-presence check before evaluation | `constraintEvaluator.test.js`; semantic resolver | Pass |
| Detailed per-constraint reason trace | `evaluateConstraints()` returns `results[]` | e2e-smoke + BB-22 responses | Pass |

### BR-3: Most-Restrictive-Wins

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Effective grant = credential intersect local policy | `localPolicyMerge.js` | BB-22 (4500 local cap, 5000 credential, 4800 denied) | Pass |
| Credential ceiling never exceeded | `localPolicyMerge.js` narrows only | BB-22; architecture | Pass |

### BR-4: Delegation with Attenuation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Per-link verification | `delegationVerifier.js` | `/evaluate/batch` | Pass |
| Permission subset enforced | `validateAttenuation()` | Architecture + unit tests | Pass |
| Constraints equal or more restrictive | `validateAttenuation()` per type | Unit + glob attenuation tests | Pass |
| Restricted-glob subset on string patterns | `globAttenuation.js` | `globAttenuation.test.js` | Pass |
| Widening returns `delegation_widened` | `delegationVerifier.js` | Endpoint contract | Pass |
| Revoked credential in chain invalidates whole chain | Per-link revocation check | e2e-smoke chain revocation step | Pass |

### BR-5: Credential Revocation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Issuer revokes after issuance | `credentialIssuer.js` → `revokeCredential()` | e2e-smoke revocation step | Pass |
| Engine checks revocation during evaluation | Tier 1 via `revocationClient.js`; Tier 3 via `statusListChecker.js` | e2e-smoke revoked-credential re-evaluation | Pass |
| Tier 1 introspection with timeout/retries/failure mode | `revocationClient.js`, migration `009` | Engine unit suite | Pass |
| Bitstring Status List fetch + decode | `statusListChecker.js` | `vcJoseVerifier.test.js` status list suite | Pass |

### BR-6: Signed Audit Trail

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Every decision produces an audit record | `auditLogger.js` called on allow and deny | e2e-smoke audit retrieval; BB-40 | Pass |
| Pre-evaluation failures produce audit records | `evaluate.js` calls audit on 422 paths | 422 responses include `auditRecordId` | Pass |
| Record carries full provenance | Stored columns: credential, agent, issuer, action, resource, context, constraint results, decision, error | BB-40 | Pass |
| Record cryptographically signed | Synchronous sign-then-INSERT in `auditLogger.js`; verified by `GET /audit/verify/:id` | e2e-smoke verification step | Pass |
| Audit subsystem health observable | `GET /audit/health` reports unsigned backlog and last re-sign sweep | Engine unit suite | Pass |

### BR-7: Independent Deployment

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Runs as standalone service | `portauth-docker-compose.yml` | Compose stack | Pass |
| Documented API accepts credential plus request context | `POST /evaluate` accepts `{credential, presenterId, requestContext}` | e2e + BB | Pass |
| No external orchestrator dependency | Source inspection | No imports from external orchestrators | Pass |

### BR-8: Specification with Test Vectors

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Published specification | arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)) | Doc review | Pass |
| Test vector suite covers happy plus deny paths | `conformance/vectors/reference-core.json`, engine unit tests, e2e-smoke | `npm run conformance` 9/9; engine 106/106 | Pass |
| Vectors cover JWT, X.509, VC-JOSE, delegation, semantic mapping, stateful | Conformance corpus plus unit suites | Engine + conformance suites | Pass |

### BR-9: Minimum Viable Vocabulary

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Engine recognizes MVV identifiers | `semanticResolver.js`, migration `003`, `contextEnricher.js` | Engine semantic resolver tests | Pass |
| Industry profile support | `portauth_mapping_profiles` seeded with `insurance-claims-v1` | Constraint evaluator tests use `claimAmount`, `claimType` | Pass |
| Unknown identifiers fail-closed | Returns `semantic_identifier_unknown` | `constraintEvaluator.test.js` | Pass |

### BR-10: Pre-flight Governance Discovery

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Receiver publishes a governance manifest | `wellknown.js` returns signed manifest | e2e-smoke manifest step | Pass |
| Manifest assembled from state | `governanceState.js`, revision bumped by admin writes | Engine unit suite | Pass |
| Sender pre-flight SDK exists | `@portauth/preflight` + `portauth-preflight` CLI | `sdks/preflight` unit tests | Pass |
| Receiver remains authoritative | No pre-flight bypass path in `evaluate.js` | Code review | Pass |

### Multi-Principal Workflow (Section 7.5.1)

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Workflow policies stored versioned | `portauth_workflow_policies` (migration `011`) | Migration applied at startup | Pass |
| `requireAll`, `requireAny`, `threshold` combinators | `workflowEvaluator.js` | Engine workflow tests | Pass |
| Per-credential sub-decision audit | Audit row + `workflow_sub_audit_ids` | Engine workflow tests | Pass |
| Role mismatch denied | `workflow_role_mismatch` error | Engine workflow tests | Pass |

### Verifiable State Authority (Section 8.5)

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Allow-listed authorities | `portauth_state_authorities` (migration `010`) | Migration applied at startup | Pass |
| Proof verification + binding to request | `stateAuthorityClient.js` | Engine state-authority tests | Pass |
| Consumed proofs retained | `portauth_state_proofs` | Migration applied at startup | Pass |
| Failure modes (open/closed) | Per-authority + per-issuer configuration | Engine tests | Pass |
| Reference authority server | `state-authority-ref/` | Standalone Express app | Pass (not wired into compose) |

---

## Security and Hardening

- [SECURITY.md](SECURITY.md): reporting process, supported versions, hardening checklist.
- [CONTRIBUTING.md](CONTRIBUTING.md): development setup, PR checklist, DCO sign-off.
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md): Contributor Covenant 2.1.

Before any deployment beyond a local laptop, replace every value flagged in [SECURITY.md](SECURITY.md), terminate TLS in front of both services, and rotate the first audit and governance signing keys.

---

## License

See [LICENSE](LICENSE.md).
