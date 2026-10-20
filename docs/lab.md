# Lab bring-up (overmind)

Guest create is a **hard gate**. This file assumes the three VMs exist and you have a Talos ISO.

## ISO

Build (or download) from [Image Factory](https://factory.talos.dev) with **system extensions**:

- `siderolabs/qemu-guest-agent`
- `siderolabs/iscsi-tools`
- `siderolabs/util-linux-tools`

Same schematic ID on all three nodes. Drop the ISO on overmind; do not commit `.iso` files here.

## After VMs exist

Fill IPs in copies of the templates (keep generated files out of git):

```bash
cp talos/machineconfig/controlplane.yaml.tmpl talos/generated/controlplane.yaml
cp talos/machineconfig/worker.yaml.tmpl talos/generated/worker.yaml
# edit LAB_* placeholders
```

Apply and bootstrap (example):

```bash
talosctl apply-config --insecure --nodes $CP_IP --file talos/generated/controlplane.yaml
talosctl apply-config --insecure --nodes $W1_IP --file talos/generated/worker.yaml
talosctl apply-config --insecure --nodes $W2_IP --file talos/generated/worker.yaml
talosctl config endpoint $CP_IP
talosctl config node $CP_IP
talosctl bootstrap
talosctl kubeconfig ./kubeconfig
```

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

After bootstrap, `clusters/lab/kustomization.yaml` must list `flux-system` plus the two Flux Kustomizations in this repo (`infrastructure.yaml`, `apps.yaml`). If bootstrap rewrote that file, add those two lines back and push.

## Acceptance

- PVC binds; volume remounts after a node reboot
- Kill a worker; Longhorn volume stays healthy on remaining replicas
- `monitor` from `hzr-sites` serves on the lab IngressRoute (`monitor.hzr.lab` — `/etc/hosts` or LAN DNS)

## Out of v1

Public TLS, hamsterbot, MetalLB, hosting provider.
