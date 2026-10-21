# Lab bring-up (overmind)

Follow the official [Talos on Proxmox](https://docs.siderolabs.com/talos/v1.14/platform-specific-installations/virtualized-platforms/proxmox) guide, then Flux from this repo.

Guest create is a **hard gate**. Live Mk II is untouched.

## Image (there is an ISO)

Factory does not look like a distro download page. The artifact for Proxmox is still an ISO:

```
https://factory.talos.dev/image/<schematic>/<version>/metal-amd64.iso
```

Vanilla schematic (no extensions), v1.14.4 example from the Proxmox docs:

```
https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.14.4/metal-amd64.iso
```

ISO boots **in RAM** (maintenance mode). Nothing is written to disk until `talosctl apply-config`. The matching **installer image** (`factory.talos.dev/metal-installer/<schematic>:v1.14.x`) is what actually lands on `/dev/sdX`.

For this lab, build a **custom schematic** at [factory.talos.dev](https://factory.talos.dev):

| Extension | Why |
|---|---|
| `siderolabs/qemu-guest-agent` | Optional. Only if Proxmox QEMU Guest Agent is enabled |
| `siderolabs/iscsi-tools` | Longhorn |
| `siderolabs/util-linux-tools` | Longhorn |

Same schematic on ISO **and** `machine.install.image`. Do not enable Guest Agent in Proxmox unless the ISO has that extension (log spam otherwise).

Do not commit `.iso` files.

## Proxmox VM settings (from the v1.14 guide)

| Setting | Value | Why |
|---|---|---|
| BIOS | OVMF (UEFI) + 4MB EFI disk, Pre-Enroll Keys **off** | |
| Machine | `q35` | |
| CPU | `host`, 2+ cores | Talos wants x86-64-v2; `kvm64` is too old |
| Memory | 4 GiB lab (official rec is 4 CP / 8 workers — we do not have that headroom) | **Ballooning off** — Talos has no memory hotplug |
| Disk | **VirtIO SCSI** (not VirtIO SCSI **Single**) | Single hangs bootstrap ([#11173](https://github.com/siderolabs/talos/issues/11173)) |
| Format | Raw | |
| NIC | virtio on `vmbr0` (or the ranch bridge) | |
| Serial | `ttyS0` | Early boot / DHCP debug |

Three VMs: `hzr-cp-1`, `hzr-w-1`, `hzr-w-2`. Upload ISO to local storage, attach as CD.

## After the VMs boot (official sequence)

ISO → maintenance console shows DHCP IP. Then **`talosctl gen config`**, not hand-edited templates.

```bash
export CONTROL_PLANE_IP=<from console>
# workers later: WORKER_IP=(...)

# ISO is RAM-only; unmount CD before install so the install disk is obvious
talosctl get disks --insecure --nodes $CONTROL_PLANE_IP

export CLUSTER_NAME=hzr-lab
export DISK_NAME=<id from get disks, often sda>

talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 \
  --install-disk /dev/$DISK_NAME \
  --output-dir talos/generated/

# Patch generated YAML: Longhorn extraMounts, factory installer image
# (see talos/README.md). Do not commit talos/generated/.

talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP \
  --file talos/generated/controlplane.yaml

# start workers, same worker.yaml, then:
# talosctl apply-config --insecure --nodes $W_IP --file talos/generated/worker.yaml

talosctl --talosconfig=./talos/generated/talosconfig config endpoints $CONTROL_PLANE_IP
talosctl bootstrap --nodes $CONTROL_PLANE_IP --talosconfig=./talos/generated/talosconfig
talosctl kubeconfig ./kubeconfig --nodes $CONTROL_PLANE_IP --talosconfig=./talos/generated/talosconfig
```

Lab VIPs on VLAN 30 (UniFi DHCP pool starts at `.32` so `.10`/`.11` stay free): API **172.16.30.10** (`api.hzr.lab`, Talos `ens18` vip), ingress **172.16.30.11** (`ingress.hzr.lab`, MetalLB L2). Point `monitor`/`grafana`/`prometheus.hzr.lab` at `.11`. If DHCP is missing, the Proxmox guide has a GRUB `ip=` one-shot.

## Flux

```bash
export KUBECONFIG=./kubeconfig
flux bootstrap github \
  --owner=Segfault198 \
  --repository=hzr-platform-mk3 \
  --branch=master \
  --path=clusters/lab \
  --personal
```

`clusters/lab/kustomization.yaml` must list `flux-system` plus `infrastructure.yaml` and `apps.yaml`. If bootstrap rewrote that file, add those two lines back and push.

Talos v1.14 / k8s 1.37 defaults unlabeled namespaces to PSA **baseline**. Longhorn (privileged + hostPath), MetalLB speaker, and Traefik need `pod-security.kubernetes.io/enforce: privileged` on `longhorn-system`, `metallb-system`, `traefik`, and `hzr-sites`.

Traefik is `Service type: LoadBalancer` at **172.16.30.11** (MetalLB L2). Container ports stay 8000/8443; Service exposes 80/443. Not hostNetwork.

Longhorn lab: workers need `node.longhorn.io/create-default-disk=true` (Helm `createDefaultDiskLabeledNodes`). Durable via Talos v1.14 `KubeNodeConfig.labels` (`talos/patches/worker-longhorn-labels.yaml`); do not put it on the CP. kubectl labels alone do not survive a Talos kubelet rewrite. Default StorageClass from the chart baked `numberOfReplicas: "3"` (parameters immutable); use `longhorn-lab` (2 replicas) until that SC is recreated. `defaultSettings.defaultReplicaCount` is the chart key, not `replicaCount`.

Lab monitoring: kube-prometheus-stack on `longhorn-lab` PVCs (Prometheus 8Gi / Grafana 2Gi). LAN Hosts `grafana.hzr.lab` and `prometheus.hzr.lab`. Alertmanager off. Grafana login is the lab Helm value (admin). Grafana 256Mi limit OOMKilled on first dashboards load; lab limit is 768Mi. Prometheus is pinned to `hzr-w-1`, Grafana to `hzr-w-2` so one 4Gi worker does not run both.

## Acceptance

- PVC binds; volume remounts after a node reboot
- Kill a worker; Longhorn volume stays healthy on remaining replicas
- `monitor` from `hzr-sites` at `Host(monitor.hzr.lab)`

## Out of v1

Public TLS, hamsterbot, hosting provider.
