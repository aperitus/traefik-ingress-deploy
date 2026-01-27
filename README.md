# Traefik Ingress Baseline (AKS via GHES Actions)

Repo-ready deliverables:
- `docs/traefik-ingress-aks-gha-requirements-bom.md`
- `docs/traefik-ingress-project-prompt.md`
- `.github/workflows/deploy-traefik.yaml`
- `helm/traefik/values.template.yaml`

Notes:
- This repo deploys Traefik as an internal-only ingress controller (Azure ILB).
- App repos (e.g., Dependency-Track) deploy later and should set their ingress class to `traefik`.
- Optional private DNS update is supported (A record updated to the ILB IP).
