# PortAuth A2A — Versioning & Compatibility

This extension has **two independently versioned surfaces**. Keeping them
separate is what lets clients in any language interoperate while the JavaScript
packages keep evolving.

1. **The wire format** — the extension URI, the `A2A-Extensions` activation
   header, the `message.metadata` keys, the VC formats, the Agent Card extension
   object, and the engine-error → A2A-error mapping. This is the contract every
   implementation (JS, Python, or third-party) must agree on.
2. **The npm packages** — `portauth-a2a` (presenter SDK) and
   `portauth-a2a-receiver` (receiver guard). These follow semver for their
   JavaScript API.

## 1. Wire format — frozen at `v1`

The wire format is identified by the extension URI:

```
https://kyndryl-open-source.github.io/aiagent-portable-authorization/a2a/v1
```

The `/v1` suffix **is** the wire version. Everything that travels on the wire is
frozen against it:

| Frozen surface | Value |
|---|---|
| Extension URI | `…/a2a/v1` |
| Activation header | `A2A-Extensions: <extension URI>` |
| Media type | `application/a2a+json` |
| Metadata keys | `…/v1/{vc, vcFormat, presenterId, proofOfPossession, idempotencyKey, delegationChain}` |
| VC formats | `tier1-jwt`, `tier2-x509-jwt`, `tier3-vc-jose` |
| Agent Card extension | `{ uri: "…/a2a/v1", required, params }` |
| Error mapping | engine error → `(HTTP status, PORTAUTH_* reason)` in a `google.rpc.Status` body |

**Rules:**

- A **backward-compatible** addition (a new optional metadata key, a new engine
  error → reason mapping, a new optional Agent Card param) keeps the `v1` URI.
- A **breaking** change (renaming/removing a metadata key, changing a key's
  meaning, changing an established error mapping, changing the activation
  mechanism) requires a **new URI** (`…/a2a/v2`) and a parallel-support window —
  receivers advertise both URIs until presenters migrate.
- Either way, the change must be reflected in `tests/wire-contract.test.js` in
  the same commit. That snapshot test fails loudly on any unreviewed drift, so a
  wire change cannot ship by accident.

The JSON Schemas under [`schemas/`](schemas/) (`metadata.schema.json`,
`agent-card-extension.schema.json`) and the [`schemas/openapi.json`](schemas/openapi.json)
HTTP contract are the machine-readable form of this surface and carry the same
`v1` `$id`s.

## 2. npm packages — semver

| Package | Role | Public? |
|---|---|---|
| `portauth-a2a` | Presenter SDK (build/send requests, map decisions) | published |
| `portauth-a2a-receiver` | Framework-neutral receiver guard | published |
| `portauth-a2a-reference-receiver` | Runnable reference receiver | `private` (not published) |
| `portauth-a2a-reference-presenter` | Runnable reference presenter | `private` (not published) |

While the packages are `0.x`, the JavaScript API may change between minor
versions (per semver's `0.x` allowance), but the **wire format above stays
frozen at `v1`** regardless of the package version. Reaching `1.0.0` signals API
stability, not a wire change.

`portauth-a2a-receiver` depends on `portauth-a2a` by a semver range
(`^0.1.0`), so the two can be installed independently from the registry; inside
this repo the npm workspace resolves the range to the local package.

### Compatibility matrix

| `portauth-a2a` | `portauth-a2a-receiver` | Wire | Status |
|---|---|---|---|
| `0.1.x` | `0.1.x` | `v1` | current |

Extend this row-per-release as versions ship. Any two rows that share the same
**Wire** column interoperate over the network even if their package versions
differ.

## 3. Publishing checklist

The packages are publish-ready (license, repository, `files` allowlist, real
dependency ranges). They are **unscoped** names, so no npm org or
`publishConfig` is needed — npm publishes unscoped packages publicly by default.
To cut a release:

1. Bump the `version` in the package(s) being released; add a row to the
   compatibility matrix above.
2. `npm test` (from `extensions/a2a/`) — must be green, including
   `tests/wire-contract.test.js`.
   Run all publish commands **from `extensions/a2a/`** (that directory is the
   npm workspace root — there is no root-level `package.json`).
3. `npm pack --dry-run -w sdk` (and `-w receiver-middleware`) to confirm the
   tarball contains only `src/`, `README.md`, `LICENSE`, and `package.json`.
4. `npm publish -w sdk` then `npm publish -w receiver-middleware` (SDK first —
   the receiver depends on it). Requires only a logged-in npm account
   (`npm login`); this is the only step that is not reproducible in-repo.
