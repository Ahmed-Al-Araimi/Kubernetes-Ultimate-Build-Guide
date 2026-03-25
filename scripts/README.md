# Scripts Notes

This folder contains helper scripts for operator speed.

These scripts are convenience tooling, not cluster installers.

---

## install-k8s-shortcuts.sh

File:

- [install-k8s-shortcuts.sh](./install-k8s-shortcuts.sh)

What it does:

- creates `~/.k8s-lab-shortcuts.sh` with useful aliases/functions
- adds one source line to `~/.bashrc` (if missing)
- loads shortcuts in current shell session

What it does not do:

- does not install Kubernetes
- does not modify kubeadm configuration
- does not edit cluster resources
- does not restart services

Use:

```bash
chmod +x scripts/install-k8s-shortcuts.sh
./scripts/install-k8s-shortcuts.sh
source ~/.bashrc
khelp
```

Examples after install:

- `kn` -> nodes wide view
- `kpa` -> all pods wide view
- `khealth` -> nodes + core namespaces snapshot
- `kapi` -> API endpoint checks
- `kcordon <node>`, `kdrain <node>`, `kuncordon <node>`
- `kwatchall`, `kwatchkuma`, `kwatchgatus`

Remove:

1. delete `~/.k8s-lab-shortcuts.sh`
2. remove this line from `~/.bashrc`:

```bash
[ -f "$HOME/.k8s-lab-shortcuts.sh" ] && source "$HOME/.k8s-lab-shortcuts.sh"
```

