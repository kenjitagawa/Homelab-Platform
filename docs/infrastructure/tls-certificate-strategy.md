# TLS Certificate Strategy

## Strategy

This homelab uses public Let's Encrypt certificates issued through Traefik.

Traefik uses the ACME DNS-01 challenge with Cloudflare. This allows certificate issuance for internal services without exposing HTTP challenge endpoints to the public internet.

## Repository Configuration

The certificate resolver is configured in:

```text
apps/traefik/values.yaml
```

The resolver name is:

```text
letsencrypt
```

Traefik stores ACME state at:

```text
/data/acme.json
```

The Traefik Helm values persist that path with a `local-path` volume.

## Cloudflare Token

The Cloudflare API token is provided through a Kubernetes secret:

```text
cloudflare-secret
```

The expected key is:

```text
api-token
```

The real token must not be committed to Git.

Create the secret manually:

```bash
kubectl create secret generic cloudflare-secret \
  --namespace traefik-system \
  --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN'
```

## Ingress Configuration

Standard Kubernetes `Ingress` resources use Traefik's `websecure` entrypoint TLS configuration.

Traefik `IngressRoute` resources can request the resolver directly:

```yaml
tls:
  certResolver: letsencrypt
```

The Traefik dashboard uses this pattern in:

```text
apps/traefik/dashboard-ingressroute.yaml
```

## Validation

Verify HTTPS for an exposed service:

```bash
curl -I https://it-tools.tagawa.ca
```

Check the certificate issuer and validity:

```bash
openssl s_client -connect it-tools.tagawa.ca:443 -servername it-tools.tagawa.ca </dev/null
```

Renewal is handled by Traefik through the persisted ACME storage.
