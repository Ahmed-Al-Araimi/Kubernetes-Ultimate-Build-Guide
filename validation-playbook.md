# Situation-Based Validation Playbook

This playbook validates that the cluster is not only built, but operationally solid under real conditions.

Lab scope used in this repo:

- control planes: `cp01`, `cp02`, `cp03`
- load balancers: `lb01`, `lb02`
- workers: `wk01`, `wk02`
- API endpoint target: `api:6443` backed by VIP `10.10.10.9`

Use this after:

1. [build-guide.md](./build-guide.md)
2. [api-endpoint-migration-runbook.md](./api-endpoint-migration-runbook.md)

---

## How to Use This Playbook

- Run scenarios in order.
- Stop when a scenario fails.
- Fix the failure before moving to the next scenario.
- Capture outputs as evidence (commands included in this playbook).

Command scope:

- run `kubectl` checks from any healthy control-plane VM
- run LB tests from LB VMs plus one control-plane VM
- run worker tests from control-plane VM (for `kubectl`) and worker VM (for power/restart actions)

---

## Scenario 0: Baseline Snapshot (Mandatory)

Goal:

- confirm healthy baseline before failure testing

Run on any control-plane VM:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get pods -n calico-system -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 120
```

Pass:

- all nodes are `Ready`
- no critical system pods stuck in `Pending`, `CrashLoopBackOff`, or `Unknown`

---

## Scenario 1: API Endpoint Consistency (Mandatory)

Goal:

- verify cluster uses the stable HA endpoint (`api:6443`) consistently

Run on any control-plane VM:

```bash
getent hosts api
nc -vz api 6443
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep controlPlaneEndpoint
kubectl config view --minify | grep server
```

Pass:

- `api` resolves to `10.10.10.9`
- `nc` succeeds on TCP 6443
- kubeadm config shows `controlPlaneEndpoint: api:6443`
- kubeconfig server shows `https://api:6443`

---

## Scenario 2: Load Balancer Failover (Mandatory)

Goal:

- prove API remains reachable when active LB changes

Step A: identify which LB currently owns the VIP

Run on `lb01` and `lb02`:

```bash
ip -br a show enp0s8
systemctl is-active haproxy
systemctl is-active keepalived
```

Step B: monitor API reachability

Run on `cp01`:

```bash
watch -n 1 'date; nc -vz api 6443; kubectl get nodes --no-headers | wc -l'
```

Step C: force VIP failover

Run on current VIP owner LB:

```bash
sudo systemctl stop keepalived
```

Step D: confirm VIP moved

Run on both LBs:

```bash
ip -br a show enp0s8
```

Step E: restore service

Run on LB where keepalived was stopped:

```bash
sudo systemctl start keepalived
```

Pass:

- API checks continue to work (or recover within a few seconds)
- VIP appears on one LB at all times

---

## Scenario 3: Single Control-Plane Loss (Mandatory)

Goal:

- prove cluster remains write-capable with one control plane down

Run monitoring on `cp01`:

```bash
watch -n 2 'kubectl get nodes -o wide; echo; kubectl get pods -n kube-system -o wide | grep -E "etcd|kube-apiserver|kube-controller-manager|kube-scheduler"'
```

Simulate failure:

- power off one control-plane VM (for example `cp03`) from VirtualBox

Write test from `cp01`:

```bash
kubectl -n kube-system create configmap cp-failure-test --from-literal=ts="$(date +%s)" --dry-run=client -o yaml | kubectl apply -f -
kubectl -n kube-system delete configmap cp-failure-test
```

Pass:

- remaining control planes stay `Ready`
- write test succeeds

Recover:

- power the stopped control plane back on and wait until `Ready`

---

## Scenario 4: Quorum Boundary (Optional, High-Risk)

Goal:

- demonstrate what happens when etcd quorum is lost

Warning:

- this is disruptive by design
- run only in lab

Method:

1. stop two control-plane VMs
2. run write operation from remaining control plane

Expected:

- reads may partially work
- writes fail or hang until quorum is restored

Recovery:

1. bring back at least one stopped control plane
2. wait for 2/3 control planes healthy
3. rerun write test

---

## Scenario 5: Worker Maintenance (Mandatory)

Goal:

- verify controlled maintenance with `cordon` and `drain`

Run on any control-plane VM:

```bash
kubectl get nodes -o wide
kubectl cordon wk01
kubectl drain wk01 --ignore-daemonsets --delete-emptydir-data --force
kubectl get pods -A -o wide
kubectl uncordon wk01
```

Pass:

- node transitions to `SchedulingDisabled` during maintenance
- regular workload pods reschedule to other worker
- node returns to normal after `uncordon`

---

## Scenario 6: Hard Worker Loss (Mandatory)

Goal:

- validate scheduler recovery from sudden worker outage

Step A: watch workload placement

Run on any control-plane VM:

```bash
watch -n 1 'kubectl get nodes -o wide; echo; kubectl get pods -A -o wide'
```

Step B: abruptly power off one worker VM from VirtualBox (for example `wk02`)

Pass:

- node transitions to `NotReady`
- replicated workloads move to remaining worker(s)

Recovery:

- power on worker and wait for `Ready`

---

## Scenario 7: Stateless Resilience Validation (Gatus)

Goal:

- validate fast failover behavior for stateless workloads

Run on any control-plane VM:

```bash
kubectl apply -f demos/gatus-stateless/gatus-stateless-manifest.yaml
kubectl -n gatus-demo rollout status deployment/gatus --timeout=300s
kubectl -n gatus-demo rollout status deployment/demo-web --timeout=300s
kubectl -n gatus-demo get pods -o wide
kubectl -n gatus-demo get svc
```

Self-heal test:

```bash
POD=$(kubectl -n gatus-demo get pod -l app=demo-web -o jsonpath='{.items[0].metadata.name}')
kubectl -n gatus-demo delete pod "$POD"
kubectl -n gatus-demo get pods -o wide
```

Pass:

- replacement pod is created quickly
- service endpoint stays available

Full guide:

- [demos/gatus-stateless/gatus-stateless-runbook.md](./demos/gatus-stateless/gatus-stateless-runbook.md)

---

## Scenario 8: Stateful Behavior Validation (Uptime Kuma)

Goal:

- validate PVC-backed workload behavior and recovery

Step A: confirm storage class

Run on any control-plane VM:

```bash
kubectl get storageclass
```

If no default class exists, configure one before continuing.

Step B: deploy

```bash
kubectl apply -f demos/uptime-kuma-stateful/uptime-kuma-stateful-manifest.yaml
kubectl -n uptime-kuma get pvc,pods -o wide
kubectl -n uptime-kuma rollout status deployment/uptime-kuma --timeout=300s
kubectl -n uptime-kuma rollout status deployment/demo-web --timeout=300s
```

Step C: restart stateful pod

```bash
POD=$(kubectl -n uptime-kuma get pod -l app=uptime-kuma -o jsonpath='{.items[0].metadata.name}')
kubectl -n uptime-kuma delete pod "$POD"
kubectl -n uptime-kuma rollout status deployment/uptime-kuma --timeout=300s
kubectl -n uptime-kuma get pvc,pods -o wide
```

Pass:

- PVC is `Bound`
- workload returns to `Running`
- demo-web remains available during pod restart

Full guide:

- [demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md](./demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md)

---

## Scenario 9: Service DNS Scope Validation (Recommended)

Goal:

- verify cluster DNS behavior correctly

Run on any control-plane VM:

```bash
kubectl run -n uptime-kuma dns-curl --image=curlimages/curl:8.10.1 --restart=Never -it --rm -- sh -c 'curl -fsS http://demo-web.uptime-kuma.svc.cluster.local | head'
```

Expected:

- service resolves and responds from inside cluster

Important:

- `*.svc.cluster.local` names are for in-cluster DNS
- host VM shell usually cannot resolve them directly

---

## Scenario 10: Node Identity and Mapping Validation (Recommended)

Goal:

- ensure VM labels, hostnames, and Kubernetes node identity match

Run on each worker VM:

```bash
hostname
ip -br a
```

Run on control-plane VM:

```bash
kubectl get nodes -o wide
```

Pass:

- node name and InternalIP match expected worker identity

---

## Scenario 11: Join Path Validation (Recommended)

Goal:

- confirm new joins use HA endpoint, not single-node endpoint

Run on control-plane VM:

```bash
sudo kubeadm token create --print-join-command
```

Pass:

- generated join command targets `api:6443`

If output uses `10.10.10.12:6443`, re-check endpoint migration runbook:

- [api-endpoint-migration-runbook.md](./api-endpoint-migration-runbook.md)

---

## Evidence Capture Template

Use this after each scenario:

```bash
mkdir -p ~/validation-evidence
TS=$(date +%Y%m%d-%H%M%S)
kubectl get nodes -o wide > ~/validation-evidence/${TS}-nodes.txt
kubectl get pods -A -o wide > ~/validation-evidence/${TS}-pods.txt
kubectl get events -A --sort-by=.lastTimestamp > ~/validation-evidence/${TS}-events.txt
```

Record in your notes:

- scenario name
- start/end time
- pass/fail
- remediation used (if failed)

---

## Completion Criteria

Validation is considered complete when:

- mandatory scenarios pass:
  - Scenario 0
  - Scenario 1
  - Scenario 2
  - Scenario 3
  - Scenario 5
  - Scenario 6
  - Scenario 7
  - Scenario 8
- no unresolved critical failures remain
- evidence files captured for final report
