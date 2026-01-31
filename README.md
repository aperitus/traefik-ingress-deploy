# Traefik Ingress for AKS (GHES + Nexus)

This repository deploys **Traefik** to an **Entra-enabled AKS** cluster using **GitHub Actions on GHES** (self-hosted runner). It supports **two integration models** for onboarding applications:

1) **Gateway API (preferred security model)**  
   Apps publish `HTTPRoute` resources that attach to a shared `Gateway` managed by the platform. Access is controlled at the Gateway listener via namespace policy (recommended: **Selector + namespace label**).

2) **Kubernetes Ingress (compatibility model)**  
   Apps publish `Ingress` resources (useful for upstream Helm charts like Dependency-Track). Access control relies on RBAC + policy (e.g., enforcing `ingressClassName=traefik`, TLS, hostname constraints).

You can run **both** modes during migration.

## Key security/segregation properties

- **Ingress plane is owned by this repo only**: Traefik install/upgrade, Service type/annotations (Azure ILB), and (optional) Private DNS automation.
- **Applications do not modify the `traefik` namespace**. App repos create only namespaced routing resources (`Ingress` or `HTTPRoute`) in their own namespaces.
- **Gateway API mode** provides a stronger default tenancy model because attachment can be restricted to specific namespaces via the Gateway listener `namespacePolicy`.

## How to deploy

Run the GitHub Action: **Deploy Traefik Ingress (AKS)**.

### Required vars (per GH Environment)

- `AKS_RESOURCE_GROUP`
- `AKS_CLUSTER_NAME`
- `TRAEFIK_CHART_VERSION`
- `HELM_TIMEOUT` (must include unit, e.g. `20m`)
- `TRAEFIK_IMAGE_REPOSITORY (default: library/traefik)` (Nexus path)
- Optional: `TRAEFIK_IMAGE_TAG`, `TRAEFIK_LB_IPV4` (pin ILB IP), `TRAEFIK_HELM_REPO_URL`
> **Chart reference:** the workflow installs `traefik/traefik` from `TRAEFIK_HELM_REPO_URL` (typically your Nexus Helm proxy/mirror).


### Required secrets (per GH Environment)

- Azure: `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `DEPLOY_CLIENT_ID`, `DEPLOY_SECRET`
- Nexus registry auth: `REGISTRY_SERVER`, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`, `IMAGE_PULL_SECRET_NAME`
- Optional DNS: `DNS_ENABLED`, `PRIVATE_DNS_ZONE_*`
- TLS (enabled by default; set workflow input `enable_tls=false` to run without TLS):
  - `ELOKO_WILDCARD_CRT` (PEM certificate / full chain)
  - `ELOKO_WILDCARD_KEY` (PEM private key)
  - Back-compat fallback names are also accepted: `WILDCARD_CRT` / `WILDCARD_KEY`


### Runner prerequisites

This workflow assumes the **self-hosted GHES runner** has these tools available on PATH:

- `az` (Azure CLI)
- `kubectl`
- `helm`
- `kubelogin` (required for Entra-enabled AKS non-interactive auth)

If the run ever prompts for a **device code** / browser login, the kubeconfig has not been converted for SPN auth (or the Azure CLI context is stale). The workflow is expected to run fully non-interactively.


## Routing mode toggle (Ingress vs Gateway API vs Both)

The workflow dispatch input `routing_mode` controls which Traefik providers are enabled:

- `gateway`  
  Enables `providers.kubernetesGateway`, deploys a default `GatewayClass` and `Gateway`, disables `providers.kubernetesIngress`.

- `ingress`  
  Enables `providers.kubernetesIngress`, disables `providers.kubernetesGateway` and does not deploy Gateway objects.

- `both`  
  Enables both providers and deploys a default Gateway. Use this while migrating.

## Gateway API access control (recommended)

When `routing_mode` is `gateway` or `both`, the workflow configures a namespace policy for route attachment using vars:

- `GATEWAY_ALLOWED_ROUTES_FROM` = `Same|All|Selector` (recommended: `Selector`)
- `GATEWAY_ALLOWED_ROUTES_LABEL_KEY` (default: `traefik-gateway-access`)
- `GATEWAY_ALLOWED_ROUTES_LABEL_VALUE (labels are strings; use a non-boolean value like "enabled")` (default: `enabled`)

**Onboarding a namespace (Gateway API)**: label the application namespace, then apply an `HTTPRoute` that references the shared Gateway.

See: `docs/onboarding-gateway-api.md`

## Ingress access control (compatibility)

Ingress mode is less opinionated; you should enforce:
- RBAC: only approved repos/teams can create `Ingress` resources in their namespaces.
- Policy: require `spec.ingressClassName: traefik`, TLS enabled, and restrict hostnames to approved zones.

See: `docs/onboarding-ingress.md`

## TLS enablement (wildcard certificate)

This repo can configure TLS on the Traefik data-plane using a wildcard certificate stored in GitHub Environment secrets. TLS is **enabled by default** (`enable_tls=true`).

### What the workflow does when TLS is enabled

When you run the workflow with `enable_tls=true` (default):

- it creates/updates a Kubernetes TLS Secret in the `traefik` namespace (default name: `wildcard-tls`)
- it adds an **HTTPS** listener (`websecure`) to the shared `Gateway` (Gateway API mode) that references that Secret
- it enforces **Option B**: keep HTTP open but redirect **HTTP → HTTPS** at the Traefik entrypoint (`web` → `websecure`)

### Configure inputs/vars

1) Store your wildcard material as Environment secrets (recommended names):
   - `ELOKO_WILDCARD_CRT` (PEM certificate; include full chain)
   - `ELOKO_WILDCARD_KEY` (PEM private key)

2) Optional Environment vars:
   - `TLS_SECRET_NAME` (default: `wildcard-tls`)
   - `GATEWAY_LISTENER_WEBSECURE_PORT` (default: `8443`)

### Validating the wildcard certificate secrets

The error `tls: failed to find any PEM data in certificate input` almost always means your secret is **not raw PEM**.

Requirements:
- `ELOKO_WILDCARD_CRT` must contain one or more PEM certificate blocks (starts with `-----BEGIN CERTIFICATE-----`).
- `ELOKO_WILDCARD_KEY` must contain an unencrypted PEM private key (starts with `-----BEGIN ... PRIVATE KEY-----`).

Local (safe) checks (these do **not** print the private key):

```bash
# Certificate parses and shows minimal metadata
openssl x509 -in wildcard.crt -noout -subject -issuer -dates -fingerprint -sha256

# Key parses non-interactively (fails fast if encrypted / wrong format)
openssl pkey -in wildcard.key -noout -passin pass:

# Confirm cert+key match (RSA/ECDSA): compare public-key hashes
cert_hash=$(openssl x509 -in wildcard.crt -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 | awk '{print $2}')
key_hash=$(openssl pkey -in wildcard.key -pubout -outform DER | openssl dgst -sha256 | awk '{print $2}')
test "$cert_hash" = "$key_hash" && echo "MATCH" || echo "MISMATCH"

# Count certificate blocks (should be >= 1; full chain often has 2+)
grep -c 'BEGIN CERTIFICATE' wildcard.crt
```

If you have a PFX/PKCS#12 instead of PEM, convert it first:

```bash
openssl pkcs12 -in wildcard.pfx -clcerts -nokeys -out wildcard.crt
openssl pkcs12 -in wildcard.pfx -nocerts -nodes -out wildcard.key
```

The workflow also performs these validations (PEM marker checks + `openssl` parse + keypair match) before creating the Kubernetes TLS Secret.

### Debugging certificate handling in the workflow

If you suspect the runner-written temp files are malformed (line endings, literal `\n`, truncation), run the workflow with:

- `debug_tls_artifacts=true`

This triggers a separate job that uploads:
- `wildcard.crt` (certificate / full chain)
- `wildcard.public.pem` (public key derived from the private key)
- `tls-diagnostics.txt` / `tls-match.txt`

The private key is **never** uploaded.

### Notes by routing model

- **Gateway API**: TLS is configured once on the shared Gateway listener; application `HTTPRoute` resources do not need to carry per-app certificates.
- **Kubernetes Ingress**: the referenced TLS Secret must exist **in the same namespace as the Ingress**. The Secret created by this repo in `traefik` is not automatically available in application namespaces.
  - If you create HTTP-only Ingress resources (no `spec.tls`), the platform-wide HTTP→HTTPS redirect will send clients to HTTPS where no router exists, typically resulting in `404` or a TLS mismatch. Treat TLS on Ingress as mandatory when `enable_tls=true`.

See also:
- `docs/onboarding-gateway-api.md` (Gateway API)
- `docs/onboarding-ingress.md` (Ingress)

## Dashboard testing

By default, the dashboard/API is not exposed on the data-plane. For dev-only validation:
- set workflow input `enable_traefik_dashboard=true` (the workflow adds Traefik CLI flags `--api=true --api.dashboard=true --api.insecure=true` on the internal admin entrypoint)
- the workflow creates an internal-only `traefik-dashboard` ClusterIP service and validates by running an **in-cluster job** that calls `http://traefik-dashboard:8080/api/version`

Manual (operator) check:
- `kubectl -n traefik port-forward svc/traefik-dashboard 8080:8080` then browse `http://127.0.0.1:8080/dashboard/`

## Azure permissions notes

- **Internal LB provisioning**: AKS managed identity must be able to read subnet/VNet and manage load balancers (you observed `subnets/read` 403 until granting).
- **Private DNS updates (cross-subscription)**: the deploy SP needs `Private DNS Zone Contributor` (or Contributor) on the target zone or RG.

## Next steps

- Onboard apps using **Gateway API** where possible.
- Keep `both` during migration; move to `gateway` once legacy charts using `Ingress` are replaced or wrapped with `HTTPRoute`.
