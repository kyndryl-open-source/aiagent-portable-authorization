# PortAuth A2A Extension v1

Status: Draft spec with production-track reference implementation. This is
reusable reference code, not a production-grade deployment. The SDK and receiver
guard are workspace packages that can be installed from this repository today
and moved to published npm packages once v1 is frozen.

The receiver ships production seams for secrets, authentication, durable
persistence, and monitoring; see [OPERATIONS.md](OPERATIONS.md) for the full
runbook. Remaining items a deployment still owns:

- Supply real secrets from a secret manager. The receiver fails closed on
  missing/placeholder secrets unless `A2A_DEV_INSECURE=true`.
- Inject durable task/audit persistence (e.g. the bundled Postgres adapter)
  instead of the default in-memory development task store.
- Terminate TLS in front of every service (see [TLS.md](TLS.md)) and enforce
  `A2A_ENGINE_AUTH_TOKEN` on the engine.
- Wire `/ready`, `/metrics`, and the `onEvent` stream to your orchestrator and
  telemetry backend, and operate a stuck-pending reconciliation sweep.
- Publish the workspace packages to npm once v1 is frozen.
- Run the compose-backed A2A smoke and CI tests for every change.
- Scope interoperability claims to the supported A2A HTTP+JSON binding; see
  [COMPATIBILITY.md](COMPATIBILITY.md).

### Receiver environment variables

| Variable | Default | Purpose |
|---|---|---|
| `ENGINE_URL` | `http://localhost:3200` | PortAuth engine base URL |
| `A2A_RECEIVER_AUTH_MODE` | `none` | `none` or `bearer` for `POST /message:send` |
| `A2A_RECEIVER_BEARER_TOKEN` | — | Required when auth mode is `bearer` |
| `A2A_ENGINE_AUTH_TOKEN` | — | Bearer attached to receiver → engine calls; set the engine's `ENGINE_EVALUATE_AUTH_TOKEN` to the same value to enforce it |
| `A2A_ENABLE_TASK_LOOKUP` / `A2A_TASK_READ_TOKEN` | `false` / — | Enable + guard `/tasks` |
| `A2A_ENABLE_METRICS` / `A2A_METRICS_TOKEN` | `false` / — | Enable + guard `/metrics` |
| `A2A_CONTEXT_MAPPER` | `default` | CLI mapper selector: `default` (domain-neutral) or `bill-payment` (bundled example). Inject your own with the `contextMapper` option |
| `A2A_REQUIRE_POP` | `false` | Require proof-of-possession for every request; missing PoP → 401 `PORTAUTH_POP_REQUIRED` before the engine call |
| `A2A_POP_REQUIRED_ACTIONS` | — | Comma-separated `action` values that require PoP (when not requiring it globally) |
| `A2A_RECEIVER_NAME` / `A2A_RECEIVER_DESCRIPTION` | — | Override the Agent Card identity for your deployment |
| `A2A_DEV_INSECURE` | `false` | Allow startup with dev placeholders (local only) |

| File | Purpose |
|---|---|
| [SPEC.md](SPEC.md) | Normative specification of the extension. |
| [sdk/](sdk/) | `portauth-a2a` caller-side helpers for Agent Card discovery, metadata construction, and `sendWithVc()`. |
| [receiver-middleware/](receiver-middleware/) | `portauth-a2a-receiver` framework-neutral receiver guard that validates A2A requests and calls the PortAuth engine. |
| [reference-receiver/](reference-receiver/) | Real A2A HTTP+JSON receiver exposing `/.well-known/agent-card.json`, `/message:send`, and task lookup. |
| [reference-presenter/](reference-presenter/) | Real A2A presenter that discovers the receiver, obtains a credential, and sends authorized bill-payment tasks. |
| [reference-intermediate/](reference-intermediate/) | Producer side of N-hop delegation: authorizes the inbound hop, mints an attenuated credential, and forwards downstream. Runnable demo: `npm run two-hop`. |
| [schemas/](schemas/) | JSON Schemas for the Agent Card extension declaration and message metadata entries. |
| [schemas/openapi.json](schemas/openapi.json) | OpenAPI 3.1 contract for the receiver's HTTP surface (also served at `GET /openapi.json`). |
| [COMPATIBILITY.md](COMPATIBILITY.md) | A2A transport-binding support matrix and seller-safe interoperability wording. |
| [TLS.md](TLS.md) | Local nginx TLS reverse-proxy example with generated localhost certificate. |
| [OPERATIONS.md](OPERATIONS.md) | Production runbook: secrets, rotation, monitoring, alerts, reconciliation. |
| [VERSIONING.md](VERSIONING.md) | Wire-format freeze policy, package semver, compatibility matrix, and the publish checklist. |
| [tests/](tests/) | Conformance checks for metadata, Agent Card declaration, allow/deny mapping, and delegation routing. |
| [poc/run.js](poc/run.js) | Legacy single-process proof-of-concept retained as a compact smoke demo. |

## Packages

Two packages are meant to be consumed from npm; the runnable references stay
in-repo (`private`):

| Package | Role | Published |
|---|---|---|
| `portauth-a2a` | Presenter SDK — discovery, metadata, `sendWithVc()` | yes |
| `portauth-a2a-receiver` | Framework-neutral receiver guard | yes |
| `portauth-a2a-reference-receiver` | Runnable reference receiver | no (`private`) |
| `portauth-a2a-reference-presenter` | Runnable reference presenter | no (`private`) |
| `portauth-a2a-reference-intermediate` | Runnable intermediate (N-hop producer side) | no (`private`) |

To use the published packages in your own agent:

```bash
npm install portauth-a2a            # presenter side
npm install portauth-a2a-receiver   # receiver side
```

Version/compat policy and the wire-format freeze are documented in
[VERSIONING.md](VERSIONING.md). The wire format is frozen at the `…/a2a/v1`
extension URI independent of package version.

### Working in this repo

```bash
cd extensions/a2a
npm install
```

This wires all four workspace packages locally (the receiver's `^0.1.0`
dependency on the SDK resolves to the in-repo package).

## Quick start

```bash
# From the repository root:
extensions/a2a/quickstart-a2a.sh
```

Or run the pieces manually:

```bash
# 1. Bring up the standard compose stack (postgres + issuer + engine).
podman compose -f portauth-docker-compose.yml up -d --build

# 2. Start the real A2A receiver.
cd extensions/a2a
npm install
ENGINE_URL=http://localhost:3200 npm run receiver

# 3. In another terminal, run allow and deny scenarios.
npm run allow
npm run deny
```

Conformance checks:

```bash
cd extensions/a2a
npm test
```

Local TLS/nginx demo:

```bash
cd extensions/a2a
npm run tls:cert
npm run tls:up
```

See [TLS.md](TLS.md) for verification commands and production guidance.

## Receiver endpoint authentication

The reference receiver treats PortAuth credential authorization and HTTP
endpoint authentication as separate controls. PortAuth answers whether the
presented credential authorizes the requested action. Endpoint authentication
answers whether the caller may invoke the receiver endpoint at all.

For local demos, `A2A_RECEIVER_AUTH_MODE` defaults to `none`. To require a
bearer token for `POST /message:send`:

```bash
A2A_RECEIVER_AUTH_MODE=bearer \
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me \
A2A_DEV_INSECURE=true \
ENGINE_URL=http://localhost:3200 \
npm run receiver
```

The receiver fails closed on placeholder secrets, so the obviously-fake
`dev-…-change-me` token above only starts under `A2A_DEV_INSECURE=true` — local
demos only. In production, supply a real token from a secret manager and drop
that flag. Then run the presenter with the same token:

```bash
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me npm run allow
```

`GET /health` and `GET /.well-known/agent-card.json` remain public so probes
and Agent Card discovery keep working. Production teams can replace this
reference token check with an API gateway, ingress policy, OIDC/JWT validation,
service mesh policy, or another control that matches their environment.

When bearer mode is enabled, the Agent Card advertises endpoint authentication
through `securitySchemes.receiverBearer` and `securityRequirements`. It never
publishes the token value; it only tells discovery clients that `POST
/message:send` requires an `Authorization: Bearer <token>` header.

## Domain context mapping (bring your own)

The receiver is **domain-neutral**. It does not assume bill payments, banking,
or any other vertical. A *context mapper* translates an inbound A2A message into
the PortAuth `requestContext` the engine evaluates. The default mapper
(`defaultRequestContextMapper`) simply forwards the message's structured data
part, so any domain works out of the box as long as the presenter sends
`action`, `resource`, and whatever constraint fields the credential expects.

Inject your own mapper to apply domain defaults, coercion, or validation:

```js
import { createReferenceReceiverServer } from "portauth-a2a-reference-receiver";

function procurementContextMapper({ message }) {
  const data = (message.parts || []).find((p) => p?.data !== undefined)?.data || {};
  return {
    action: data.action || "procurement:purchase_order",
    resource: data.resource,
    requestId: data.requestId || message.taskId || message.messageId,
    amount: Number(data.amount),
    vendorId: data.vendorId,
  };
}

const { server } = createReferenceReceiverServer({ contextMapper: procurementContextMapper });
```

The mapper may be `async`. It must return an object with at least `action` and
`resource` (string); non-finite numeric fields are rejected before the engine is
called. A worked banking example lives in
[`reference-receiver/examples/bill-payment-mapper.js`](reference-receiver/examples/bill-payment-mapper.js)
— run it from the CLI with `A2A_CONTEXT_MAPPER=bill-payment`, or import
`billPaymentContextMapper` and pass it as `contextMapper`.

## OpenTelemetry

The receiver guard emits one structured lifecycle event per request through its
`onEvent` seam. `createOtelObserver` turns those events into OpenTelemetry
metrics (and optional spans) without this package depending on the OTel SDK — you
pass in your application's meter/tracer, the same way the Postgres store accepts
any `{ query }`:

```js
import { metrics, trace } from "@opentelemetry/api";
import { createPortAuthReceiver, createOtelObserver } from "portauth-a2a-receiver";

const onEvent = createOtelObserver({
  meter: metrics.getMeter("portauth-a2a"),
  tracer: trace.getTracer("portauth-a2a"), // optional
});

createPortAuthReceiver({ engineUrl, onEvent });
```

It records `portauth.a2a.decisions`, `.rejected`, `.replays`,
`.persistence_failures` counters and `.engine_call.duration` /
`.request.duration` histograms, attributed with the `requestId`/`auditRecordId`
correlation keys. The in-process `/metrics` collector remains for environments
without an OTel pipeline.

## Durable task and audit persistence

`portauth-a2a-receiver` is storage-neutral. It accepts an async `taskStore`
that persists task lifecycle records:

```js
const taskStore = {
  async createPending(record) {
    return { created: true, record };
  },
  async update(id, patch) {
    return updatedRecord;
  },
  async get(id) {
    return recordOrNull;
  },
  async getByIdempotencyKey(idempotencyKey) {
    return recordOrNull;
  },
  async getByTaskId(taskId) {
    return latestRecordOrNull;
  },
  async list({ limit } = {}) {
    return records;
  },
};
```

The receiver persists a `pending` record before calling the PortAuth engine,
then updates that same record to `completed`, `denied`, or `failed` with the
final A2A response and any `auditRecordId` before returning. Duplicate requests
with the same PortAuth metadata `idempotencyKey`, or the same A2A `messageId`
when no PortAuth idempotency key is supplied, replay terminal `completed` or
`denied` responses instead of calling the engine again.

The A2A `taskId` is deliberately not used as the receiver idempotency key
because a single A2A task can receive multiple continuation messages.
Transient `failed` records are not replayed; a retry with the same key attempts
PortAuth evaluation again.

`createMemoryTaskStore()` remains the default for dev and CI. A Postgres
reference adapter is available for adopters that already use `pg`:

```js
import pg from "pg";
import {
  createPostgresTaskStore,
  ensurePostgresTaskStoreSchema,
} from "portauth-a2a-receiver";

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });
await ensurePostgresTaskStoreSchema(pool);

const receiver = createPortAuthReceiver({
  engineUrl: process.env.ENGINE_URL,
  taskStore: createPostgresTaskStore({ queryable: pool }),
});
```

The Postgres adapter is a reference pattern, not a required storage layer.
Production receivers should use whatever durable store their service already
operates, such as Postgres, DynamoDB, Redis, Cosmos DB, or another database.
Task records intentionally do not persist raw credentials.

`update()` writes in a single atomic statement — it sets only the columns
present in the patch and never reads first, so concurrent updates to the same
record cannot lose each other's writes. The adapter is exercised by a live
Postgres integration test (`tests/postgres-task-store.test.js`) covering
idempotency-key dedup, the concurrent insert race, atomic partial updates, and
concurrent updates; it runs in CI against the job's postgres service and skips
locally unless `A2A_TEST_DATABASE_URL` (or `DATABASE_URL`) is set.

Schema changes are applied by a versioned migrator (`runTaskStoreMigrations`,
which `ensurePostgresTaskStoreSchema` delegates to): it records applied versions
in a `<table>_migrations` ledger and applies only the pending migrations, each
in its own transaction. Migrations (`TASK_STORE_MIGRATIONS`) are append-only —
add a new version, never edit a shipped one.

The reference flow:

1. Registers the issuer with the engine and trusts it for `payments:bill_pay`.
2. Issues a UC-03 Tier 1 credential at the issuer with three constraints
   (amount cap, recipient enum, geo enum).
3. The presenter discovers the receiver Agent Card and activates the extension
   through the `A2A-Extensions` header.
4. The presenter sends `POST /message:send` with the credential under
   `message.metadata["<extension-URI>/vc"]`.
5. The receiver extracts metadata, translates the A2A message to PortAuth
   request context, and calls `/evaluate` or `/evaluate/batch`.
6. The receiver returns an A2A task on allow or a google.rpc-style A2A error on
   deny.

## Roadmap

| Phase | Deliverable | Status |
|---|---|---|
| 0 | SPEC.md + PoC | done |
| 1 | `sdk/` caller SDK with Agent Card discovery and `sendWithVc()` | done |
| 2 | `sdk/` self-contained `preflight()` compatibility and manifest verification | done |
| 3 | `receiver-middleware/` receiver guard | done |
| 4 | `reference-presenter/` and `reference-receiver/` real HTTP+JSON agents | done |
| 5 | Conformance tests under `tests/` and CI gating | done |
| 6 | Publish-ready packages: metadata, semver dep ranges, `v1` wire freeze ([VERSIONING.md](VERSIONING.md)) | done |
| 7 | Optional Google ADK adapter and Python receiver example | planned |
