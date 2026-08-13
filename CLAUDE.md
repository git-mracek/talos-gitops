# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This is a **GitOps state repository** — not an application codebase. It declaratively defines a bare-metal
Kubernetes cluster (Talos Linux, on Proxmox VMs on a Hetzner dedicated server) and every workload running on
it. There is no build/test/compile step for this repo itself: the "build" is ArgoCD rendering these Helm
charts and applying them to the cluster. Editing a file here and merging to `main` is the deployment action.

## Repository structure (App of Apps)

```
apps/          ArgoCD Application manifests — one per component, all pointing into charts/*
charts/        Local Helm charts with the actual K8s resources (one directory per component)
talos/         Talos OS cluster config (talconfig.yaml) + SOPS-encrypted secrets (talsecret.yaml)
bootstrap.yaml Root ArgoCD Application; points at apps/ with directory.recurse — the single entrypoint
pub-cert.pem   Public key used by `kubeseal` to encrypt new SealedSecrets for this cluster
```

`bootstrap.yaml` is applied once by hand to bootstrap the cluster; after that, ArgoCD watches `apps/` and
self-manages everything, including its own config (`apps/argocd.yaml` → `charts/argocd`).

Every `apps/*.yaml` is an ArgoCD `Application` with `syncPolicy.automated: {prune: true, selfHeal: true}` —
merging to `main` is a live, unattended production deploy (see `.github/workflows/argocd-sync.yml`, which
pings ArgoCD's webhook over Tailscale on every push to `main` for near-instant sync). There is no staging
environment. Treat every change to `apps/` or `charts/` as something that goes live immediately.

## Two chart patterns in `charts/`

1. **Wrapper charts** around upstream Helm charts (metallb, cert-manager, ingress-nginx, longhorn,
   sealed-secrets, kube-prometheus-stack, cnpg, gitlab-runner, metrics-server). `Chart.yaml` declares the
   upstream chart as a `dependencies:` entry pinned to an exact version; `values.yaml` holds only the local
   overrides; `templates/` (if present) holds extra resources layered on top (e.g. `charts/metallb/templates/config.yaml`
   for `IPAddressPool`/`L2Advertisement`, `charts/argocd/templates/ingress.yaml` for ArgoCD's own Ingress,
   `charts/cert-manager` for the ClusterIssuer). Renovate bumps the `version:` in these `Chart.yaml`
   dependency blocks automatically.
2. **Fully local charts** for actual workloads (n8n, vaultwarden, tuwunel, opnsense-exporter,
   proxmox-exporter). No dependencies — `templates/` contains hand-written Deployment/Service/Ingress/
   NetworkPolicy/CNPG-Cluster/SealedSecret manifests. `charts/n8n` is the fullest example of the workload
   pattern (app + DB + ingress + rbac + network-policies + sealed-secrets) and is a good template to copy
   when adding a new workload.

Each `apps/<name>.yaml` maps 1:1 to `charts/<name>`, sets the target `namespace`, and usually sets
`syncOptions: [CreateNamespace=true]`. Wrapper charts around operators typically add `ServerSideApply=true`
(cert-manager, cnpg, kube-prometheus-stack, exporters) because their CRDs are too large for client-side
apply. Some apps also set `managedNamespaceMetadata.labels` to apply Pod Security Standards to the namespace
(e.g. `ingress-nginx` → baseline, `longhorn`/`metallb` → privileged, since they need host access).

## Adding a new workload

1. Create `charts/<name>/Chart.yaml` + `templates/` following the `charts/n8n` layout (or `charts/vaultwarden`
   for a simpler example).
2. Add `apps/<name>.yaml` — copy an existing Application manifest, change `name`, `path`, and `destination.namespace`.
3. Default-deny NetworkPolicies are the norm per namespace (see `charts/n8n/templates/network-policies.yaml`):
   deny-all baseline, then explicit allows for DNS egress, intra-namespace, ingress-nginx ingress, monitoring
   scrape, CNPG operator access, and any required internet egress (S3 backups, SMTP, webhooks).
4. Never commit plaintext secrets. Use `kubeseal` with `pub-cert.pem` to produce `SealedSecret` resources
   (see `charts/n8n/templates/sealed-secret-*.yaml`); only the `SealedSecrets` controller in-cluster can
   decrypt them.
5. Databases use CloudNativePG (`postgresql.cnpg.io/v1 Cluster`), storage class `longhorn` or
   `longhorn-crypto` (encrypted), and back up to S3 (`barmanObjectStore`) — see `charts/n8n/templates/db.yaml`.

## Talos / cluster config (`talos/`)

- `talconfig.yaml` is the source of truth for the 3-node control-plane cluster (talhelper format): Talos
  version, Kubernetes version, node list (`10.10.10.11-13`, VIP `10.10.10.10`), image factory extensions
  (qemu-guest-agent, iscsi-tools, util-linux-tools), and machine patches (kernel modules for Longhorn/LUKS,
  extra kubelet mount for Longhorn storage, nameservers, NTP).
- `talsecret.yaml` is SOPS-encrypted (age recipient in `talos/.sops.yaml`) — never edit or decrypt it without
  the corresponding age private key; it is not something to open or print.
- Changes here require regenerating machine configs with `talhelper` and applying with `talosctl`/`talhelper`
  outside of ArgoCD (Talos itself is not managed by GitOps, only the Kubernetes workloads on top of it are).

## Network model (needed to reason about ingress/egress changes)

Double-NAT topology: Internet → Proxmox (`eno1`, DNAT) → OPNsense VM (router/firewall, `10.10.10.1`) →
internal LAN `10.10.10.0/24`. Talos API VIP is `10.10.10.10`; MetalLB hands out `10.10.10.200-202`
(`vip-pool`, manually assigned — e.g. `.200` is the Ingress VIP) and `10.10.10.203-250` (`general-pool`,
auto-assign) — see `charts/metallb/templates/config.yaml`. All management access (SSH, Proxmox GUI, Talos
API) is Tailscale-only, never publicly exposed; only HTTP/HTTPS (and app-specific ports like Minecraft
Bedrock) are port-forwarded from the public IP through to the Ingress VIP. Full details and a packet-flow
walkthrough are in `README.md`.

## Conventions

- Commits follow Conventional Commits (`feat`, `fix`, `chore`, `refactor`, `docs`, `style`, `perf`), with an
  optional scope, imperative mood, no trailing period — see `CONTRIBUTING.md`. Commits must be signed. PRs
  are squash-merged.
- Dependency updates (Helm chart versions in `Chart.yaml`, and image tags in `values.yaml` matched via the
  custom regex manager in `renovate.json`) are automated by Renovate and land as individual PRs — recent
  commit history is almost entirely these bumps. `automerge` is disabled, so they need review before merging.
