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
