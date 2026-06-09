# Evaluation Pipeline

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

## Audience and Proof of Possession

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

A reference authority server lives under `../state-authority-ref/` (Node 20 / Express / jose). It is not started by `../portauth-docker-compose.yml`; see [Operations: Feature Status](operations.md#feature-status).
