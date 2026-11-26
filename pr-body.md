<!-- claude-projects-attribution: {"github_login":"kamcio2603"} -->
_Requested by **Kamil** · [project thread](https://claude.ai/code/project/chan_01TqfyFBygvWa2jHU5UDqSJv?thread=cmsg_01TqfyFBygvWa2jHU5UDqSJvUuTvqeDgdYMHPpgSZ2HGaf)_

## Co i dlaczego

PoC autoskalowania workerów k3s: Karpenter + provider Proxmox (chart `karpenter-provider-proxmox` 0.7.1, app v0.17.0) na testowym klastrze PVE `KRA-PROXMOXC-03` (region `kra`, sieć SDN `kamilsim`). Nowa aplikacja `apps/karpenter-proxmox`: Application (chart + manifesty z gita), `ProxmoxTemplate` (Ubuntu noble), `ProxmoxNodeClass`, `NodePool` (limit 4 vCPU / 8 GiB, taint `pool=kamil-poc`, konsolidacja `WhenEmpty` 5 min) i szablon cloud-init (Tailscale + k3s agent, `provider-id` z kubeleta).

## Ważne

- Kontroler stoi tylko na węźle `kamil-karpenter-host` (`dedicated=karpenter`), bo tylko on widzi API PVE.
- Zero sekretów w repo. Przed merge trzeba ręcznie utworzyć w `kube-system` (z Infisical) `karpenter-provider-proxmox` i `karpenter-template-values`, patrz `apps/karpenter-proxmox/README.md`. Bez nich pod kontrolera będzie w CrashLoop (bez wpływu na resztę klastra).
- Bez Proxmox CCM (mastery mają `disable-cloud-controller`). Ryzyko opisane w README.
- Merge dopiero po review.

## Test plan

- [ ] Sekrety utworzone, `argocd app get karpenter-proxmox --core` = Synced/Healthy
- [ ] Pod z `nodeSelector: pool=kamil-poc` w Pending -> NodeClaim -> VM na PVE -> węzeł Ready
- [ ] Usunięcie poda -> po 5 min węzeł skonsolidowany i VM skasowana

🤖 Generated with [Claude Code](https://claude.com/claude-code)
