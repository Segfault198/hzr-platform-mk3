# Talos machine configs

Templates only. Copy to `talos/generated/` (gitignored) and replace `LAB_*` placeholders.

Required Image Factory extensions (bake into the ISO):

- `qemu-guest-agent` — Proxmox
- `iscsi-tools` — Longhorn
- `util-linux-tools` — Longhorn

Worker extraMount `/var/lib/longhorn` is in the worker template. Control plane can run Longhorn too on a 3-node lab (label nodes `node.longhorn.io/create-default-disk=true` after join).
