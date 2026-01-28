# Project Prompt — Traefik Ingress Baseline Repo (AKS via GHES Actions)

You are operating on the repository that owns the **cluster ingress baseline** deployment for an AKS cluster using **Traefik**.

## Non-negotiable constraints
- Do NOT use Terraform Helm/Kubernetes providers for deploying charts/resources. Helm and kubectl operations occur in GitHub Actions.
- GHES is private/firewalled. Do NOT use GitHub OIDC federation. Authenticate to Azure using Service Principal + client secret:
  - `DEPLOY_CLIENT_ID` (SP appId)
  - `DEPLOY_SECRET` (SP secret)
- Secrets are sourced from GitHub **Environment secrets** (not repo-level secrets).
- Nexus-only images:
  - All Traefik images MUST be pulled from Nexus (override image registry/repository).
  - Create/update `IMAGE_PULL_SECRET_NAME` in namespace `traefik`.

## Deployment targets
- Namespace: `traefik`
- Release name: `traefik`
- Ingress class: `traefik` (non-default)
- Internal-only ingress: Traefik Service type `LoadBalancer` using Azure **internal** load balancer annotations.
- Optional DNS: if `DNS_ENABLED=true`, update Azure **Private DNS** `A` record (e.g., `ingress-traefik.logiki.co.uk`) to the ILB IP.
  - If the Private DNS zone is in a different subscription, the workflow must `az account set` to `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID` before record updates.
  - The Service Principal must have **Private DNS Zone Contributor** (or equivalent custom role) on the zone or its resource group in that subscription.

## Required workflow behaviours
- Idempotent: safe to rerun without manual cleanup.
- Helm operations MUST use `--wait --atomic --timeout <configured>` and pin chart versions.
- Must generate a temporary values file without leaking secrets to logs:
  - no `set -x`
  - `umask 077`
  - `chmod 600` on generated files
- Must include post-deploy checks:
  - controller rollout status
  - confirm ILB IP assigned
  - port-forward and validate dashboard/API responds (no public dashboard exposure)

## Output expectations
- Use explicit, readable multi-line YAML blocks.
- Avoid brittle assumptions; prefer parameterized inputs and documented defaults.
- Never print secrets to logs.

## GHES note
- Use Azure CLI login and runner-installed `kubectl`/`helm` (no Marketplace actions). `actions/checkout@v3` is allowed.
