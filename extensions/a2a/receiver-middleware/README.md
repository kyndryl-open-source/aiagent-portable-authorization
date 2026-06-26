# portauth-a2a-receiver

Framework-neutral A2A receiver guard for enforcing PortAuth before a task is
accepted.

## Async task store contract

`createPortAuthReceiver()` accepts a `taskStore` option. The default is
`createMemoryTaskStore()`, which is intended for development and CI only.
Production receivers should inject a durable implementation backed by their
own service datastore.

The store contract is async:

```js
{
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
  async clear() {}
}
```

`createPending(record)` must be atomic for `record.id` and
`record.idempotencyKey`. If a record already exists, it returns
`{ created: false, record: existingRecord }`. The receiver uses that behavior
to replay stored terminal responses for duplicate requests instead of
re-calling the PortAuth engine.

The atomic dedupe guarantee lives in `createPending()`. `getByIdempotencyKey()`
and `getByTaskId()` are provided for external lookups, diagnostics, task
lookup endpoints, and tests; the receiver does not rely on a separate
read-before-create path for dedupe. `update(id, patch)` assumes a single
writer per task record id after `createPending()` succeeds.

Task records use this shape:

```js
{
  id: "task-record-123",
  taskId: "task-123",
  idempotencyKey: "bill-2026-06-electric",
  lifecycleState: "pending", // pending | completed | denied | failed
  task: { /* A2A Task */ },
  response: null, // final { status, headers, body } after evaluation
  requestContext: { action: "payments:bill_pay", resource: "..." },
  presenterId: "agent:customer-finance-1",
  vcFormat: "tier1-jwt",
  auditRecordId: null,
  errorReason: null,
  createdAt: "2026-06-26T00:00:00.000Z",
  updatedAt: "2026-06-26T00:00:00.000Z"
}
```

The receiver does not persist raw credentials in task records.

## Lifecycle ordering

For each valid inbound A2A request, the receiver:

1. Computes the receiver idempotency key from PortAuth metadata
   `idempotencyKey`, then A2A `messageId`.
2. Persists a `pending` task record before calling the PortAuth engine.
3. Calls `/evaluate` or `/evaluate/batch`.
4. Persists the final `completed`, `denied`, or `failed` record, including the
   final A2A response and any engine `auditRecordId`.
5. Returns the persisted response.

The A2A `taskId` is not used as the idempotency key because one A2A task can
legitimately receive multiple continuation messages. Terminal `completed` and
`denied` responses are replayed for duplicate keys. Transient `failed` records
are not replayed; a retry re-enters PortAuth evaluation and overwrites the
stored failure if it succeeds.

If final persistence fails after engine evaluation, the receiver returns
`PORTAUTH_TASK_PERSISTENCE_FAILED` rather than claiming success. A production
receiver that performs real side effects should also run a reconciliation
sweeper or define a TTL/retry policy for orphaned `pending` records. Pending
rows can occur if the process exits during evaluation or if final persistence
fails after an external action. The reference receiver has no real side effect,
so it only documents this operational pattern.

## Postgres reference adapter

The package includes a Postgres reference adapter, but does not depend on
`pg`. Pass any `pg`-compatible pool or client with `query(sql, params)`:

```js
import pg from "pg";
import {
  createPortAuthReceiver,
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

This adapter is a working pattern for persist-before-act ordering. It is not a
required database choice; real receivers should use the datastore their service
already operates.

## Engine authentication

`createPortAuthReceiver({ engineAuthToken })` attaches
`Authorization: Bearer <token>` to every `/evaluate` and `/evaluate/batch`
call, and forwards an `X-Request-Id` for cross-service correlation. Set this
(or front the engine with mTLS) so the durable audit trail cannot be bypassed
by an unauthenticated caller.

## Monitoring (`onEvent`)

`createPortAuthReceiver({ onEvent })` invokes the observer with one structured
lifecycle event per request. The observer never affects request handling —
exceptions it throws are caught and logged.

```js
const receiver = createPortAuthReceiver({
  engineUrl: process.env.ENGINE_URL,
  engineAuthToken: process.env.A2A_ENGINE_AUTH_TOKEN,
  onEvent: (event) => metrics.record(event), // { type, requestId, decision, auditRecordId, engineDurationMs, ... }
});
```

Event types: `rejected`, `in_progress`, `replay`, `decision`,
`persistence_failed`. Every event carries `requestId`; `decision`/`replay`
carry `auditRecordId` (the join key to the engine audit row). See
[../OPERATIONS.md](../OPERATIONS.md) for the full field reference, metrics,
alerts, and the stuck-pending reconciliation procedure.
