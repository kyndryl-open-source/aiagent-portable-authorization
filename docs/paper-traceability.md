# Paper Traceability

Requirements traced from the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)) to implementation and test evidence.

## BR-1: Credential Issuance with Embedded Authorization Policy

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Credential contains all 4 authorization policy components | `credentialIssuer.js` builds JWT with `iss`, `sub`, `permissions`, `constraints` | e2e-smoke step "Issue credential"; BB-10 | Pass |
| JWS/JWT profile carries permissions and constraints as JWT claims | `credentialIssuer.js` signs with `jose.SignJWT` (ES256) | e2e-smoke + BB-10 | Pass |
| VC profile uses W3C VC Data Model 2.0 | `vcCredentialIssuer.js` builds `vc` claim with `credentials/v2` context | `vcJoseVerifier.test.js` | Pass |
| VC profile places permissions in `credentialSubject` | `vcCredentialIssuer.js` | `vcJoseVerifier.test.js` | Pass |
| Credential cryptographically signed by issuer | `keyManager.js` + `jose.SignJWT` | e2e-smoke verification step | Pass |
| Credential includes validity window | `nbf` / `exp` on Tier 1; `vc.validFrom` / `vc.validUntil` on Tier 3 | `vcJoseVerifier.test.js` expired-credential test | Pass |
| VC profile includes `credentialStatus` for revocation | `vcCredentialIssuer.js` adds `BitstringStatusListEntry` when configured | `vcJoseVerifier.test.js` status list suite | Pass |

## BR-2: Constraint Evaluation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Each typed constraint evaluated correctly | `constraintEvaluator.js` | `constraintEvaluator.test.js`; e2e-smoke deny steps; BB-22 | Pass |
| AND logic across constraints | `evaluateConstraints()` | `constraintEvaluator.test.js` | Pass |
| Unknown constraint types fail-closed | Default branch returns `{pass:false, error:"constraint_unknown"}` | `constraintEvaluator.test.js` | Pass |
| Missing context fields fail-closed | Field-presence check before evaluation | `constraintEvaluator.test.js`; semantic resolver | Pass |
| Detailed per-constraint reason trace | `evaluateConstraints()` returns `results[]` | e2e-smoke + BB-22 responses | Pass |

## BR-3: Most-Restrictive-Wins

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Effective grant = credential intersect local policy | `localPolicyMerge.js` | BB-22 (4500 local cap, 5000 credential, 4800 denied) | Pass |
| Credential ceiling never exceeded | `localPolicyMerge.js` narrows only | BB-22; architecture | Pass |

## BR-4: Delegation with Attenuation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Per-link verification | `delegationVerifier.js` | `/evaluate/batch` | Pass |
| Permission subset enforced | `validateAttenuation()` | Architecture + unit tests | Pass |
| Constraints equal or more restrictive | `validateAttenuation()` per type | Unit + glob attenuation tests | Pass |
| Restricted-glob subset on string patterns | `globAttenuation.js` | `globAttenuation.test.js` | Pass |
| Widening returns `delegation_widened` | `delegationVerifier.js` | Endpoint contract | Pass |
| Revoked credential in chain invalidates whole chain | Per-link revocation check | e2e-smoke chain revocation step | Pass |

## BR-5: Credential Revocation

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Issuer revokes after issuance | `credentialIssuer.js` → `revokeCredential()` | e2e-smoke revocation step | Pass |
| Engine checks revocation during evaluation | Tier 1 via `revocationClient.js`; Tier 3 via `statusListChecker.js` | e2e-smoke revoked-credential re-evaluation | Pass |
| Tier 1 introspection with timeout/retries/failure mode | `revocationClient.js`, migration `009` | Engine unit suite | Pass |
| Bitstring Status List fetch + decode | `statusListChecker.js` | `vcJoseVerifier.test.js` status list suite | Pass |

## BR-6: Signed Audit Trail

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Every decision produces an audit record | `auditLogger.js` called on allow and deny | e2e-smoke audit retrieval; BB-40 | Pass |
| Pre-evaluation failures produce audit records | `evaluate.js` calls audit on 422 paths | 422 responses include `auditRecordId` | Pass |
| Record carries full provenance | Stored columns: credential, agent, issuer, action, resource, context, constraint results, decision, error | BB-40 | Pass |
| Record cryptographically signed | Synchronous sign-then-INSERT in `auditLogger.js`; verified by `GET /audit/verify/:id` | e2e-smoke verification step | Pass |
| Audit subsystem health observable | `GET /audit/health` reports unsigned backlog and last re-sign sweep | Engine unit suite | Pass |

## BR-7: Independent Deployment

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Runs as standalone service | `portauth-docker-compose.yml` | Compose stack | Pass |
| Documented API accepts credential plus request context | `POST /evaluate` accepts `{credential, presenterId, requestContext}` | e2e + BB | Pass |
| No external orchestrator dependency | Source inspection | No imports from external orchestrators | Pass |

## BR-8: Specification with Test Vectors

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Published specification | arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)) | Doc review | Pass |
| Test vector suite covers happy plus deny paths | `conformance/vectors/reference-core.json`, engine unit tests, e2e-smoke | `npm run conformance` 9/9; engine 106/106 | Pass |
| Vectors cover JWT, X.509, VC-JOSE, delegation, semantic mapping, stateful | Conformance corpus plus unit suites | Engine + conformance suites | Pass |

## BR-9: Minimum Viable Vocabulary

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Engine recognizes MVV identifiers | `semanticResolver.js`, migration `003`, `contextEnricher.js` | Engine semantic resolver tests | Pass |
| Industry profile support | `portauth_mapping_profiles` seeded with `insurance-claims-v1` | Constraint evaluator tests use `claimAmount`, `claimType` | Pass |
| Unknown identifiers fail-closed | Returns `semantic_identifier_unknown` | `constraintEvaluator.test.js` | Pass |

## BR-10: Pre-flight Governance Discovery

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Receiver publishes a governance manifest | `wellknown.js` returns signed manifest | e2e-smoke manifest step | Pass |
| Manifest assembled from state | `governanceState.js`, revision bumped by admin writes | Engine unit suite | Pass |
| Sender pre-flight SDK exists | `@portauth/preflight` + `portauth-preflight` CLI | `sdks/preflight` unit tests | Pass |
| Receiver remains authoritative | No pre-flight bypass path in `evaluate.js` | Code review | Pass |

## Multi-Principal Workflow (Section 7.5.1)

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Workflow policies stored versioned | `portauth_workflow_policies` (migration `011`) | Migration applied at startup | Pass |
| `requireAll`, `requireAny`, `threshold` combinators | `workflowEvaluator.js` | Engine workflow tests | Pass |
| Per-credential sub-decision audit | Audit row + `workflow_sub_audit_ids` | Engine workflow tests | Pass |
| Role mismatch denied | `workflow_role_mismatch` error | Engine workflow tests | Pass |

## Verifiable State Authority (Section 8.5)

| Requirement | Implementation | Test | Result |
|-------------|---------------|------|--------|
| Allow-listed authorities | `portauth_state_authorities` (migration `010`) | Migration applied at startup | Pass |
| Proof verification + binding to request | `stateAuthorityClient.js` | Engine state-authority tests | Pass |
| Consumed proofs retained | `portauth_state_proofs` | Migration applied at startup | Pass |
| Failure modes (open/closed) | Per-authority + per-issuer configuration | Engine tests | Pass |
| Reference authority server | `state-authority-ref/` | Standalone Express app | Pass (not wired into compose) |
