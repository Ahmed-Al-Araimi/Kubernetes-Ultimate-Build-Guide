# Kubernetes HA Lab FAQ (Operator Edition)

This FAQ supports the build in this repo with practical, reusable troubleshooting.

It is based on real failures during testing, but written in a general way so it stays useful across repeated builds.

---

## 1) First Principles

- Follow the runbook order exactly.
- Do not continue past a failed validation step.
- Prefer one command at a time unless a full block is explicitly shown.
- Keep package install commands on one line when the guide says so.

---

## 2) Where Should I Run Commands?

- Run cluster admin commands (`kubectl`, test operations) on any healthy control-plane node with working kubeconfig.
- Run node prep commands on the node being prepared.
- Run load balancer and VIP checks on LB nodes and from one control-plane node.
- Run resilience tests from any control-plane node.

---

## 3) Quick Cluster Health Snapshot

If something feels wrong, run this first on a control-plane node:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 120
kubectl config view --minify | grep server
kubectl get storageclass
kubectl get pvc -A
```

Interpretation:

- `NotReady` nodes usually indicate node runtime, network, or kubelet issues.
- `Pending` pods usually indicate scheduling, storage, or image pull issues.
- API endpoint mismatch in kubeconfig causes random admin command failures.

---

## 4) Package Install and Repo Problems

### Why do Kubernetes packages fail to install on some nodes?

Most failures come from repo trust setup, not from `apt install` itself.

Use this sequence:

1. Reset Kubernetes repo/key files.
2. Download and validate the signing key file.
3. Configure repo with `signed-by` keyring.
4. Run `apt-get update` and confirm no signature errors.
5. Install packages.

If packages were previously held and install fails, unhold, install, then hold again.

### Why do I see unexpected shell errors during install?

This is usually paste formatting:

- command wrapped across lines incorrectly
- copied only part of a loop block
- options split into separate commands

Rule: for critical install lines, paste exactly as one line.

### Why does `kubelet.service` not exist?

That typically means packages did not install successfully yet. Fix repo/install first, then enable kubelet.

---

## 5) API Endpoint, LB, and Control Plane Behavior

### When should I use `api:6443`?

Use a stable control-plane endpoint (VIP or DNS) for operational consistency.

- kubeconfig should point to the stable endpoint.
- joins and routine admin operations should use the stable endpoint.

### Why can TCP to `api:6443` work while kubeadm/kubectl writes fail?

Because TCP reachability only proves port access.

Write operations still require:

- healthy API backend
- etcd quorum for writes
- correct kubeconfig endpoint

### Do all control planes need to be on?

In a 3-control-plane stacked-etcd setup, normal writes generally require quorum (2 of 3 healthy members).

Running only one control plane can cause write failures or unstable behavior.

### What if `api` resolves but is not reachable?

That usually means LB/VIP path is down.

Check:

- HAProxy/keepalived service status on LB nodes
- backend control-plane reachability
- host routing and `/etc/hosts` consistency

---

## 6) CNI and Calico Bring-Up

### Why does Calico custom resource apply fail early?

Custom resources can fail if operator CRDs are not present or not ready yet.

Correct order:

1. Apply CRDs/operator manifests.
2. Wait for CRDs to be available.
3. Apply Calico custom resources.

### Are apply annotation warnings always fatal?

No. Some apply warnings are informational.

Treat as non-blocking if:

- resources exist
- controllers reconcile
- pods become healthy

Treat as blocking if:

- resources remain missing
- operators fail to start
- cluster network pods stay unhealthy

---

## 7) Storage and Stateful Workloads

### Is storage required for every Kubernetes build?

No.

- Stateless demos can run without dynamic storage.
- Stateful workloads need a storage strategy (dynamic provisioner or pre-created PVs).

### Why do stateful pods stay `Pending`?

Most common reason: PVC cannot bind (no default StorageClass, no matching PV, or no healthy provisioner).

Check:

```bash
kubectl get storageclass
kubectl get pvc -A
kubectl describe pvc -n <namespace> <pvc-name>
```

### Why can PVC be `Bound` but pod still not ready?

Volume and scheduling timing can lag. If a pod remains stuck after PVC is healthy, recreating that pod once is usually enough.

---

## 8) Worker Naming and Identity

### Cluster node names look swapped with VM labels. What should I trust?

Trust Kubernetes node name + InternalIP and verify inside each VM:

```bash
hostname
ip -br a
kubectl get nodes -o wide
```

If names/IPs are wrong due to join config, reset and rejoin the affected worker with corrected values.

---

## 9) Resilience Tests (What They Prove)

### What does `cordon` do?

Marks a node unschedulable for new pods.

### What does `drain` do?

Evicts workload pods from a node so they reschedule elsewhere.

### What does `uncordon` do?

Makes the node schedulable again.

Suggested live demo sequence:

1. Watch pod placement (`kubectl get pods -n <ns> -o wide -w`).
2. `cordon` one worker.
3. `drain` the same worker.
4. Observe pod rescheduling.
5. `uncordon` and confirm recovery.

---

## 10) Most Common Operator Mistakes

- Skipping validation checks because previous steps "looked fine".
- Mixing command scopes (running node prep or LB commands on wrong VM).
- Pasting multiline commands with broken formatting.
- Continuing after first failure instead of recovering from that step.
- Testing stateful apps before storage is prepared.

---

## 11) Authoritative References

- kubeadm install: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- kubeadm join: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/
- kubeadm HA: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- kubeadm token create: https://kubernetes.io/docs/reference/setup-tools/kubeadm/generated/kubeadm_token/kubeadm_token_create/
- kubeadm config: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/
- declarative apply: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/
- StorageClass: https://kubernetes.io/docs/concepts/storage/storage-classes/
- change default StorageClass: https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
- etcd quorum FAQ: https://etcd.io/docs/v3.2/faq/
- local-path provisioner: https://github.com/rancher/local-path-provisioner
- Calico Installation API: https://docs.tigera.io/calico/latest/reference/installation/api

---

## 12) Secondary Community Pattern Checks

Used only as supplemental context, not as primary authority:

- https://stackoverflow.com/questions/55780083/error-no-persistent-volumes-available-for-this-claim-and-no-storage-class-is-se
- https://stackoverflow.com/questions/60317567/cluster-doesnt-have-a-stable-controlplaneendpoint-address
- https://www.reddit.com/r/kubernetes/comments/wabkq5/default_grafana_k8s_app_pv_issue_failedbinding/
- https://www.reddit.com/r/kubernetes/comments/nc5zdg/eli5_the_missing_annotation_will_be_patched/
