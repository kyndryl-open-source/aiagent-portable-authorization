# state-authority-ref

Reference implementation of the **State Authority** protocol introduced in
Plan 2 of `portauth_refimpl_features.md`.

It is a minimal, non-production Express service that:

1. Exposes a JWKS at `/.well-known/jwks.json`.
2. Returns a signed counter-value proof at `GET /proofs`.
3. Accepts increments at `POST /increment`.

Proof JWS payload (ES256, kid set to the SHA-256 prefix of the public JWK):

```json
{
  "iss": "<authority id>",
  "counterKey": "<counter id>",
  "metric": "amount" | "count",
  "windowStart": "<ISO-8601>",
  "value": <number>,
  "iat": <unix seconds>
}
```

## Run

```
cd state-authority-ref
npm install
STATE_AUTHORITY_PORT=3400 STATE_AUTHORITY_ID=state-authority-ref npm start
```

Then register it with the engine:

```sql
INSERT INTO portauth_state_authorities (authority_id, base_url, jwks_uri, active)
VALUES ('state-authority-ref',
        'http://localhost:3400',
        'http://localhost:3400/.well-known/jwks.json',
        TRUE);
```

## Limitations

* In-memory counters (lost on restart).
* No authentication on `/increment`.
* No multi-replica coordination.

These are intentional for the reference. A production authority would back
counters with a replicated transactional store, gate writes with mutual TLS
or signed requests, and implement clock-skew tolerance.
