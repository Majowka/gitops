# ai-factory

[AI Software Factory](https://github.com/Majowka/ai-software-factory): GitHub issues and comments become
pull requests, made by Tekton pipelines with OpenCode (OpenCode Go) or Claude Code as the agent engine.
Namespace `ai-factory`; needs `apps/tekton`.

| Piece | Where |
|---|---|
| Pipelines, triggers, CronJob, agent config | remote bases from `Majowka/ai-software-factory` (`agents/`, `tekton/`), ref pinned in `manifests/kustomization.yaml` |
| Agent image | `ghcr.io/majowka/factory-agent:<sha>` (public), tag pinned in `manifests/kustomization.yaml` |
| `pipeline` service account | `manifests/serviceaccount-pipeline.yaml` |
| Public webhook URL | `manifests/webhook-funnel.yaml`: Tailscale Funnel, `https://ai-factory-webhook.tail07246.ts.net/` |
| Secrets | outside gitops (this repo is public), below |

## Secrets (until ESO + Infisical)

Create them before (or right after) the first sync; pods that need a missing secret wait in
`CreateContainerConfigError`. Keep the values in Infisical.

```bash
NS=ai-factory
kubectl create secret generic github-app -n $NS \
  --from-literal=app-id=<APP_ID> --from-literal=installation-id=<INSTALLATION_ID> \
  --from-file=private-key.pem=./<app>.private-key.pem
kubectl create secret generic github-webhook-secret -n $NS --from-literal=secret="$(openssl rand -hex 32)"  # same value in the GitHub App
kubectl create secret generic opencode-go -n $NS --from-literal=api-key=<OPENCODE_GO_API_KEY>
kubectl create secret generic claude-code-oauth -n $NS --from-literal=token=<output of: claude setup-token>
# Tailscale Funnel: reusable auth key with tag:ai-factory-funnel, plus an empty state secret
kubectl create secret generic webhook-funnel-authkey -n $NS --from-literal=authkey=<TS_AUTHKEY>
kubectl create secret generic webhook-funnel-state -n $NS
```

The tag `tag:ai-factory-funnel` needs `tagOwners` and the `funnel` node attribute in
`Majowka/infrastructure` `tailscale/acl.hujson`.

## First run

1. `kubectl -n argocd get app tekton ai-factory` → `Synced` + `Healthy`;
   `kubectl get clusterinterceptors` lists `github` and `cel`.
2. `kubectl -n ai-factory get pods` → `el-new-issue-*` and `webhook-funnel-*` running.
3. `curl -s -o /dev/null -w '%{http_code}\n' https://ai-factory-webhook.tail07246.ts.net/` from outside the tailnet:
   any HTTP answer from the listener (not a timeout) means Funnel works.
4. Set the GitHub App webhook URL to `https://ai-factory-webhook.tail07246.ts.net/`, open a test issue in a
   repository where the app is installed, watch `kubectl -n ai-factory get pipelinerun -w`.

The requirements scan (`CronJob requirements-scan`) is suspended here. To enable it, patch `REPOS` and
`suspend: false` in `manifests/kustomization.yaml`.

Before using it on real repositories, set the bot login in the `new-issue` CEL filter and the
`GIT_AUTHOR_*` / `GIT_COMMITTER_*` of the publish steps to our GitHub App (upstream values are
`ai-software-factory-agent[bot]`), in `Majowka/ai-software-factory`.
