# HZR Platform Mk III

Talos Linux + Flux GitOps + Longhorn. **Lab-first** on Proxmox (overmind).

Live public stack remains [`hzr-platform-mk2`](https://github.com/Segfault198/hzr-platform-mk2) until a later DNS cut. Do not point Flux at this repo on the EU cluster.

Static HTML lives in [`hzr-sites`](https://github.com/Segfault198/hzr-sites). Apps (hamsterbot, …) stay in their own repos.

## Lab v1

| | |
|---|---|
| Nodes | `hzr-cp-1` + `hzr-w-1` + `hzr-w-2` (VMs, not LXC) |
| Size | 2 vCPU / 4 GiB / ~32–40 GiB disk |
| OS | Talos ISO you supply (Image Factory: `qemu-guest-agent`, `iscsi-tools`, `util-linux-tools`) |
| Net | LAN only |
| Must work | Longhorn replicas, Flux, one site from `hzr-sites` |

Bring-up: [`docs/lab.md`](docs/lab.md).

## Layout

```
clusters/lab/          Flux path (bootstrap --path clusters/lab)
k8s/infrastructure/    Longhorn + Traefik HelmReleases
k8s/apps/              GitRepository for hzr-sites
talos/                 machineconfig templates (IPs filled locally, not in git)
```

## Default branch

`master`
