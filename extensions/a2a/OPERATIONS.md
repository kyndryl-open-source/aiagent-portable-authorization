# PortAuth A2A — Operations Guide

This guide covers running the PortAuth A2A receiver in production: secret
handling, the monitoring surface, alerting, and recovery procedures. The
reference receiver under `reference-receiver/` implements the seams described
here; a real deployment supplies its own secrets, datastore, and telemetry
backend through the same options.

## 1. Secret inventory

| Secret | Where | Purpose | Notes |
|---|---|---|---|
| `A2A_RECEIVER_BEARER_TOKEN` | receiver | Authenticates inbound `/message:send` when `A2A_RECEIVER_AUTH_MODE=bearer` | Compared with a timing-safe digest |
| `A2A_ENGINE_AUTH_TOKEN` | receiver → engine | Bearer attached to `/evaluate` and `/evaluate/batch` | Must equal the engine's `ENGINE_EVALUATE_AUTH_TOKEN`; prefer mTLS where available |
| `ENGINE_EVALUATE_AUTH_TOKEN` | engine | Enforces bearer auth on `/evaluate*` when set | Pairs with the receiver's `A2A_ENGINE_AUTH_TOKEN`; unset = open (dev/compose) |
| `A2A_TASK_READ_TOKEN` | receiver | Guards `/tasks` lookup when `A2A_ENABLE_TASK_LOOKUP=true` | |
| `A2A_METRICS_TOKEN` | receiver | Guards `/metrics` when set | Optional; otherwise scrape over a private network |
| TLS private key | reverse proxy | Terminates TLS in front of the receiver | See [TLS.md](TLS.md); never commit real keys |
| Issuer signing keys (JWKS) | issuer/engine | Sign and verify VCs | Rotate by `kid` |
| Verifiable Credentials | in transit (`message.metadata`) | The authorization grant | Bearer-equivalent without proof-of-possession |

## 2. Secret handling rules

1. **Fail closed.** The receiver refuses to start when a required secret is
   missing or looks like a development placeholder (`*-change-me`, `changeme`,
   `dev-*`, `secret`, etc.). `resolveReceiverSecrets()` / `assertReceiverSecrets()`
   enforce this. For local development only, set `A2A_DEV_INSECURE=true`.
2. **No placeholders in production.** Any supplied token that matches a
   placeholder pattern is rejected even when it is not strictly required.
3. **Externalize.** Inject secrets at runtime from a secret manager (Vault, AWS
   Secrets Manager, GCP Secret Manager, sealed Kubernetes Secrets). Never bake
   them into an image layer or a committed compose/env file.
4. **Least privilege.** The admin key that registers issuers must not be the
   credential the receiver uses to call `/evaluate`. Keep them separate.
5. **Authenticate the engine hop, on both ends.** The receiver attaches
   `A2A_ENGINE_AUTH_TOKEN` to every `/evaluate*` call, **and** the engine
   enforces it when `ENGINE_EVALUATE_AUTH_TOKEN` is set — so a caller cannot
   bypass the receiver and reach the evaluator directly. Set both to the same
   value (or front the engine with mTLS). The receiver warns at startup when its
   token is unset; the engine warns when its gate is unset.
6. **Make high-value VCs non-bearer.** Require `cnf` + proof-of-possession on
   sensitive permissions so a captured VC is not replayable (SPEC §8). Turn on
   the receiver's mandatory-PoP policy for those actions (`A2A_REQUIRE_POP=true`
   for all requests, or `A2A_POP_REQUIRED_ACTIONS=action1,action2` for specific
   ones; programmatically, the `requireProofOfPossession` option accepts a
   boolean or a predicate). A request missing `<URI>/proofOfPossession` for a
   covered action is rejected with 401 `PORTAUTH_POP_REQUIRED` before the engine
   is called. The engine still verifies the proof cryptographically; the receiver
   policy makes its presence non-optional at the boundary.

### Rotation

- **Bearer/engine tokens:** roll with an overlap window — accept the old and new
  value simultaneously while callers migrate, then retire the old one.
- **Issuer signing keys:** publish a new key with a new `kid` in the JWKS, sign
  new VCs with it, and retire the old key only after the maximum VC TTL has
  elapsed so in-flight credentials still verify.
- **TLS:** automate renewal; the reference dev cert under `tls/` is gitignored
  and for local use only.

## 3. Monitoring surface

The receiver guard emits one structured lifecycle event per request through the
injected `onEvent` callback. The reference receiver feeds these into a metrics
collector and a redacting JSON logger.

### Events (`onEvent`)

| `type` | When | Key fields |
|---|---|---|
| `rejected` | Request fails validation/activation before the engine | `reason`, `status`, `durationMs` |
| `in_progress` | Duplicate arrives while the first is still pending | `taskId`, `idempotencyKey` |
| `replay` | Idempotent replay of a terminal decision | `lifecycleState`, `auditRecordId` |
| `decision` | Engine returned allow/deny, or the engine failed | `decision`, `engineError`, `auditRecordId`, `engineDurationMs`, `durationMs` |
| `persistence_failed` | Task store write failed | `phase` (`pending`/`final`) |

Every event carries a `requestId`, which is also returned to the caller as the
`X-Request-Id` response header and forwarded to the engine as `X-Request-Id` —
use it to correlate caller, receiver, and engine logs. Decision and replay
events carry `auditRecordId`, the join key to the engine's authoritative audit
row.

### Metrics (`GET /metrics`, Prometheus text)

Enable with `A2A_ENABLE_METRICS=true` (guard with `A2A_METRICS_TOKEN`):

- `portauth_a2a_decisions_total{decision}` — allow / deny / error.
- `portauth_a2a_rejected_total{reason}` — pre-engine rejections.
- `portauth_a2a_replays_total` — idempotent replays served.
- `portauth_a2a_persistence_failures_total{phase}` — should stay at 0.
- `portauth_a2a_engine_call_duration_ms_{sum,count}` — engine latency.
- `portauth_a2a_request_duration_ms_{sum,count}` — end-to-end latency.

For a real deployment, swap the in-process collector for OpenTelemetry via the
same `onEvent` seam: `createOtelObserver({ meter, tracer })` (from
`portauth-a2a-receiver`) records the counters/histograms above as OTel
instruments (`portauth.a2a.*`) and emits decision spans, without adding an OTel
dependency to the package — you inject your application's meter/tracer. Continue
the trace across presenter → receiver → engine using the propagated
`X-Request-Id`/`traceparent`.

### Logging

Use `createStructuredLogger()` (the CLI default) for JSON logs with secret
redaction. It drops `authorization`, `vc`, `credential(s)`, `password`, and any
`*token`/`*secret` field. Raw VCs are never logged (SPEC §8.1). Standard fields:
`ts`, `level`, `msg`, `requestId`, `taskId`, `auditRecordId`.

### Health vs. readiness

- `GET /health` — liveness; the process is up.
- `GET /ready` — readiness; probes engine reachability and task-store
  connectivity. Returns 503 until both pass. Wire it to your orchestrator so
  traffic is not routed before dependencies are live.

## 4. Suggested SLOs and alerts

- **SLOs:** decision latency p99; engine availability; persistence success ≈ 100%.
- **Page on:** any `portauth_a2a_persistence_failures_total` increase; sustained
  engine error/timeout rate; readiness flapping; anomalous deny-rate
  (misconfiguration or attack); spike in receiver 401s.

## 5. Stuck-pending reconciliation

Persist-before-act guarantees the receiver never performs a side effect without
a recoverable record, but a crash (or a `persistence_failed` at the `final`
phase) after a real action can leave a record in `pending`. Because the
idempotency key is reserved, a retry returns `202` with the pending task and
does **not** re-execute — safe against double-acting, but the record needs
resolution. Operate a reconciliation sweep:

1. Query records where `lifecycle_state = 'pending'` and
   `updated_at < now() - <threshold>`.
2. For each, look up the engine audit row by `auditRecordId` (if present) or by
   the request's `idempotencyKey`/`requestId` to determine the true outcome.
3. Transition the record to `completed`/`failed` accordingly, and verify whether
   the downstream side effect actually occurred before retrying it.

Set an alert on the count of pending records older than the threshold.

## 6. Monitoring is not audit

Metrics, logs, and traces are operational and may be sampled or rotated. The
engine's signed audit log is the authoritative, tamper-evident compliance
record. Monitoring should **reference** `auditRecordId`, never replace the
audit. Do not relax audit retention because operational logs exist.
