# Architecture

PortAuth implements the three-layer authorization architecture described in the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)):

| Layer | Responsibility | What varies | What is constant |
|-------|---------------|-------------|-----------------|
| **Container** | Cryptographic envelope + transport | JWT, VC-JOSE, PKI |  |
| **Payload** | Authorization semantics |  | Agent ID, issuer ID, permissions, typed constraints |
| **Engine** | Evaluation + enforcement |  | Conjunctive, total, fail-closed processing model |

A credential issuer creates a signed authorization credential containing a complete authorization policy, who the agent is, what it may do, and under what conditions. The receiving engine verifies the credential, evaluates every embedded constraint against the actual request context, merges with receiver-local policy (most-restrictive-wins), and produces a cryptographically signed audit record of the decision.

## System diagram

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

## Components

| Directory | Purpose | Stack |
|-----------|---------|-------|
| [`portauth-engine/`](../portauth-engine/) | Credential verification, constraint evaluation, stateful authorization, local policy merge, audit, admin APIs, governance manifest, workflow composition | Node.js 20, Express, jose, pg |
| [`portauth-issuer/`](../portauth-issuer/) | Credential issuance (JWT, X.509 JWT, VC-JOSE), revocation, introspection, JWKS, Bitstring Status List | Node.js 20, Express, jose, pg |
| [`portauth-db/`](../portauth-db/) | PostgreSQL schema migrations (11 files, applied in order at startup) | PostgreSQL 16 Alpine |
| [`state-authority-ref/`](../state-authority-ref/) | Reference verifiable state authority server (paper Section 8.5) | Node.js 20, Express, jose |
| [`sdks/preflight/`](../sdks/preflight/) | `@portauth/preflight` sender-side discovery client and `portauth-preflight` CLI | Node.js 20, jose |
| [`conformance/`](../conformance/) | Machine-readable conformance vectors and runner | JSON, Node.js |
| [`opa/`](../opa/) | Advisory compatibility policy used by the local compose stack | OPA/Rego |

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

## Database schema

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
