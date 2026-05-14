# gitops

GitOps repo for the `Majowka` k3s cluster. Managed by ArgoCD running at
`https://argocd.tail07246.ts.net` (access through Tailscale, `tag:mgmt`).

## Layout

```
bootstrap/
  root-app.yaml          # app-of-apps — entry point for ArgoCD
apps/
  <app-name>/
    Application.yaml     # ArgoCD Application pointing at manifests/
    manifests/           # plain k8s YAML (or Helm/Kustomize)
```

## Adding a new app

1. Drop manifests into `apps/<name>/manifests/`
2. Add `apps/<name>/Application.yaml` (copy from an existing app)
3. PR + merge — root-app sees the new Application and ArgoCD syncs it

## Sync policy

Default is `automated: { prune: true, selfHeal: true }` — ArgoCD reverts manual
changes back to git within ~3 minutes. Opt out per-app for things where manual
intervention is expected (e.g. Traefik during config experiments).

## What stays out of GitOps (for now)

- k3s OS-level baseline (Ansible `roles/bootstrap`, kernel sysctl, Tailscale)
- k3s addons that ship pre-rendered in `/var/lib/rancher/k3s/server/manifests/`
  (metrics-server, coredns-custom — will be migrated in a later phase)
- Secrets — `External Secrets Operator` will pull from Infisical (TBD)

## Related repos

- `Majowka/infrastructure` — Terraform, Ansible (super-repo)
- `Majowka/ans_playbooks` — Ansible roles (k3s install, OS, Helm bootstrap for ArgoCD itself)
- `Majowka/ans_inventory` — inventory + vault (secrets)
