# Kubernetes Ultimate Build Guide

<img src="./pictures/k8s%20topology.png" alt="Kubernetes HA Lab Topology" width="400">

This guide originated while preparing a Wazuh HA SOC lab.  
The Kubernetes foundation proved strong enough to stand on its own, so it is published here as a practical guide for others.

This is a tested, repeatable path. If you follow the steps exactly, you should end up with a working multi-control-plane Kubernetes environment for lab work and serious infrastructure testing.

## Resources Included

When you clone this repo, you get:

- [Build Guide](./build-guide.md): full VM-by-VM Kubernetes HA build flow.
- [API Endpoint Migration Runbook](./api-endpoint-migration-runbook.md): move API endpoint to stable `api:6443` VIP flow.
- [Situation-Based Validation Playbook](./validation-playbook.md): pass/fail scenarios for resilience and recovery.
- [Key Commands](./key-commands.md): daily command reference for operations and debugging.
- [FAQ](./faq.md): practical troubleshooting by symptom.
- [Gatus Stateless Demo](./demos/gatus-stateless/gatus-stateless-runbook.md) and [manifest](./demos/gatus-stateless/gatus-stateless-manifest.yaml).
- [Uptime Kuma Stateful Demo](./demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md) and [manifest](./demos/uptime-kuma-stateful/uptime-kuma-stateful-manifest.yaml).
- [Shortcut Installer Script](./scripts/install-k8s-shortcuts.sh) and [script notes](./scripts/README.md).
- [IP Index](./ip-index.md) for lab addressing.

## Start Here

1. [Build Guide](./build-guide.md)
2. [Situation-Based Validation Playbook](./validation-playbook.md)
3. [Key Commands](./key-commands.md)
4. [Gatus Demo (Stateless)](./demos/gatus-stateless/gatus-stateless-runbook.md)
5. [Uptime Kuma Demo (Stateful)](./demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md)
6. [API Endpoint Migration Runbook](./api-endpoint-migration-runbook.md)
7. [FAQ](./faq.md)

## What This Guide Gives You

- a full `kubeadm` HA build flow (control planes + workers)
- clear command order and VM-by-VM execution
- practical troubleshooting for real failure cases
- a demo path to prove resilience and continuity

## Quick References

- [IP Index](./ip-index.md)
- [Situation-Based Validation Playbook](./validation-playbook.md)
- [Key Commands](./key-commands.md)
- [Shortcut Installer Script](./scripts/install-k8s-shortcuts.sh)
- [Scripts Notes](./scripts/README.md)
- [Gatus Demo Manifest](./demos/gatus-stateless/gatus-stateless-manifest.yaml)
- [Uptime Kuma Demo Manifest](./demos/uptime-kuma-stateful/uptime-kuma-stateful-manifest.yaml)

## Notes

- Core Kubernetes build does not require a storage layer.
- Use Gatus for stateless fast-failover demonstration.
- Use Uptime Kuma for stateful PVC behavior demonstration.


