# Contributing to PortAuth

Thank you for your interest in contributing. PortAuth is a reference implementation that demonstrates the policy-embedded authorization architecture described in the arXiv paper *Digital Identity for Agentic Systems* ([arxiv.org/pdf/2605.11487](https://arxiv.org/pdf/2605.11487)). Contributions that improve correctness, paper alignment, test coverage, or operability are very welcome.

## Code of conduct

This project adopts the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to abide by its terms.

## Reporting bugs and requesting features

- Use GitHub Issues for non-security bug reports, feature requests, and design discussions.
- For security issues, follow [SECURITY.md](SECURITY.md) instead of opening a public issue.
- Include the component (`portauth-engine`, `portauth-issuer`, `state-authority-ref`, `sdks/preflight`, migrations, conformance vectors), the version or commit SHA, and a minimal reproduction.

## Development setup

```bash
# 1. Install Node.js 20+ and either Docker Compose or Podman + Podman Compose.
# 2. From the repo root, start the local stack:
podman compose -f portauth-docker-compose.yml up -d --build
# or
docker compose -f portauth-docker-compose.yml up -d --build

# 3. Run the engine unit tests:
cd portauth-engine && npm install && npm test

# 4. Run the issuer unit tests:
cd ../portauth-issuer && npm install && npm test

# 5. Run the machine-readable conformance vectors:
cd ../conformance && node run-conformance.js

# 6. Run the end-to-end smoke (services must already be running):
cd .. && \
  ISSUER_URL=http://localhost:3100 \
  ENGINE_URL=http://localhost:3200 \
  ISSUER_ID_FOR_ENGINE=http://portauth-issuer:3100 \
  ISSUER_JWKS_URI=http://portauth-issuer:3100/.well-known/jwks.json \
  ./portauth-engine/test/e2e-smoke.sh
```

`ISSUER_ID_FOR_ENGINE` and `ISSUER_JWKS_URI` need to point at the in-network hostname (`portauth-issuer`) because that is what appears in the `iss` claim of issued JWTs.

## Pull request checklist

Before opening a PR:

1. **Tests:** add or update tests in the appropriate place.
   - Engine logic: `portauth-engine/test/*.test.js` (Node.js test runner).
   - Issuer logic: `portauth-issuer/test/*.test.js`.
   - Cross-component behavior the paper specifies: a new conformance vector in `conformance/vectors/reference-core.json` and a row in the table at the top of that file.
   - End-to-end behavior visible to a smoke run: a step in `portauth-engine/test/e2e-smoke.sh`.
2. **Format and lint:** keep code clean; if you use [pre-commit](https://pre-commit.com/), wire up your own hooks (this repo does not ship a default config).
3. **Style:**
   - Avoid em-dashes and en-dashes in documentation and code comments. Use commas, parentheses, semicolons, or split sentences.
   - Keep new comments short and only for non-obvious context.
   - Do not add docstrings, comments, or type annotations to unchanged code.
4. **Paper alignment:** if your change affects credential, audit, manifest, or decision semantics, cite the paper section (e.g. "Section 7.5.1") in the PR description so reviewers can verify alignment.
5. **DCO:** sign your commits with `git commit -s`. By signing off, you certify that your contribution complies with the Developer's Certificate of Origin 1.1 reproduced in full in the [Developer's Certificate of Origin 1.1](#developers-certificate-of-origin-11) section at the bottom of this file. The canonical text is also at <https://developercertificate.org/>.
6. **No secrets:** scan your changes for credentials, private keys, and personal identifiers before pushing. We recommend running [gitleaks](https://github.com/gitleaks/gitleaks) or [detect-secrets](https://github.com/Yelp/detect-secrets) locally; this repo does not ship a pre-configured baseline.

## Branch and commit conventions

- Branch off `main`. Use prefixes:
  - `feature/<short-name>` for new features.
  - `fix/<short-name>` for bug fixes.
  - `chore/<short-name>` for tooling, docs, or refactors.
  - `security/<short-name>` for security work (consider opening a private security advisory first).
- Commit messages: imperative mood, short subject (max ~72 chars), wrapped body if needed. Reference issues with `Refs #NNN` or `Fixes #NNN`.

## Review

A maintainer will review within a few business days. CI must be green before merge. We squash-merge by default; preserve the issue reference in the squashed commit body.

## Releases

PortAuth uses [Semantic Versioning](https://semver.org/). Pre-1.0 releases may include breaking schema or API changes; these are flagged in [CHANGELOG.md](CHANGELOG.md). Database migrations are forward-only and live in `portauth-db/migrations/`.

## Out of scope for contributions

These are intentionally not in v1 and PRs adding them should first open a design issue so the scope can be discussed:

- A full HSM-backed audit-signing path.
- A full ASN.1 OCSP/CRL client (currently scaffolded in schema only; see [README.md](README.md#feature-status)).
- Cross-domain workflow policies that resolve remote workflow definitions.
- Selective-disclosure or ZK presentation flows.

Thanks again for helping move the reference implementation forward.

## Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
