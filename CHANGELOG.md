# Changelog

All notable changes to this project are documented in this file. The format is loosely based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project follows [Semantic Versioning](https://semver.org/).

Pre-1.0 releases may include breaking schema, API, or wire-format changes.

## [Unreleased]

### Added

- **Synchronous, self-healing audit signing** (paper Section 7.10, 7.16.7). `createAuditRecord` now signs the canonical record before insert in a single transaction. A startup re-sign sweep (`resignUnsignedAuditRecords`) processes any unsigned rows in batches of 100. A new `GET /audit/health` endpoint reports `unsignedOlderThan60s`, `lastResignSweepAt`, and `lastResignSweepCount`. Signing failures during evaluation surface as HTTP 500 with `error: "audit_signing_failed"` and no row is written.
- **Restricted-glob attenuation** (paper Section 7.3.4, 7.7.3). `isRestrictedGlobRefinement(parent, delegate)` implements segment-based subset checking. `**` absorbs segments, `*` matches `[^/]*` within a segment, literal-token subset checks. Wired into delegation verification, replacing the prior literal-prefix shortcut.
- **MVV identifier coverage** (paper Section 7.12.2). Built-in fallbacks added for `core.issuer_id`, `core.subject_id`, `core.presenter_id`, `core.audience_id`, `core.permission`, `core.valid_from`, `core.valid_until`, `core.delegator_id`, `core.workflow_role`, `core.workflow_step_id`, `core.resource_type`, `core.total_budget`, `core.geo_region`, `core.ip_address`, `core.request_id`. A new `enrichRequestContext()` helper populates these from the credential payload and request headers (with UUIDv4 fallback for `requestId`).
- **Tier 1 revocation hardening** (paper Section 7.9, 7.16.4). New `revocationClient.js` introduces `PORTAUTH_TIER1_REVOCATION_TIMEOUT_MS`, `PORTAUTH_TIER1_REVOCATION_RETRIES`, and `PORTAUTH_TIER1_REVOCATION_FAILURE_MODE` (`open` or `closed`). Per-issuer overrides live in `portauth_trusted_issuers` (`revocation_mechanism`, `revocation_failure_mode`, `revocation_timeout_ms`, `revocation_retries`). New error code `tier1_revocation_unavailable`. Migration `009-revocation-hardening.sql` also creates the `portauth_x509_ocsp_cache`, `portauth_x509_crl_cache`, and `portauth_x509_crl_entries` tables. **OCSP and CRL ASN.1 parsing is not yet implemented**; see *Known limitations*.
- **Governance manifest derived from state** (paper Section 7.13, 7.14). `governanceState.js` queries `portauth_trusted_issuers` and `portauth_mapping_profiles` at manifest assembly time. A revision counter (`bumpGovernanceRevision`) is bumped from `routes/admin.js` on issuer create/update/delete so the cached manifest invalidates after admin writes.
- **Verifiable state authority client** (paper Section 8.5.1, 8.5.2). `stateAuthorityClient.js` verifies JWS-signed reservation proofs from an allow-listed authority. Error codes: `state_authority_untrusted`, `state_authority_unreachable`, `state_authority_proof_invalid`. Migration `010-state-authority.sql` creates `portauth_state_authorities` and `portauth_state_proofs`. A reference authority server lives in `state-authority-ref/`. **The reference authority is not yet wired into the compose file** and there is no built-in two-engine race test; see *Known limitations*.
- **Multi-principal workflow composition** (paper Section 7.5.1). `POST /evaluate/workflow` endpoint with `requireAll`, `requireAny`, and `threshold` combinators. Per-credential sub-decisions are recorded; one wrapping audit row captures the policy id and child audit ids. Migration `011-workflow-policies.sql` creates `portauth_workflow_policies` and extends `portauth_audit_records` with `workflow_policy_id` and `workflow_sub_audit_ids`.
- **Pre-flight discovery SDK** (paper Section 7.13). `@portauth/preflight` package under `sdks/preflight/`. Exposes `preflightCheck()` and a `portauth-preflight` CLI with exit codes `0` (compatible), `1` (operational error), `2` (incompatible). Verifies the receiver manifest's JWS, then runs five rules: profile match, constraint types, vocabulary, trust anchors, required context fields. **The SDK is not yet exercised by the e2e smoke**; see *Known limitations*.
- **OSS hygiene** baseline: `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, a CI workflow under `.github/workflows/ci.yml`, and a paper-aligned README rewrite.

### Changed

- Migration files `001-issuer-tables.sql`, `002-engine-tables.sql`, `004-vc-did-support.sql` now use `CREATE INDEX IF NOT EXISTS` / `CREATE UNIQUE INDEX IF NOT EXISTS` so the issuer and engine, which both run `runMigrations()` against the shared database at startup, do not race.
- `portauth-issuer/src/schemas/issue-request.json` enumerates `UsageLimitConstraint` and `CumulativeLimitConstraint`, and moves the `field` requirement into per-type branches so usage-counter constraints (which do not have a `field`) validate.
- `portauth-docker-compose.yml` sets `DB_SSL: "false"` on the issuer and engine services so connections to the in-network `postgres` host do not auto-enable SSL against an Alpine PostgreSQL that does not support it.
- `portauth-engine/test/e2e-smoke.sh` no longer triggers `set -e` on the first `((var++))` increment. The revoked re-evaluation step accepts the engine's HTTP 422 response. Step 12 deletes its local policy before step 13 (stateful) runs.

### Fixed

- Audit-signing race that previously allowed an unsigned row to be observed via `GET /audit/records/:id` before the asynchronous signer reached it.
- Restricted-glob attenuation accepted some delegate patterns that widened, rather than narrowed, the parent pattern.
- Local policies leaked across smoke runs and falsely denied unrelated request contexts.

### Known limitations

These are recognized gaps. They are out of scope for this release and tracked separately.

- **OCSP and CRL (Tier 2)**: schema and per-issuer config columns exist, but no ASN.1 parsing or wire client is implemented. Tier 2 revocation falls back to the local revoked-certificate registry plus the `PORTAUTH_X509_REVOCATION_POLICY` soft/hard-fail switch.
- **Tier 3 Bitstring Status List e2e**: implementation exists end-to-end, but the smoke script does not yet exercise issue then revoke then deny via status list.
- **State authority in compose**: the reference state authority is not started by `portauth-docker-compose.yml` and there is no two-engine race integration test.
- **Workflow and preflight in smoke**: covered by unit tests only.
- **Conformance vectors** for revocation hardening, state authority, glob attenuation, and workflow combinators are not yet in `conformance/vectors/reference-core.json`.
- **Migration runner**: each migration file is one `pool.query(sql)` call. A partial failure can leave the database in an inconsistent state. A real migrator with a `schema_migrations` ledger and per-file transactions is recommended before any production use.

## [0.1.0-template] Initial repository scaffold

### Added

- Repository template structure, default branch protection conventions, and pre-commit configuration.

