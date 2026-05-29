# gitops

<!-- AI-CONTEXT-START -->

## Quick Reference

- **Build**: `kubectl apply --dry-run=client -f apps/`
- **Test**: `kustomize build bootstrap/ | kubectl apply --dry-run=client -f -`
- **Deploy**: Managed by ArgoCD — push to main and ArgoCD syncs automatically

## Project Overview

GitOps source of truth for Kubernetes manifests across the VRTX homelab
infrastructure. ArgoCD watches this repo and syncs all application state.
This repo is **public** — no secrets should ever be committed here; use
External Secrets Operator + Infisical for secret injection.

## Architecture

- `apps/` — ArgoCD Application manifests per service
- `bootstrap/` — cluster bootstrap (ArgoCD install, app-of-apps)
- Direct `kubectl apply` is discouraged — let ArgoCD handle sync

## Conventions

- Commits: [Conventional Commits](https://www.conventionalcommits.org/)
- Branches: `feature/`, `bugfix/`, `hotfix/`, `refactor/`, `chore/`
- **No secrets** — this repo is public; all secrets via ESO + Infisical

## Key Files

| File | Purpose |
|------|---------|
| `.agents/AGENTS.md` | Project-specific agent instructions |
| `TODO.md` | Task tracking |
| `apps/` | ArgoCD Application definitions |
| `bootstrap/` | Cluster bootstrap manifests |

<!-- AI-CONTEXT-END -->

