# Talos machine configs

**Do not start from these templates.** Official path is `talosctl gen config` after the nodes are in maintenance mode (see `docs/lab.md`).

`machineconfig/*.tmpl` is a **patch reference** for what we add on top of generated YAML:

- `machine.install.image` = Image Factory `metal-installer/<schematic>:v1.14.x` matching the ISO
- `machine.kubelet.extraMounts` for `/var/lib/longhorn` (Longhorn on Talos)
- `machine.network.interfaces[].vip` **172.16.30.10** on CP `ens18` (API). Ingress VIP is MetalLB **172.16.30.11**, not Talos.

Generated files (`controlplane.yaml`, `worker.yaml`, `talosconfig`) stay in `talos/generated/` (gitignored).

Factory extensions for this lab:

- `siderolabs/iscsi-tools` — Longhorn (required)
- `siderolabs/util-linux-tools` — Longhorn (required)
- `siderolabs/qemu-guest-agent` — only if Proxmox QEMU Agent is enabled
