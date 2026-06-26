# PortAuth A2A Reference Quickstart

This quickstart runs a real A2A HTTP+JSON presenter and receiver. The receiver
does not simulate authorization: it extracts PortAuth metadata from
`message.metadata`, calls the PortAuth engine, and returns an A2A task or A2A
error response.

## 1. Start PortAuth issuer and engine

From the repository root:

```bash
podman compose -f portauth-docker-compose.yml up -d --build
```

Docker Compose works as well:

```bash
docker compose -f portauth-docker-compose.yml up -d --build
```

## 2. Start the A2A receiver

```bash
cd extensions/a2a
npm install
ENGINE_URL=http://localhost:3200 npm run receiver
```

The receiver exposes:

- `GET /.well-known/agent-card.json`
- `POST /message:send`
- `GET /tasks` only when `A2A_ENABLE_TASK_LOOKUP=true`
- `GET /tasks/{id}` only when `A2A_ENABLE_TASK_LOOKUP=true`

If task lookup is enabled, set `A2A_TASK_READ_TOKEN` and call task endpoints
with `Authorization: Bearer <token>`.

The default task store is in-memory and intended for local development and CI.
It records the same async lifecycle used by durable stores: `pending` before
PortAuth evaluation, then `completed`, `denied`, or `failed` with the final
response and any `auditRecordId`. See
[`receiver-middleware/README.md`](receiver-middleware/README.md) for the
storage contract, idempotency semantics, pending-record recovery guidance, and
Postgres reference adapter.

For endpoint authentication on `POST /message:send`, start the receiver with a
bearer token. The receiver fails closed on placeholder secrets, so the fake
`dev-…-change-me` token below requires `A2A_DEV_INSECURE=true` (local demos
only — use a real token from a secret manager in production):

```bash
A2A_RECEIVER_AUTH_MODE=bearer \
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me \
A2A_DEV_INSECURE=true \
ENGINE_URL=http://localhost:3200 \
npm run receiver
```

`GET /health` and `GET /.well-known/agent-card.json` stay public for health
checks and Agent Card discovery.

## 3. Send an authorized task

In another terminal:

```bash
cd extensions/a2a
npm install
npm run allow
```

If the receiver was started with bearer auth, provide the same token:

```bash
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me npm run allow
```

Expected result: the presenter discovers the receiver Agent Card, issues a
PortAuth credential, sends an A2A `POST /message:send`, and receives a completed
A2A task with an `auditRecordId`.

## 4. Send a denied task

```bash
cd extensions/a2a
npm run deny
```

Expected result: the receiver calls the PortAuth engine, the engine denies the
over-limit payment, and the receiver maps that decision into an A2A error
payload.

## 5. Run conformance checks

```bash
cd extensions/a2a
npm install
npm test
```

The tests cover metadata validation, Agent Card extension declaration, header
activation, missing metadata, allow/deny mapping, and delegation-chain routing.

## 6. Optional local TLS

To run the reference receiver behind nginx with a generated localhost
self-signed certificate:

```bash
cd extensions/a2a
npm run tls:cert
npm run tls:up
```

See [TLS.md](TLS.md) for certificate trust, HTTPS presenter commands, and the
production TLS pattern.
