# Credentials and Constraints

## Credential profiles

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

## Constraint types

| Type | Properties | Evaluation | Attenuation Rule |
|------|-----------|------------|-----------------|
| **NumericLimitConstraint** | `field`, `operator` (eq/lt/lte/gt/gte), `value`, `currency?` | Extract field, apply operator | Delegate must be equal or more restrictive |
| **TemporalWindowConstraint** | `field`, `validFrom`, `validUntil`, `timezone`, `daysOfWeek?` | Convert to TZ, check time + day | Delegate window must be subset |
| **EnumeratedListConstraint** | `field`, `allowed?`, `denied?` | Member check; deny takes precedence | Delegate allowed ⊆ parent allowed |
| **StringPatternConstraint** | `field`, `pattern`, `matchType` (exact/prefix/suffix/restricted_glob) | Pattern match per matchType | Delegate pattern matches subset of parent |
| **CumulativeLimitConstraint** | `id`, `field`, `maxAmount`, `period`, `scope` | Transactional counter reservation | Delegate must preserve or lower the cumulative limit |
| **UsageLimitConstraint** | `id`, `maxUses`, `period`, `scope` | Transactional use counter reservation | Delegate must preserve or lower the use limit |

**Processing model:** conjunctive (AND), total (every constraint evaluated), fail-closed (unknown types, missing fields → deny). Stateful constraints are deferred during stateless evaluation and enforced transactionally after receiver-local policy passes. Use `idempotencyKey` or the `Idempotency-Key` header to avoid duplicate reservations on retries.
