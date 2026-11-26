# karpenter-proxmox (PoC)

Autoskalowanie workerów k3s przez Karpenter + provider Proxmox (`sergelogvinov/karpenter-provider-proxmox`,
chart 0.7.1, app v0.17.0). Testowy klaster PVE `KRA-PROXMOXC-03` (region `kra`), sieć SDN `kamilsim` (10.98.0.0/24, SNAT).

## Założenia

- Kontroler działa tylko na węźle `kamil-karpenter-host` (`dedicated=karpenter:NoSchedule`): tylko ten węzeł widzi API PVE.
- Nowe węzły: Ubuntu noble, Tailscale (`tag:k3s-worker`) + k3s agent (`node-ip` z Tailscale, `flannel-iface=tailscale0`),
  `provider-id` ustawiony przez kubelet. Bez Proxmox CCM (mastery mają `disable-cloud-controller`).
- NodePool `kamil-poc`: limit 4 vCPU / 8 GiB, taint `pool=kamil-poc:NoSchedule`, konsolidacja `WhenEmpty` po 5 min.

## Sekrety (poza repo, tworzone ręcznie z Infisical `infra-platform/prod`)

| Secret (kube-system) | Klucze | Źródło |
|---|---|---|
| `karpenter-provider-proxmox` | `config.yaml` (`clusters: [{url, token_id, token_secret, region: kra, insecure: true}]`) | `/kra/kamil-karpenter` (`API_HOST`, `API_TOKEN_ID`, `API_TOKEN`) |
| `karpenter-template-values` | `K3sURL`, `K3sToken` (agent-token), `K3sVersion`, `TsAuthKey` | master k3s, `/tailscale/AUTH_KEY` |

Zmiany tylko przez PR. Test: pod z `nodeSelector: {pool: kamil-poc}` i tolerancją `pool=kamil-poc` ->
nowy NodeClaim -> VM na PVE -> węzeł Ready.

## Znane ryzyka PoC

- Bez CCM: węzły są kojarzone z VM przez `provider-id` z kubeleta, nie przez CCM.
- `TsAuthKey` to współdzielony klucz (nie ephemeral): usunięte węzły zostają w tailnecie jako offline.
- `insecure: true` dla API PVE (certyfikat self-signed na teście).
