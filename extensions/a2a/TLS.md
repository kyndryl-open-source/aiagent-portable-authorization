# Local TLS for the A2A Reference Receiver

This guide shows TLS termination for the A2A reference receiver using nginx in
front of the Node service.

The reference topology is:

```text
A2A presenter/client
  -> https://localhost:3443
  -> nginx TLS reverse proxy
  -> http://a2a-reference-receiver:3300
  -> http://portauth-engine:3200
```

The Node receiver stays HTTP-only on the private compose network. The public
Agent Card URL is `https://localhost:3443`. The compose overlay also enables
the reference receiver's bearer-token endpoint authentication for
`POST /message:send` using the development token
`dev-a2a-receiver-token-change-me`. Because the receiver fails closed on
placeholder secrets, the overlay sets `A2A_DEV_INSECURE=true` to permit that
fake token — a local-demo escape hatch only; production supplies a real token
from a secret manager and omits the flag.

## Local self-signed certificate

Generate a local certificate for `localhost`:

```bash
cd extensions/a2a
npm run tls:cert
```

This creates ignored local files:

- `tls/localhost.crt`
- `tls/localhost.key`
- `tls/localhost-openssl.cnf`

These files are local development material only. Do not commit private keys and
do not use this self-signed certificate outside a local demo.

## Start the TLS stack

From `extensions/a2a`:

```bash
npm run tls:up
```

Equivalent repository-root command:

```bash
docker compose \
  -f portauth-docker-compose.yml \
  -f extensions/a2a/docker-compose.tls.yml \
  up -d --build
```

Podman users can run the same file set with `podman compose`.

## Verify TLS

Use the generated certificate as the trust anchor:

```bash
curl --cacert extensions/a2a/tls/localhost.crt https://localhost:3443/health
curl --cacert extensions/a2a/tls/localhost.crt https://localhost:3443/.well-known/agent-card.json
```

The Agent Card should advertise an HTTPS interface:

```json
{
  "supportedInterfaces": [
    {
      "url": "https://localhost:3443",
      "protocolBinding": "HTTP+JSON",
      "protocolVersion": "1.0"
    }
  ]
}
```

`GET /health` and `GET /.well-known/agent-card.json` are intentionally public.
The bearer token is required only for `POST /message:send`.

The repository CI runs this same TLS overlay as an A2A reference smoke: it
generates the localhost certificate, starts nginx with
`docker-compose.tls.yml`, verifies HTTPS health and Agent Card discovery, then
runs allow/deny presenter flows with `NODE_EXTRA_CA_CERTS` set to the generated
certificate.

## Run presenter scenarios over TLS

From `extensions/a2a`:

```bash
NODE_EXTRA_CA_CERTS="$PWD/tls/localhost.crt" \
A2A_RECEIVER_URL=https://localhost:3443 \
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me \
ISSUER_URL=http://localhost:3100 \
ENGINE_URL=http://localhost:3200 \
ISSUER_ID_FOR_ENGINE=http://portauth-issuer:3100 \
npm run allow
```

Denied path:

```bash
NODE_EXTRA_CA_CERTS="$PWD/tls/localhost.crt" \
A2A_RECEIVER_URL=https://localhost:3443 \
A2A_RECEIVER_BEARER_TOKEN=dev-a2a-receiver-token-change-me \
ISSUER_URL=http://localhost:3100 \
ENGINE_URL=http://localhost:3200 \
ISSUER_ID_FOR_ENGINE=http://portauth-issuer:3100 \
npm run deny
```

For quick local debugging only, `curl -k` or
`NODE_TLS_REJECT_UNAUTHORIZED=0` will bypass certificate validation. Do not use
those options in production examples, CI, or deployed environments.

## Stop the stack

```bash
cd extensions/a2a
npm run tls:down
```

## Production pattern

For production, keep the same boundary but replace the local certificate and
demo defaults:

- Terminate TLS at nginx, a cloud load balancer, an API gateway, Caddy, Envoy,
  or a Kubernetes ingress controller.
- Use a certificate signed by a trusted CA, an internal enterprise CA, or an
  automated certificate manager such as ACME/cert-manager.
- Store private keys in approved secret storage. Do not bake keys into images.
- Set `A2A_RECEIVER_URL=https://<public-hostname>` so the Agent Card advertises
  the real HTTPS endpoint.
- Keep the receiver on a private network; expose only the TLS terminator.
- Replace the demo bearer token with a value from approved secret storage, or
  replace the reference token check with an API gateway, ingress policy,
  OIDC/JWT validation, service mesh policy, or another endpoint-auth control
  where appropriate.
- Add request logging/redaction, rate limits, WAF/API gateway controls where
  appropriate, and operational monitoring.
- Use service-to-service authentication or mTLS for private hops where your
  environment requires it.

## Kubernetes note

The compose overlay maps directly to common Kubernetes objects:

- `a2a-reference-receiver`: Deployment + ClusterIP Service.
- `a2a-nginx`: Ingress controller, Gateway API, or sidecar/proxy Deployment.
- `tls/localhost.crt` and `tls/localhost.key`: Kubernetes TLS Secret for local
  experiments only; use cert-manager or enterprise secret management in real
  clusters.
- `A2A_RECEIVER_URL`: ConfigMap or environment variable set to the HTTPS
  ingress hostname.
