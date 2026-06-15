# PortAuth A2A Extension v1 (DRAFT)

Status: Draft. This document specifies how PortAuth authorization credentials are
carried by, and evaluated within, a Google Agent2Agent (A2A) message flow.

| Field | Value |
|---|---|
| Extension URI | `https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1` |
| Short name | `portauth-a2a/v1` |
| Spec version | 1.0.0-draft.1 |
| Engine compatibility | `aiagent-portable-authorization` v0.1.0 and later |
| A2A spec compatibility | A2A 0.3.x |
| License | MIT |

## 1. Goals

1. Let an A2A caller present a Verifiable Credential (VC) on any A2A
   `SendMessage` request without forking the A2A schema.
2. Let an A2A receiver authorize the request by calling the PortAuth engine,
   with no engine-side changes required.
3. Support N-hop delegation chains (a request that crosses two or more A2A
   receivers) using PortAuth's existing delegation verifier.
4. Let a caller decide in advance whether its VC will be accepted, by
   discovering the receiver's policy through the Agent Card.

Non-goals for v1:

- Push-notification (server-initiated) message authorization. Deferred to v2.
- VC issuance flows. The extension assumes a VC is already in hand.
- A new VC format. The extension transports the existing PortAuth credential
  shape (Tier 1 JWT, Tier 2 x5c JWT, or Tier 3 vc+jwt).

## 2. A2A compliance

The extension follows the standard A2A extension contract.

### 2.1 Declaration

The receiver MUST advertise the extension in its Agent Card under
`capabilities.extensions[]`:

```json
{
  "capabilities": {
    "extensions": [
      {
        "uri": "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1",
        "description": "PortAuth authorization. Callers MUST attach a Verifiable Credential.",
        "required": true,
        "params": {
          "governanceManifestUrl": "https://bank.example/.well-known/agent-governance",
          "acceptedProfiles": ["tier1-jwt", "tier3-vc-jose"],
          "acceptedAudiences": ["did:web:bank.example/payments"]
        }
      }
    ]
  }
}
```

`required: true` means messages without the activation header SHOULD be
rejected. `required: false` means the receiver evaluates with PortAuth only
when the caller activates the extension.

`params.governanceManifestUrl` MUST resolve to a PortAuth governance manifest
(see `GET /.well-known/agent-governance` on the engine). Callers SHOULD fetch
and verify this manifest before sending.

### 2.2 Activation

The caller MUST set the standard A2A extension service-parameter header (see
A2A spec §3.2.6 and §14.2.2):

```
A2A-Extensions: https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1
```

Note: the header is `A2A-Extensions` (no `X-` prefix). On a successful response
the receiver MUST echo the same header. Receivers MUST NOT skip evaluation
based on the header's absence alone; they MUST also inspect `message.metadata`
for any keys under the extension URI (see §8 item 5).

### 2.3 Data placement

All extension data goes under `message.metadata`, keyed by the extension URI
plus a slash-separated path. No new top-level fields are introduced on the
A2A `Message` or `Task` objects.

| Metadata key | Required | Type | Meaning |
|---|---|---|---|
| `<URI>/vc` | yes | string (compact JWS) | The credential being presented. |
| `<URI>/vcFormat` | yes | string | One of `tier1-jwt`, `tier2-x509-jwt`, `tier3-vc-jose`. Selects the engine verifier path. |
| `<URI>/presenterId` | yes | string | Stable identifier of the presenting agent (`sub` of its own identity), passed to `/evaluate` as `presenterId`. |
| `<URI>/proofOfPossession` | no | object | Per-request proof signed by the credential's bound key. Maps 1:1 to PortAuth's `proofOfPossession` field. |
| `<URI>/idempotencyKey` | no | string | If present, forwarded to the engine for stateful constraint enforcement. |
| `<URI>/delegationChain` | no | string[] | Compact JWSs for upstream credentials in a delegation chain. Used only for N-hop calls (see §4). |

The full metadata key for the credential itself is therefore:

`https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1/vc`

### 2.4 PoC coverage vs deferred behavior

The v1 spec defines the full data placement above. The reference PoC in
`extensions/a2a/poc/` exercises only the subset required to prove the
metadata round-trip end-to-end:

| Metadata key | Spec | v0.1 PoC | Deferred to |
|---|---|---|---|
| `<URI>/vc` (Tier 1 JWT) | yes | yes | n/a |
| `<URI>/vc` (Tier 3 vc+jose) | yes | no | Phase 2 (receiver middleware) |
| `<URI>/vcFormat` | yes | yes (forwarded) | n/a |
| `<URI>/presenterId` | yes | yes | n/a |
| `<URI>/idempotencyKey` | yes | yes (forwarded) | n/a |
| `<URI>/proofOfPossession` | yes | no | Phase 2 |
| `<URI>/delegationChain` (N-hop) | yes | no | Phase 2 |

Receivers built against this spec MUST support every row marked `yes` under
Spec. The PoC's narrower coverage is a property of the demo, not a relaxation
of the spec.

## 3. Receiver evaluation

On receipt of an A2A request that carries the extension, the receiver MUST:

1. Validate the `A2A-Extensions` request header against its own declaration.
   If the extension is required but absent, return
   `ExtensionSupportRequiredError` (A2A spec §3.3.2).
2. Read `<URI>/vc`, `<URI>/vcFormat`, and `<URI>/presenterId` from
   `message.metadata`. If any is missing, return A2A error `invalid_request`.
3. Translate A2A request fields into a PortAuth `requestContext`:

| A2A source | PortAuth requestContext field | Notes |
|---|---|---|
| `message.role` + `message.parts[].kind`/skill invoked | `action` | The receiver maps to its permission namespace, e.g. `payments:transfer`. |
| Target skill identifier or resource path | `resource` | Receiver-defined. The bank example uses `urn:bank:accounts/<account_id>`. |
| `message.taskId` or generated UUID | `requestId` | Surfaces in audit. |
| Caller `presenterId` from metadata | `presenterId` | Required by the engine. |

4. POST to the engine `/evaluate` (single VC) or `/evaluate/batch`
   (delegation chain) with the translated body.
5. On `decision: "allow"`, perform the action and reply with the normal A2A
   success envelope. On `decision: "deny"`, map the engine `error` code into
   an A2A error response (see §6) and return.
6. Echo the extension URI in the response's `A2A-Extensions` header.

The receiver MUST persist `auditRecordId` from the engine response on the
task record so the audit row can be located later.

## 4. Two-decision and N-hop chains

A2A explicitly contemplates agents that re-delegate work to downstream agents.
PortAuth handles this through delegation chains.

### 4.1 Single-hop (one A2A receiver, no further delegation)

The caller posts the original VC under `<URI>/vc`. The receiver evaluates
once via `POST /evaluate`. This is the most common case.

### 4.2 Two-hop (the receiver re-delegates to a downstream agent)

The first receiver, on accepting the request, issues a NEW credential
attenuated from the inbound one (the engine's restricted-glob attenuation
applies). It then makes an outbound A2A call to a downstream receiver with:

- `<URI>/vc` = the newly-issued attenuated credential.
- `<URI>/delegationChain` = `[inboundVc]` (the credential the first receiver
  was given).
- `<URI>/presenterId` = the first receiver's own identifier.

The downstream receiver MUST call `POST /evaluate/batch` with
`credentials = [...delegationChain, vc]`. The engine verifies the chain (each
link's audience matches the next presenter, each link narrows the parent),
runs constraint and local-policy evaluation against the leaf, and returns a
single `decision`.

### 4.3 Three-hop and beyond

The same pattern extends. Each intermediate receiver appends the inbound VC
to `delegationChain` before issuing its own attenuated credential and
forwarding. The engine's `verifyDelegationChain` is depth-agnostic.

### 4.4 Two-decision pattern

Where Iron Book runs two separate policy decisions per handoff (requester
role and executor role), PortAuth folds both into a single `/evaluate/batch`
call: chain verification implicitly answers "was the requester allowed to
delegate this?" and constraint + local policy evaluation answers "is the
executor allowed to act?". The receiver therefore does ONE engine call per
inbound A2A request regardless of chain depth, which is operationally
simpler.

## 5. Caller preflight

A caller SHOULD perform preflight before sending a real A2A message,
especially on high-cost or visible operations.

The flow:

1. Fetch the receiver's Agent Card.
2. Read the extension declaration. If `required` and the caller cannot
   satisfy it, abort.
3. Fetch and verify the governance manifest from
   `params.governanceManifestUrl` using `@portauth/preflight`
   `fetchManifest()` + `verifyManifest()`.
4. Run the five preflight rules from `@portauth/preflight` against the
   intended VC:
   - `profileMatch` (credential format is accepted)
   - `constraintTypes` (every constraint type is recognized)
   - `vocabulary` (every semantic field is recognized)
   - `trustAnchors` (the credential's issuer is in the manifest)
   - `contextFields` (every required context field will be present in the
     planned request)
5. If any rule fails with severity `error`, do not send.

The `@portauth/a2a` SDK provides `preflight(agentCardUrl, vc, intendedAction)`
that wraps these steps.

## 6. Error mapping

The receiver maps PortAuth engine error codes to A2A error responses as
follows. All engine error codes are stable strings.

| Engine `error` | HTTP | A2A error code | Notes |
|---|---|---|---|
| `validation_error` | 400 | `invalid_request` | Caller bug. |
| `credential_incomplete` | 422 | `invalid_request` | VC is malformed. |
| `signature_invalid` | 422 | `auth_failed` | |
| `credential_expired` | 422 | `auth_failed` | |
| `issuer_untrusted` | 422 | `auth_failed` | |
| `audience_mismatch` | 422 | `auth_failed` | The VC was issued for a different receiver. |
| `proof_of_possession_failed` | 422 | `auth_failed` | |
| `subject_binding_mismatch` | 422 | `auth_failed` | `presenterId` does not match the VC's `sub`. |
| `credential_revoked` | 422 | `auth_failed` | |
| `tier1_revocation_unavailable` | 422 | `auth_unavailable` | Receiver MAY retry. |
| `permission_denied` | 200 | `forbidden` | VC valid, but the requested action is out of scope. |
| `constraint_failed` | 200 | `forbidden` | Default code returned when a typed bound rejects the request. |
| `local_policy_denied` | 200 | `forbidden` | Receiver-side policy (KYC, AML, fraud) rejected. |
| `delegation_widened` | 200 | `forbidden` | A chain link tried to escalate. |
| `audit_signing_failed` | 500 | `internal_error` | Receiver MAY retry. |

Individual constraint evaluators MAY surface a more specific `error` value
(for example `context_field_missing`, `numeric_limit_exceeded`,
`semantic_identifier_unknown`, `temporal_window_violated`,
`enumerated_list_violated`, `pattern_violated`, `cumulative_limit_exceeded`,
`usage_limit_exceeded`). Receivers SHOULD treat any unrecognized engine
`error` that arrives with `decision: "deny"` as `forbidden`, preserve the
raw code in the A2A error `data` payload for diagnostics, and SHOULD log
for MVV alignment drift.

Receivers SHOULD include the engine `auditRecordId` in the A2A error
response's `data` field for debuggability.

## 7. Worked example: UC-03 banking cross-boundary

Scenario: a customer's personal-finance agent (external) tells a bank's
intake agent to pay a USD 2,400 utility bill. The intake agent delegates
the actual transfer to the bank's payments-execution agent.

### 7.1 Customer issues the VC

Customer-side issuer (`POST /credentials/issue-vc`):

```json
{
  "agentId": "did:key:zPersonalFinanceAgent",
  "audience": "did:web:bank.example:intake",
  "permissions": ["payments:bill_pay"],
  "constraints": [
    {
      "type": "EnumeratedListConstraint",
      "field": "core.recipient_id",
      "allowed": ["utilities", "telco", "mortgage"]
    },
    {
      "type": "CumulativeLimitConstraint",
      "id": "monthly-cap",
      "field": "core.amount",
      "maxAmount": 2500,
      "period": "month",
      "scope": "agent"
    },
    {
      "type": "EnumeratedListConstraint",
      "field": "core.geo_region",
      "allowed": ["US"]
    }
  ],
  "validUntil": "2026-09-09T00:00:00Z"
}
```

### 7.2 Customer's agent sends the A2A request to the bank intake

Using the HTTP+JSON/REST binding (A2A spec §11):

```http
POST /message:send HTTP/1.1
Host: bank.example
Content-Type: application/a2a+json
A2A-Extensions: https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1
A2A-Version: 0.3
```

```json
{
  "message": {
    "messageId": "a8f5f167-f44f-4964-a6b1-2e0c3a2f4d31",
    "role": "ROLE_USER",
    "parts": [{ "text": "Please pay my electric bill $2,400." }],
    "extensions": [
      "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1"
    ],
    "metadata": {
      "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1/vc":
        "eyJhbGciOi...redacted...sig",
      "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1/vcFormat":
        "tier3-vc-jose",
      "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1/presenterId":
        "did:key:zPersonalFinanceAgent",
      "https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1/idempotencyKey":
        "bill-2026-06-electric"
    }
  }
}
```

Using the JSON-RPC binding (A2A spec §9), the same payload is wrapped as
`{"jsonrpc":"2.0","id":1,"method":"SendMessage","params": <body above>}` and
POSTed to the JSON-RPC endpoint declared in the Agent Card. The `A2A-Extensions`
and `A2A-Version` headers apply identically (A2A spec §9.2).

### 7.3 Bank intake calls the engine

```http
POST /evaluate HTTP/1.1
Host: portauth-engine.bank.example
```

```json
{
  "credential": "eyJhbGciOi...redacted...sig",
  "presenterId": "did:key:zPersonalFinanceAgent",
  "requestContext": {
    "action": "payments:bill_pay",
    "resource": "urn:bank:utilities/electric",
    "requestId": "task-7d3c",
    "claimAmount": 2400,
    "currency": "USD",
    "recipient": "utilities",
    "geoRegion": "US",
    "idempotencyKey": "bill-2026-06-electric"
  }
}
```

Engine response on allow:

```json
{
  "decision": "allow",
  "auditRecordId": "ar_01HXYZ...",
  "verification": { "valid": true, "checks": ["signature", "issuer_trusted", "audience", "expiry"] },
  "permissionMatch": { "matched": true, "requested": "payments:bill_pay", "granted": ["payments:bill_pay"] },
  "constraintResults": [
    { "type": "EnumeratedListConstraint", "field": "core.recipient_id", "pass": true },
    { "type": "CumulativeLimitConstraint", "field": "core.amount", "pass": true, "remaining": 100 },
    { "type": "EnumeratedListConstraint", "field": "core.geo_region", "pass": true }
  ],
  "localPolicyApplied": true
}
```

### 7.4 Bank intake re-delegates to payments execution

Intake issues an attenuated VC scoped to a single transfer, then makes an A2A
call to `did:web:bank.example:payments`:

```json
{
  "agentId": "did:web:bank.example:intake",
  "audience": "did:web:bank.example:payments",
  "permissions": ["payments:bill_pay"],
  "constraints": [
    { "type": "NumericLimitConstraint", "field": "core.amount", "operator": "lte", "value": 2400 },
    { "type": "EnumeratedListConstraint", "field": "core.recipient_id", "allowed": ["utilities"] }
  ],
  "validUntil": "2026-06-11T01:00:00Z",
  "delegatedFrom": "<original-vc-id>"
}
```

The outbound A2A call carries `<URI>/vc` = newly attenuated credential and
`<URI>/delegationChain` = `[original customer VC]`.

### 7.5 Payments execution calls the engine batch endpoint

```json
{
  "credentials": ["<customer-vc>", "<intake-attenuated-vc>"],
  "presenterId": "did:web:bank.example:intake",
  "requestContext": {
    "action": "payments:bill_pay",
    "resource": "urn:bank:transfers/wire/abc123",
    "claimAmount": 2400,
    "currency": "USD",
    "recipient": "utilities",
    "geoRegion": "US"
  }
}
```

The engine verifies the chain (intake's audience matched the first VC,
intake's constraints narrowed the parent), runs constraint + local policy on
the leaf, and returns one `decision`. The bank's local policy is what
escalates to HITL if `claimAmount > 1000` or runs the AML watchlist; that
logic lives in the engine's `localPolicyMerge` and is out of scope for this
spec.

## 8. Security considerations

1. **VC confidentiality.** A VC in `message.metadata` is end-to-end visible
   on every A2A hop. Receivers MUST NOT log raw VCs at INFO level. Use only
   the `credentialId` claim for diagnostics.
2. **Replay.** Receivers SHOULD pass the A2A `taskId` or generate a
   `requestId` and supply it to the engine as `idempotencyKey` for any
   action that has stateful constraints.
3. **Proof of possession.** Tier 3 VCs SHOULD include a `cnf` claim binding
   the credential to a public key, and callers SHOULD attach a
   `<URI>/proofOfPossession` signed with the bound key. Without PoP a stolen
   VC is bearer-equivalent.
4. **Required vs optional.** A receiver that publishes `required: true` and
   then accepts unauthorized requests during outages defeats the
   declaration. Failure-mode behavior MUST match the declaration.
5. **Header trust.** The `A2A-Extensions` header is not signed. Receivers
   MUST NOT skip evaluation based on its absence alone; they MUST also
   check `message.metadata` for any keys under the extension URI.

## 9. Versioning

The extension URI ends in `/v1`. Backwards-incompatible changes will mint a
new URI (`/v2`). Forwards-compatible additions (new optional metadata keys)
will be documented as `1.x` revisions of this spec.

## 10. Relation to other work

### 10.1 A2A Iron Book Extension (Identity Machines)

Iron Book pioneered the metadata-keyed A2A extension pattern and the
two-decision authorization framing. PortAuth's A2A extension adopts the same
A2A compliance recipe (declaration, activation, data placement) and the same
mental model. It differs in:

- Credential format: real Verifiable Credentials (JWS-signed) instead of
  one-shot bearer tokens.
- Delegation: N-hop attenuation chains with restricted-glob verification
  instead of a fixed two-role pattern.
- Trust: self-hosted governance manifests with versioned vocabulary instead
  of a SaaS registry.
- Hosting: drop-in OSS engine, no API key, no third-party dependency.

Both extensions are intentionally interoperable at the A2A layer: a receiver
can declare both, and a caller can satisfy whichever the receiver
prefers.

### 10.2 OAuth 2.0 bearer tokens

OAuth bearer tokens carry no typed bounds and no delegation context. PortAuth
VCs do, which is what enables the per-action evaluation in §3 and the
chain evaluation in §4. An A2A receiver can accept both: bearer for
low-stakes read operations, this extension for stateful or cross-domain
actions.

## Appendix A. JSON Schema (informative)

A formal JSON Schema for the `message.metadata` entries will accompany the
first non-draft version of this spec. The current draft codifies the shape
in §2.3.

## Appendix B. Reference implementation

- `extensions/a2a/poc/` in this repository runs the §7 UC-03 flow end-to-end
  against the standard PortAuth compose stack. It is the smallest possible
  artifact that demonstrates the metadata-envelope round-trip.
- `extensions/a2a/sdk/` (planned) will package `@portauth/a2a` for callers.
- `extensions/a2a/middleware/` (planned) will package
  `@portauth/a2a-receiver` for receivers.
- `extensions/a2a/reference-receiver/` and `reference-presenter/` (planned)
  will provide minimal A2A agents wired against the SDK and middleware.
