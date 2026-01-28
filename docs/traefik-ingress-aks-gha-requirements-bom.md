# Traefik Ingress (AKS) — Requirements BOM (GitHub Actions / GHES)

**Scope**: Dedicated repository owns the **cluster ingress baseline** deployment using **Traefik** on AKS.
Deployment runs via **GitHub Actions (GHES)** on a **self-hosted runner**. Container images are pulled from **Nexus**.
Azure authentication uses **Service Principal + client secret** (no OIDC federation).

This repository is deployed **before** application repos (e.g., Dependency-Track). Application repos later create their own `Ingress`
resources with `ingressClassName: traefik` (or chart equivalent) to route behind this controller.

---

## 1) Decisions and fixed parameters
- Ingress controller: **Traefik (official Helm chart)**
- Namespace: `traefik`
- Release name: `traefik`
- Ingress class name: `traefik` (not default class)
- Service exposure: **Internal-only** `Service` type `LoadBalancer` (Azure ILB)
- Images: overridden to **Nexus** (`image.registry` and/or `image.repository`)
- Image pull auth: docker-registry Secret `IMAGE_PULL_SECRET_NAME` in `traefik`
- Post-deploy validation: `kubectl` readiness + dashboard/API reachability via **port-forward** (no public exposure)
- DNS (optional): Update **Azure Private DNS** `A` record to the internal LB IP (enables stable internal name even if IP changes)

---

## 2) Runner prerequisites
Self-hosted runner (Linux recommended) with:
- `az` (Azure CLI)
- `kubectl`
- `helm`
- `python3`

Network reachability from runner to:
- Azure ARM / Entra endpoints
- AKS API endpoint
- Nexus (Docker registry) and (optionally) Nexus Helm proxy

---

## 3) Workflow inputs (non-secret)
| Input | Required | Description |
|---|---:|---|
| `environment` | Yes | GitHub Environment name (e.g., `dev`) |
| `aks_resource_group` | Yes | AKS resource group name |
| `aks_cluster_name` | Yes | AKS cluster name |
| `use_admin_credentials` | No | Uses `az aks get-credentials --admin` |
| `traefik_chart_version` | Yes | Pinned Traefik chart version |
| `helm_timeout` | Yes | Helm timeout (e.g., `10m`) |

---

## 4) GitHub Environment secrets

### Azure (Service Principal + secret)
| Secret | Required | Description |
|---|---:|---|
| `AZURE_TENANT_ID` | Yes | Entra tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Yes | Target subscription |
| `DEPLOY_CLIENT_ID` | Yes | Service Principal appId |
| `DEPLOY_SECRET` | Yes | Service Principal client secret |

### Nexus registry (Docker)
| Secret | Required | Description |
|---|---:|---|
| `REGISTRY_SERVER` | Yes | Registry host[:port], no scheme |
| `REGISTRY_USERNAME` | Yes | Username / token |
| `REGISTRY_PASSWORD` | Yes | Password / token |
| `IMAGE_PULL_SECRET_NAME` | Yes | Secret name created in namespace `traefik` |

### Optional: Private DNS update (Azure Private DNS)
| Secret | Required | Description |
|---|---:|---|
| `DNS_ENABLED` | No | Set to `true` to update private DNS records |
| `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID` | No | Subscription ID that contains the private DNS zone (required when DNS_ENABLED=true and zone is in a different subscription) |
| `PRIVATE_DNS_ZONE_RESOURCE_GROUP` | No | RG containing the private DNS zone |
| `PRIVATE_DNS_ZONE_NAME` | No | Private DNS zone name (e.g., `logiki.co.uk`) |
| `PRIVATE_DNS_A_RECORD_NAME` | No | Record set name to update (default: `ingress-traefik`) |

---

## 4.1 Azure role requirements for Private DNS update
If `DNS_ENABLED=true` and you update a Private DNS record-set from this workflow, the deployment Service Principal must have RBAC in the **DNS zone subscription** (which may differ from the AKS subscription).

**Minimum recommended role** (built-in):
- **Private DNS Zone Contributor** on the scope of either:
  - the Private DNS Zone resource, or
  - the Resource Group containing the zone

**Example scope (zone-level)**:
`/subscriptions/<dns-subscription-id>/resourceGroups/<dns-rg>/providers/Microsoft.Network/privateDnsZones/<zone-name>`

**Required GitHub Environment secret**:
- `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID` (DNS subscription)



---

## 5) Required cluster actions
- Create namespace `traefik` if missing.
- Create/update docker-registry Secret `IMAGE_PULL_SECRET_NAME` in `traefik`.
- `helm upgrade --install` Traefik with:
  - `Service` type `LoadBalancer` and ILB annotations
  - IngressClass name `traefik` (non-default)
  - images overridden to Nexus
  - `imagePullSecrets` set
  - `--wait --atomic --timeout` and pinned chart version

---

## 6) Post-deployment checks
Required checks in workflow:
- `kubectl rollout status` for Traefik deployment.
- Wait for `Service` `.status.loadBalancer.ingress[0].ip`.
- Best-effort port-forward check to Traefik deployment (tries ports 9000 then 8080).
  - Any HTTP status other than `000` (connection failure) is considered a successful reachability check.

---

## 7) Deliverables (repo-ready)
- Workflow YAML: `.github/workflows/deploy-traefik.yaml`
- Minimal values template: `helm/traefik/values.template.yaml`
- This BOM: `docs/traefik-ingress-aks-gha-requirements-bom.md`
- Project prompt: `docs/traefik-ingress-project-prompt.md`

## Repository variables (vars)
Set these as GitHub **Repository Variables** (or Environment Variables if you scope by environment):
- `AKS_RESOURCE_GROUP`
- `AKS_CLUSTER_NAME`
- `TRAEFIK_CHART_VERSION`
- `HELM_TIMEOUT`
- Optional:
  - `TRAEFIK_HELM_REPO_URL`
  - `TRAEFIK_IMAGE_REPOSITORY`, `TRAEFIK_IMAGE_TAG`
  - `TRAEFIK_LB_IPV4`
  - DNS vars: `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID`, `PRIVATE_DNS_ZONE_RESOURCE_GROUP`, `PRIVATE_DNS_ZONE_NAME`, `PRIVATE_DNS_A_RECORD_NAME`

## GHES runner prerequisites
The self-hosted runner must have:
- Azure CLI (`az`)
- `kubectl`
- `helm`

This workflow intentionally avoids GitHub.com Marketplace actions (except `actions/checkout@v3`).
