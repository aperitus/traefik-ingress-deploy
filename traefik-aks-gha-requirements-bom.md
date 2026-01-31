# Traefik Ingress (AKS) — Requirements BOM (Repo deliverable)

This repo deploys **Traefik** to an **Entra-enabled AKS** cluster using **GitHub Actions on GHES** (self-hosted runner), with **Nexus-only images**, and supports two routing integration modes:

- **Gateway API mode (preferred)** — apps publish `HTTPRoute` objects; attachment is controlled at the Gateway listener via namespace policy.
- **Ingress mode (compatibility)** — apps publish Kubernetes `Ingress` objects (e.g., charts like Dependency-Track).
- **Both** — transitional mode while migrating.

## Non-negotiables

- Helm/kubectl operations run in GitHub Actions (no Terraform Helm/Kubernetes providers).
- **Azure auth to ARM** uses Service Principal + secret (no OIDC federation due to GHES firewalls).
- **AKS API auth must be non-interactive**:
  - After `az aks get-credentials`, the workflow **must** run `kubelogin convert-kubeconfig --login spn` (unless using `--admin` kubeconfig).
  - Use an isolated kubeconfig (e.g. `KUBECONFIG=/tmp/kubeconfig`) to avoid runner-global state.
  - If you ever see a **device-code** prompt, treat it as a failure: some step is still using interactive `azurecli` exec auth.
- **Nexus-only**: Traefik images must be pulled from Nexus. Use `TRAEFIK_IMAGE_REPOSITORY` to point to Nexus (e.g. `library/traefik`).  
  *Chart source remains `traefik/traefik`* (from `TRAEFIK_HELM_REPO_URL`), but **image repo** must be overridden to Nexus.
- Idempotent and safe to re-run:
  - `helm upgrade --install --wait --atomic --timeout ...` with **pinned chart version**.
  - Namespace and secrets created with `kubectl apply` patterns.
- Never print secret material to logs. Generated values are written under `/tmp` with restrictive permissions.

## Targets

- Namespace: `traefik`
- Release name: `traefik`
- Service: **Internal LoadBalancer** (Azure ILB)
  - Optional static IP pin (via `vars.TRAEFIK_LB_IPV4`)
  - Optional Azure Private DNS A record update (cross-subscription supported)

## Inputs / Variables / Secrets

### Workflow inputs (dispatch)

- `environment`: `dev|preprod|prod`
- `routing_mode`: `gateway|ingress|both` (default: `both`)
- `debug_values`: upload rendered values file as artifact (safe; implemented as a separate job so it still appears even when the deploy job fails)
- `enable_traefik_dashboard`: dev-only enable of Traefik API/dashboard (validated internally)

### Repository/Environment variables (vars)

- `AKS_RESOURCE_GROUP`
- `AKS_CLUSTER_NAME`
- `TRAEFIK_CHART_VERSION` (pinned)
- `HELM_TIMEOUT` (must include unit, e.g. `20m`; workflow normalises digits-only to minutes)
- `TRAEFIK_HELM_REPO_URL` (optional; typically Nexus Helm proxy/mirror of upstream)
- `TRAEFIK_IMAGE_REPOSITORY` (required; **Nexus** image repo path, e.g. `library/traefik`)
- `TRAEFIK_IMAGE_TAG` (optional)
- `TRAEFIK_LB_IPV4` (optional private IP to pin)
- **Gateway access policy** (used for `routing_mode=gateway|both`):
  - `GATEWAY_ALLOWED_ROUTES_FROM` = `Same|All|Selector` (recommended `Selector`)
  - `GATEWAY_ALLOWED_ROUTES_LABEL_KEY` (default `traefik-gateway-access`)
  - `GATEWAY_ALLOWED_ROUTES_LABEL_VALUE` (default `enabled`)
    - **Must be a string** (avoid values like `true`, `false`, `yes`, `no`, `on`, `off`, `null`, `0`, `1`)

### Secrets (Environment secrets)

- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`
- `DEPLOY_CLIENT_ID`
- `DEPLOY_SECRET`
- `REGISTRY_SERVER`
- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`
- `IMAGE_PULL_SECRET_NAME`
- DNS (optional):
  - `DNS_ENABLED` (`true|false`)
  - `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID`
  - `PRIVATE_DNS_ZONE_RESOURCE_GROUP`
  - `PRIVATE_DNS_ZONE_NAME`
  - `PRIVATE_DNS_A_RECORD_NAME`

## Permissions (Azure)

1) **AKS internal LoadBalancer**  
   The AKS-managed identity must be able to read the subnet/VNet and manage load balancers. In practice, `Network Contributor` on the node subnet or VNet is commonly required (you observed `subnets/read` 403 until granting).

2) **Private DNS update (cross-subscription)**  
   The deploy SP (`DEPLOY_CLIENT_ID`) needs `Private DNS Zone Contributor` (or Contributor) on the Private DNS zone (or its RG) in the target subscription.

## Post-deploy checks

Required:
- Verify Traefik deployment rollout and that the ILB has an IP.

Optional (dev-only):
- When `enable_traefik_dashboard=true`, the workflow:
  1) Enables `api.dashboard=true` and `api.insecure=true` (internal admin port only).
  2) Creates an internal-only `traefik-dashboard` **ClusterIP** Service on port `8080`.
  3) Validates `/api/version` by running an **in-cluster Job** (using a Nexus-hosted test image containing `curl` or `wget`) that calls `http://traefik-dashboard:8080/api/version`.
     - This avoids kube-apiserver service proxy behaviour differences and keeps the check fully in-cluster.

## Deliverables (repo contents)

- Workflow: `.github/workflows/deploy-traefik.yaml`
- Values template: `helm/traefik/values.template.yaml`
- Docs:
  - `README.md`
  - `docs/onboarding-ingress.md`
  - `docs/onboarding-gateway-api.md`
- Project guidance files (for this chat / attachments):
  - `traefik-aks-gha-requirements-bom.md`
  - `traefik-project-prompt.md`
