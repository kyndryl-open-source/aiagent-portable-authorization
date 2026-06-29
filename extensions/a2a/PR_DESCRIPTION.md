## A2A reference implementation hardening

This hardens the reusable `extensions/a2a` reference implementation and narrows
the public documentation to technical interoperability claims.

### Changes

- Advertise receiver bearer endpoint authentication in the Agent Card when
  `A2A_RECEIVER_AUTH_MODE=bearer` is enabled, without exposing token values.
- Add an explicit A2A compatibility matrix that scopes support to the HTTP+JSON
  binding and avoids over-claiming JSON-RPC/gRPC coverage.
- Add storage-neutral stuck-pending reconciliation helpers plus a Postgres-backed
  reference receiver CLI.
- Gate the local nginx/self-signed TLS compose overlay in CI with HTTPS health,
  Agent Card discovery, and allow/deny presenter flows.
- Fix stale SDK quick-use docs and pin the docs wording through tests.

### Validation

- Full local TLS Docker stack:
  - `npm run tls:cert`
  - `docker-compose -f portauth-docker-compose.yml -f extensions/a2a/docker-compose.tls.yml up -d --build`
  - `https://localhost:3443/health`
  - `https://localhost:3443/.well-known/agent-card.json`
  - TLS presenter `allow` scenario returned `200`
  - TLS presenter `deny-amount` scenario returned expected `403`
- `npm test` from `extensions/a2a`: `91 pass / 0 fail`
- `git diff --check`
- `node --check reference-receiver/src/reconcile-pending.js`
- `npm ls --omit=dev pg -w reference-receiver`

### Notes

- No wire-format version change.
- The local Docker TLS stack was left running after the smoke so local state and
  volumes were not torn down implicitly.
