# Onboarding an application using Gateway API (preferred security mode)

This mode uses the **Kubernetes Gateway API** with Traefik as the controller. Applications publish `HTTPRoute` resources that attach to a shared `Gateway`.

## Why this is more secure by default

Gateway API provides a first-class tenancy mechanism: **who can attach routes** is controlled by the Gateway listener. In this repo we recommend:

- `GATEWAY_ALLOWED_ROUTES_FROM=Selector`
- App namespaces must carry a label (e.g., `traefik-gateway-access=enabled`) to attach routes.

> Note: Avoid boolean-like values such as `true`/`false` for this label. Labels are strings, but YAML may parse unquoted `true`/`false` as booleans and fail Gateway validation.

Traefik’s Helm chart provides built-in values to create a default `Gateway` and set the namespace policy. (See chart `gateway.listeners.*.namespacePolicy` and `providers.kubernetesGateway.enabled`.) 

## Prerequisites

1) Traefik deployed with `routing_mode=gateway` or `routing_mode=both`.
2) Gateway API CRDs installed on the cluster. The Traefik docs reference installing the Standard channel CRDs via the Gateway API release manifest.   
   - In air-gapped environments, mirror the manifest internally and apply it from your trusted source.
3) Your app namespace exists, and you can label it.

## Onboarding steps

### Step 1 — Label the namespace (Selector policy)

Assuming your platform vars are left at defaults:

- `GATEWAY_ALLOWED_ROUTES_LABEL_KEY=traefik-gateway-access`
- `GATEWAY_ALLOWED_ROUTES_LABEL_VALUE=enabled`

Label the app namespace:

```bash
kubectl label namespace myapp traefik-gateway-access=enabled --overwrite
```

### Step 2 — Create an HTTPRoute

Example `HTTPRoute` that routes `myapp.logiki.co.uk` to Service `myapp:80`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp
  namespace: myapp
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
  hostnames:
    - myapp.logiki.co.uk
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: myapp
          port: 80
```

### Step 3 — TLS options

Gateway API TLS is configured at the **Gateway listener** (not on each HTTPRoute).

In this repo, TLS enablement is workflow-driven:

1) Store your wildcard certificate in the GitHub Environment as secrets:
   - `ELOKO_WILDCARD_CRT` (PEM certificate; include full chain)
   - `ELOKO_WILDCARD_KEY` (PEM private key)

2) Run the deploy workflow with:
   - `routing_mode=gateway` or `routing_mode=both`
   - `enable_tls=true` (default)

The workflow will:
- Create/update a Kubernetes TLS Secret in the `traefik` namespace (default name: `wildcard-tls`, configurable via `vars.TLS_SECRET_NAME`)
- Add a `websecure` (**HTTPS**) listener to the shared `Gateway` that references that Secret
- Enforce **Option B**: keep HTTP open but redirect **HTTP → HTTPS** at the Traefik entrypoint (`web` → `websecure`)
- Enforce a platform-wide redirect: keep HTTP open but redirect **HTTP → HTTPS** (`web` → `websecure`)

Verify listeners:

```bash
kubectl -n traefik get gateway traefik -o yaml
```

> Some Gateway API controllers require binding an `HTTPRoute` to a specific listener via `parentRefs[].sectionName`.
> If required, set `sectionName: websecure` on the parentRef.

## Controlling access

- **Most restrictive:** `Same` — only routes in the `traefik` namespace attach (usually too restrictive).
- **Broad:** `All` — any namespace can attach (usually too permissive).
- **Recommended:** `Selector` — only labeled namespaces attach.

This is the core control that makes Gateway API attractive for shared ingress planes. 


### Dashboard/API verification (internal-only)
The deployment workflow can enable Traefik's internal API/dashboard on the admin entrypoint by setting `enable_traefik_dashboard=true`.

Validation is performed internally by creating a `traefik-dashboard` **ClusterIP** Service and running an **in-cluster Job** that calls `http://traefik-dashboard:8080/api/version` using a Nexus-hosted test image (must include `curl` or `wget`).

Use this only for internal/dev validation and do not expose it externally without authentication.
