# ai-factory

[AI Software Factory](https://github.com/Majowka/ai-software-factory): GitHub issues and comments become
pull requests, made by Tekton pipelines with OpenCode (OpenCode Go) or Claude Code as the agent engine.
Namespace `ai-factory`; needs `apps/tekton`.

| Piece | Where |
|---|---|
| Pipelines, triggers, CronJob, agent config | remote bases from `Majowka/ai-software-factory` (`agents/`, `tekton/`), ref pinned in `manifests/kustomization.yaml` |
| Agent image | `ghcr.io/majowka/factory-agent:<sha>` (public), tag pinned in `manifests/kustomization.yaml` |
| `pipeline` service account | `manifests/serviceaccount-pipeline.yaml` |
| GitHub webhooks | GitHub App `KlaudiuszAI` → Cloudflare Worker `github-openbot-relay` (route `factory`, `Majowka/infrastructure` `workers/github-openbot-relay/src/factory.js`) → Workers VPC Service `ai-factory-el` → `manifests/cloudflared.yaml` (tunnel `k3s-ai-factory`, outbound only) → `el-new-issue:8080` |
| Secrets | outside gitops (this repo is public), below |

## Secrets (until ESO + Infisical)

Create them before (or right after) the first sync; pods that need a missing secret wait in
`CreateContainerConfigError`. Keep the values in Infisical.

```bash
NS=ai-factory
kubectl create secret generic github-app -n $NS \
  --from-literal=app-id=<APP_ID> --from-literal=installation-id=<INSTALLATION_ID> \
  --from-file=private-key.pem=./<app>.private-key.pem
# The webhook secret of the GitHub App (the Worker forwards the original body and X-Hub-Signature-256 unchanged)
kubectl create secret generic github-webhook-secret -n $NS --from-literal=secret=<GITHUB_APP_WEBHOOK_SECRET>
kubectl create secret generic opencode-go -n $NS --from-literal=api-key=<OPENCODE_GO_API_KEY>
# Claude Code OAuth (Infisical: infra-platform / prod, /platform/claude-oauth, CLAUDE_OAUTH_TOKEN). Without it the tasks
# routed to claude (PLAN, IMPLEMENT of bug issues) fall back to OpenCode.
kubectl create secret generic claude-code-oauth -n $NS --from-literal=token=<CLAUDE_OAUTH_TOKEN>
# Cloudflare tunnel k3s-ai-factory (Infisical: infra-k3s / vrtx-cluster, AI_FACTORY_CLOUDFLARED_TUNNEL_TOKEN)
kubectl create secret generic cloudflared-tunnel -n $NS --from-literal=token=<TUNNEL_TOKEN>
```

The tunnel needs QUIC (UDP 7844) to the Cloudflare edge, and Workers VPC accepts only QUIC. The pod network (flannel over
Tailscale, MTU 1230) is too small for QUIC packets, so `cloudflared` runs with `hostNetwork`.

## First run

1. `kubectl -n argocd get app tekton ai-factory` → `Synced` + `Healthy`;
   `kubectl get clusterinterceptors` lists `github` and `cel`.
2. `kubectl -n ai-factory get pods` → `el-new-issue-*` and `cloudflared-*` running;
   `kubectl -n ai-factory logs deploy/cloudflared` shows `Registered tunnel connection`.
3. In the Worker set `FACTORY_REPO_IDS` (IDs of the repositories the factory serves) and
   `FACTORY_ROUTE_ENABLED=true`. The GitHub App webhook URL stays as it is (the Worker).
4. Open a test issue in one of those repositories and watch `kubectl -n ai-factory get pipelinerun -w`.
   GitHub → App → Advanced → Recent Deliveries shows the Worker's answer: `forwarded: factory`,
   `ignored: factory-route-off`, or `relay error` (502: tunnel or listener down).

The requirements scan (`CronJob requirements-scan`) runs hourly for `Majowka/mesh-manager` (`REPOS`, `MAX_OPEN` and
`suspend` are patched in `manifests/kustomization.yaml`). It only reads issues labeled `requirements`; the task issues
it creates (label `req-<n>`) start a run in mesh-manager only after an `@ai` comment.
