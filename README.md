# homelab-gitops

Replaces the old `argocd-gitops` folder, which was a generated scaffold that
never matched the actual cluster (pointed at `flux-gitops.git`, still used
`ingress-nginx` instead of MetalLB + Traefik/Gateway API, had no Jellyfin app
at all). This repo is split in two, matching how the two halves actually get
touched — provisioning almost never, GitOps constantly.

## 01-provisioning/

The tested Terraform + Ansible pipeline from `k8s-proxmox` — on GitHub as
`KubernetesClusterScripts` (`k8s-proxmox` is just the local clone dir name
used throughout the scripts, e.g. `~/k8s-proxmox`) — 3-node cluster (kubeadm
1.31, Calico 3.28.2, privileged LXC on Proxmox), copied over in full —
`terraform/{main.tf,variables.tf,outputs.tf,terraform.tfvars.example}`,
`ansible/{ansible.cfg,group_vars,roles/{common,master,worker},site.yml}`,
`deploy.sh`. Content is unchanged from your tested version, decoded
byte-for-byte from Drive — diff it against your live copy if you want to
confirm nothing drifted.

Two things intentionally weren't carried over:
- `terraform.tfvars` (live credentials) — never belongs in git. Keep using
  `terraform.tfvars.example` as the template; `.gitignore` already excludes
  the real file.
- `inventory/hosts.ini` — generated fresh by `deploy.sh` on every run from
  Terraform output, not static content.

I added one new play to `site.yml`: **Install ArgoCD**, appended after the
original bootstrap/master/worker/verify plays, tagged `argocd` so it can be
skipped with `--skip-tags argocd`. It installs ArgoCD on the master, waits
for `argocd-server` to roll out, then applies `02-gitops/clusters/homelab/
{project.yaml,root-app.yaml}` — that's the handoff into `02-gitops/`, which
takes it from there.

## 02-gitops/

Everything ArgoCD manages after that handoff, as an app-of-apps:

```
clusters/homelab/
  project.yaml          AppProject — correct repoURL + Helm sources
  root-app.yaml          the one Application you apply manually
  applications/           child Applications, one per component
infrastructure/homelab/
  metallb/                IPAddressPool + L2Advertisement
  gateway/                 Gateway API CRDs + Gateway + HTTPRoutes
  cert-manager/            ClusterIssuer (unchanged from old repo)
apps/
  base/jellyfin/           Deployment + Service
  base/jellyseerr/         Deployment + Service
  homelab/                  overlay: namespace + both bases together
```

**Sync-wave order:** metallb + traefik + cert-manager + reloader (wave 0) →
metallb-config + gateway-config + cert-manager-config + kube-prometheus-stack
(wave 1) → apps (wave 2, Jellyfin/Jellyseerr). Same pattern the old repo
already used for cert-manager — extended to cover the whole networking layer
now that MetalLB and Gateway API replace ingress-nginx.

### Bootstrap

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Load the SOPS age private key so KSOPS (below) can decrypt secrets.
# The key itself is never stored in this repo or in Drive — keep your own
# copy safe (e.g. ~/age.agekey) and load it here.
kubectl create secret generic sops-age -n argocd --from-file=age.agekey=./age.agekey

kubectl apply -f 02-gitops/clusters/homelab/project.yaml
kubectl apply -f 02-gitops/clusters/homelab/root-app.yaml
```

Everything else — MetalLB, Traefik, the Gateway, cert-manager, monitoring,
Jellyfin, Jellyseerr — comes up on its own from there.

### SOPS secrets need KSOPS in the repo-server

`.sops.yaml` only carries the age *public* key so secrets can be encrypted at
rest — Argo CD itself doesn't decrypt SOPS natively (unlike Flux), so it
needs the **KSOPS** kustomize plugin in `argocd-repo-server` before any
encrypted secret will apply cleanly:

- Mount the `sops-age` secret (created above) into `argocd-repo-server` as
  `SOPS_AGE_KEY_FILE` (e.g. `/home/argocd/.config/sops/age/keys.txt`).
- Add a KSOPS init-container to install the plugin binary, and enable
  `--enable-alpha-plugins --enable-exec` for kustomize via the `argocd-cm`
  ConfigMap.
- Reference: https://github.com/viaduct-ai/kustomize-sops#argo-cd-integration

None of the current apps (Jellyfin, Jellyseerr) use a secret, so the cluster
comes up fine without KSOPS — set it up before the first encrypted file
lands (e.g. the Grafana admin secret referenced by
`kube-prometheus-stack.yaml`).

To encrypt a new secret once KSOPS is wired up:

```bash
sops --encrypt --in-place apps/homelab/<app>/secret.yaml
```

### Jellyfin database

Kept on the default SQLite backend. Postgres support in Jellyfin is still an
unofficial, experimental plugin as of mid-2026, not something upstream
ships — not worth the stability risk for a homelab media server. If that
changes, the swap is contained to `apps/base/jellyfin/` (add a
`apps/base/postgres/` StatefulSet + Secret, wire env vars into the Jellyfin
Deployment).

### Before this actually works

- Push this to a real repo at `github.com/ShijoeBytesBric/homelab-gitops`
  (placeholder used throughout — swap for your actual path if different).
- Confirm your worker node still has `/mnt/mediadisk` bind-mounted and is
  labeled `media-node=true`, and that `/opt/jellyfin/{config,cache}` and
  `/opt/jellyseerr/config` exist on it — same prerequisites as before, ArgoCD
  doesn't create those for you.
- DNS/hosts entries for `jellyfin.home.lab` / `jellyseerr.home.lab` still
  point at whatever IP MetalLB hands Traefik (check `kubectl -n traefik get
  svc traefik`).
