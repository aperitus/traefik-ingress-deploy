# Onboarding an application using Kubernetes Ingress (compatibility mode)

Use this mode when your application Helm chart natively creates a Kubernetes `Ingress` (e.g., Dependency-Track).

## Prerequisites

- Traefik deployed with `routing_mode=ingress` or `routing_mode=both`.
- Your app namespace exists (e.g., `dependency-track`).
- Your app repo has Kubernetes RBAC to deploy into its namespace and create `Ingress`.

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
kubectl label namespace <your-namespace> traefik-gateway-access=true --overwrite
```

(Labels are always strings. The workflow uses `--set-string` to avoid Helm converting `true` into a boolean.)


### Dashboard/API verification (internal-only)
The deployment workflow can enable Traefik's internal API/dashboard on the admin entrypoint for **service-proxy validation** by setting `enable_traefik_dashboard=true` (this sets `api.dashboard=true` and `api.insecure=true`). Use this only for internal/dev validation and do not expose it externally without authentication.
