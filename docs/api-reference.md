# API Reference

## Issuer Service (`portauth-issuer`, port 3100)

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

## Engine Service (`portauth-engine`, port 3200)

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

## HTTP status semantics

| Status | Meaning |
|--------|---------|
| `200 OK` | Credential verified, authorization decision computed (both `allow` and `deny`) |
| `201 Created` | Credential or resource created successfully |
| `400 Bad Request` | Request body malformed, never reached credential processing |
| `422 Unprocessable Entity` | Credential parsed but unusable (invalid signature, untrusted issuer, expired, revoked, subject mismatch) |
| `500 Internal Server Error` | Unexpected failure |

## Error codes

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

## End-to-end walkthrough

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
