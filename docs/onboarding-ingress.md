# Onboarding an application using Kubernetes Ingress (compatibility mode)

Use this mode when your application Helm chart natively creates a Kubernetes `Ingress` (e.g., Dependency-Track).

## Prerequisites

- Traefik deployed with `routing_mode=ingress` or `routing_mode=both`.
- Your app namespace exists (e.g., `dependency-track`).
- Your app repo has Kubernetes RBAC to deploy into its namespace and create `Ingress`.
- When the platform workflow runs with `enable_tls=true` (default), HTTP is redirected to HTTPS at the Traefik entrypoint.
  That means **Ingress resources must be TLS-capable** (define `spec.tls`) or clients will be redirected to HTTPS where no router exists (typically `404` or TLS mismatch).
- The TLS Secret referenced by the Ingress must exist **in the same namespace as the Ingress**.

### TLS Secret requirement (Ingress)

Kubernetes Ingress TLS looks up `spec.tls[].secretName` in the **Ingress namespace**. The wildcard TLS Secret created by this repo in the `traefik` namespace is not automatically available to app namespaces.

Options:

1) **Create the TLS Secret as part of the app deployment** (recommended)
   - Your app pipeline creates `wildcard-tls` (or another name) in its namespace from the same wildcard certificate material.

2) **Manual creation (operator)**
   - Example (replace file paths):

```bash
kubectl -n myapp create secret tls wildcard-tls \
  --cert=./wildcard.crt \
  --key=./wildcard.key \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Minimal Ingress example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myapp.logiki.co.uk
      secretName: wildcard-tls
  rules:
    - host: myapp.logiki.co.uk
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

## Access control recommendations

Ingress is flexible but less tenancy-aware than Gateway API. For security:

1) **Require explicit ingress class**  
   Enforce `spec.ingressClassName: traefik` so only Traefik processes ingresses.

2) **Restrict hostnames**  
   Enforce allowed FQDN patterns (e.g., `*.logiki.co.uk`) to prevent unapproved host takeover.

3) **Require TLS**  
   Enforce `spec.tls` and forbid cleartext-only ingresses unless explicitly approved.

4) **Restrict annotations**  
   Many ingress controllers allow powerful annotations; restrict or deny them via policy unless you explicitly support them.

## Operational note

If you later migrate to Gateway API, keep the app’s Service as-is; replace the Ingress with an `HTTPRoute` that targets that Service.


### Namespace label requirement (Gateway API Selector mode)
If the platform Gateway is configured with `namespacePolicy.from: Selector`, your app namespace must carry the label **as a string**:

```bash
kubectl label namespace <your-namespace> traefik-gateway-access=enabled --overwrite
```

(Labels are always strings, but the deployment workflow explicitly rejects boolean-like values (`true/false/yes/no/on/off/null/~`).
Use a clear string value with at least one letter (default: `enabled`).)


### Dashboard/API verification (internal-only)
The deployment workflow can enable Traefik's internal API/dashboard on the admin entrypoint by setting `enable_traefik_dashboard=true` (this sets `api.dashboard=true` and `api.insecure=true`).

Validation is performed internally by creating a `traefik-dashboard` **ClusterIP** Service and running an **in-cluster Job** that calls `http://traefik-dashboard:8080/api/version` using a Nexus-hosted test image (must include `curl` or `wget`).

Use this only for internal/dev validation and do not expose it externally without authentication.
