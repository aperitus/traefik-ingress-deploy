# Project Prompt — Traefik Ingress for AKS via GitHub Actions (GHES)

You are assisting on a project that deploys **Traefik** to **AKS** using **GitHub Actions on GitHub Enterprise Server (GHES)**.

## Output contract (non-negotiable)

When asked to “regenerate”, “update”, “apply fixes”, or “produce deliverables” for this repo:
- **Always output exactly one ZIP** containing the full repo (workflow + values + docs) for the change.
- The ZIP must include a **version number** in its filename (e.g. `v10`) so the user can track iterations.
- Do not scatter deliverables across multiple zips unless explicitly asked.

## Constraints (must follow)

- Do NOT use Terraform Helm/Kubernetes providers for deploying charts/resources.
- GHES is private/firewalled. Do NOT use OIDC federation. Authenticate to Azure using Service Principal + client secret:
  - `DEPLOY_CLIENT_ID` (appId)
  - `DEPLOY_SECRET` (secret)
- Nexus-only for runtime images:
  - The Helm chart reference remains `traefik/traefik` (from `TRAEFIK_HELM_REPO_URL`).
  - **All images must be pulled from Nexus** by overriding `image.registry` and `image.repository` to the Nexus path (default `library/traefik`).
- Workflow must be idempotent and safe to re-run (`helm upgrade --install --wait --atomic --timeout ...`) and **pin chart versions**.
- Never print secret material to logs.
- Use repo/environment **vars** for non-secret defaults (AKS resource group, cluster name, chart version, helm timeout, DNS zone details).
- Keep secrets in **GitHub Environment secrets**.

## Entra-enabled AKS (non-interactive requirement)

- Azure CLI login (ARM) must be **non-interactive** using SP credentials.
- The AKS API auth must also be **non-interactive**:
  - After `az aks get-credentials`, run `kubelogin convert-kubeconfig --login spn` using the same SP credentials.
  - Use isolated paths to avoid runner-global state:
    - `AZURE_CONFIG_DIR=/tmp/azcfg` (clear stale context)
    - `KUBECONFIG=/tmp/kubeconfig` (per-run kubeconfig file)
  - Any device-code / browser prompt is a failure and must be eliminated.

## Routing integration modes

The repo must support:
- **Gateway API mode (preferred security)**: `providers.kubernetesGateway.enabled=true` and a default `GatewayClass` + `Gateway` installed. Access controlled via Gateway listener namespace policy.
- **Ingress mode (compatibility)**: `providers.kubernetesIngress.enabled=true` to support Helm charts that emit Kubernetes Ingress.
- **Both** mode to migrate.

Routing mode is selected by workflow input `routing_mode`.

## Access control guidance

- Prefer Gateway mode with `GATEWAY_ALLOWED_ROUTES_FROM=Selector` and onboard namespaces by label.
- Gateway selector label values are **strings**; avoid boolean-like values (`true/false/yes/no/on/off/null`) or pure numbers.
- If Ingress mode is used, require explicit `ingressClassName=traefik` and enforce hostname/TLS rules via policy where possible.

## Post-deploy checks

- Must verify Traefik is ready and has an ILB IP assigned.
- If dashboard testing is enabled (`enable_traefik_dashboard=true`):
  - Enable `api.dashboard=true` and `api.insecure=true` (internal admin port only).
  - Create an internal-only `traefik-dashboard` ClusterIP service on port `8080`.
  - Validate `/api/version` via **kube-apiserver Service proxy** using `kubectl get --raw .../services/.../proxy/api/version`.

## DNS update

If `DNS_ENABLED=true`, update a Private DNS A record (possibly in another subscription). Document required Azure roles for:
- AKS managed identity (ILB provisioning)
- Deploy SP (Private DNS Zone Contributor)
