# Variables and inputs

This repo is designed to be driven by **GitHub Environment vars/secrets** with optional overrides via `workflow_dispatch` inputs.

## Precedence rules

For **non-sensitive** values, the workflow resolves values in this order:

1. `workflow_dispatch` inputs (when provided)
2. GitHub Environment **vars** (`vars.*`)
3. GitHub Environment **secrets** (`secrets.*`) as a compatibility fallback

**Sensitive material** must be stored in **secrets** (e.g., `DEPLOY_SECRET`, `REGISTRY_PASSWORD`, wildcard TLS key).

---

## Workflow dispatch inputs

| Input | Type | Required | Default | Example | Notes |
|---|---|---:|---|---|---|
| `environment` | choice | Yes | — | `dev` | Selects the GitHub Environment to source `vars.*` / `secrets.*`. |
| `routing_mode` | choice | Yes | `both` | `gateway` | `gateway` enables Gateway API, `ingress` enables Ingress, `both` enables both during transition. |
| `enable_traefik_dashboard` | boolean | No | `false` | `true` | Enables Traefik API/Dashboard flags and runs the in-cluster `/api/version` check. Requires `TRAEFIK_TEST_IMAGE`. |
| `debug_values` | boolean | No | `false` | `true` | Uploads safe debug artifacts (rendered values + deploy debug bundle). |
| `enable_tls` | boolean | No | `true` | `true` | When `true`, creates/updates a TLS Secret in `traefik`, adds HTTPS Gateway listener (Gateway mode), and enforces HTTP→HTTPS redirect. Requires wildcard PEM secrets. |
| `aks_resource_group` | string | No | `""` | `rg-aks-dev-uks-01` | Overrides `AKS_RESOURCE_GROUP`. |
| `aks_cluster_name` | string | No | `""` | `aks-dev-01` | Overrides `AKS_CLUSTER_NAME`. |
| `use_admin_credentials` | boolean | No | `false` | `false` | Adds `--admin` to `az aks get-credentials` (skips `kubelogin convert-kubeconfig`). Use only when intended. |
| `traefik_chart_version` | string | No | `""` | `28.4.0` | Overrides `TRAEFIK_CHART_VERSION`. |
| `helm_timeout` | string | No | `""` | `20m` | Overrides `HELM_TIMEOUT`. If numeric (e.g. `10`), treated as minutes (`10m`). |

---

## Environment variables and secrets

### AKS target (required)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `AKS_RESOURCE_GROUP` | var | Yes | `rg-aks-dev-uks-01` | Resource group containing the AKS cluster. Can be overridden by input `aks_resource_group`. |
| `AKS_CLUSTER_NAME` | var | Yes | `aks-dev-01` | AKS cluster name. Can be overridden by input `aks_cluster_name`. |

### Azure authentication (required)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `AZURE_TENANT_ID` | var | Yes | `11111111-1111-1111-1111-111111111111` | Entra tenant ID. |
| `AZURE_SUBSCRIPTION_ID` | var | Yes | `22222222-2222-2222-2222-222222222222` | Subscription used for `az aks get-credentials`. |
| `DEPLOY_CLIENT_ID` | var | Yes | `33333333-3333-3333-3333-333333333333` | Client ID for the deployment service principal (SPN). |
| `DEPLOY_SECRET` | secret | Yes | `***` | Client secret for the deployment SPN. Must remain a secret. |

### Helm and Traefik chart (required)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `TRAEFIK_CHART_VERSION` | var | Yes | `28.4.0` | Version of `traefik/traefik` chart to install. Can be overridden by input `traefik_chart_version`. |
| `HELM_TIMEOUT` | var | Yes | `20m` | Helm timeout (e.g., `600s`, `20m`, `1h`). Can be overridden by input `helm_timeout`. |
| `TRAEFIK_HELM_REPO_URL` | var | No | `https://nexus.example.local/repository/helm-traefik/` | Helm repo URL (defaults to Traefik upstream chart repo if unset). Typically point this at your Nexus proxy/mirror. |

### Nexus image pull (required)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `REGISTRY_SERVER` | var | Yes | `nexus.example.local` | Container registry hostname used for `imagePullSecret` creation. |
| `REGISTRY_USERNAME` | var | Yes | `svc-ghes-runner` | Username for registry auth. |
| `REGISTRY_PASSWORD` | secret | Yes | `***` | Password/token for registry auth. Must remain a secret. |
| `IMAGE_PULL_SECRET_NAME` | var | Yes | `nexus-pull` | Name of the Kubernetes `docker-registry` secret created in `traefik`. |
| `TRAEFIK_IMAGE_REGISTRY` | var | No | `nexus.example.local` | Defaults to `REGISTRY_SERVER` when unset. |
| `TRAEFIK_IMAGE_REPOSITORY` | var | No | `library/traefik` | Nexus repository path for Traefik image. Defaults to `library/traefik`. |
| `TRAEFIK_IMAGE_TAG` | var | No | `v2.11.8` | Optional override: pins `image.tag` via Helm `--set image.tag=...`. |

### Load balancer IP pinning (optional)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `TRAEFIK_LB_IPV4` | var | No | `10.100.20.150` | Optional override: pins ILB IP via Helm `--set service.spec.loadBalancerIP=...`. The IP must be free in the target subnet. |

### Gateway API access control (recommended)

These control how the shared Gateway restricts which namespaces may attach routes.

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `GATEWAY_ALLOWED_ROUTES_FROM` | var | No | `Selector` | `Same`, `All`, or `Selector` (recommended). Only applies in `gateway` / `both` modes. |
| `GATEWAY_ALLOWED_ROUTES_LABEL_KEY` | var | No | `traefik-gateway-access` | Namespace label key used when `from=Selector`. |
| `GATEWAY_ALLOWED_ROUTES_LABEL_VALUE` | var | No | `enabled` | Namespace label value. Must be a **string** (avoid boolean-like values such as `true`). |
| `GATEWAY_LISTENER_WEB_PORT` | var | No | `8000` | HTTP listener port for the Gateway. Must align with Traefik chart `web` entrypoint port. |

### TLS (enabled by default)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `TLS_SECRET_NAME` | var | No | `wildcard-tls` | Name of the TLS Secret created/updated in namespace `traefik`. |
| `GATEWAY_LISTENER_WEBSECURE_PORT` | var | No | `8443` | HTTPS listener port on the Gateway (Gateway mode). |
| `ELOKO_WILDCARD_CRT` | secret | Yes (if TLS enabled) | `-----BEGIN CERTIFICATE-----...` | PEM certificate / chain. Must be raw PEM text (not base64, not PFX). |
| `ELOKO_WILDCARD_KEY` | secret | Yes (if TLS enabled) | `-----BEGIN PRIVATE KEY-----...` | PEM private key. Unencrypted. |
| `WILDCARD_CRT` | secret | Optional | `-----BEGIN CERTIFICATE-----...` | Back-compat name for the cert. Used if `ELOKO_WILDCARD_CRT` is not set. |
| `WILDCARD_KEY` | secret | Optional | `-----BEGIN PRIVATE KEY-----...` | Back-compat name for the key. Used if `ELOKO_WILDCARD_KEY` is not set. |

### Optional Private DNS automation

If you want the workflow to create/update a Private DNS **A record** pointing at the Traefik ILB IP, enable DNS.

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `DNS_ENABLED` | var | No | `true` | When `true`, the workflow writes/updates the A record via Azure CLI. |
| `PRIVATE_DNS_ZONE_SUBSCRIPTION_ID` | var | Yes (if DNS enabled) | `44444444-4444-4444-4444-444444444444` | Subscription hosting the Private DNS zone (may differ from `AZURE_SUBSCRIPTION_ID`). |
| `PRIVATE_DNS_ZONE_RESOURCE_GROUP` | var | Yes (if DNS enabled) | `rg-dns-shared-uks-01` | Resource group hosting the Private DNS zone. |
| `PRIVATE_DNS_ZONE_NAME` | var | Yes (if DNS enabled) | `privatelink.internal.example.local` | DNS zone name. |
| `PRIVATE_DNS_A_RECORD_NAME` | var | Yes (if DNS enabled) | `traefik-ingress` | Record-set name (e.g. `traefik-ingress` -> `traefik-ingress.<zone>`). |

### Dashboard/API test image (conditional)

| Name | Store as | Required | Example | Notes |
|---|---|---:|---|---|
| `TRAEFIK_TEST_IMAGE` | var | Yes (if dashboard enabled) | `nexus.example.local/library/alpine:3.20` | Must be Nexus-hosted and contain **curl** or **wget**. Used only by the in-cluster job for `/api/version` checks. |
