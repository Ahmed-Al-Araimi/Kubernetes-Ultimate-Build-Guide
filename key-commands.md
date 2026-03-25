# Key Commands (Daily Operations)

Use this as the quick command reference for cluster operations.

Command scope:

- run `kubectl` commands on any healthy control-plane VM (`cp01`, `cp02`, `cp03`)
- run node power/restart actions on the target VM or VirtualBox host

---

## 1) Health Snapshot

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 120
```

---

## 2) API Endpoint Checks

```bash
getent hosts api
nc -vz api 6443
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep controlPlaneEndpoint
kubectl config view --minify | grep server
```

Expected:

- `api` resolves to `10.10.10.9`
- endpoint target is `api:6443` in kubeadm config and kubeconfig

---

## 3) Node Operations

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
kubectl uncordon <node>
kubectl get nodes -o wide
```

---

## 4) Join and Token Operations

Worker join command:

```bash
sudo kubeadm token create --print-join-command
```

Control-plane cert key:

```bash
sudo kubeadm init phase upload-certs --upload-certs | tail -n 1
```

---

## 5) System Pods and Networking

```bash
kubectl get pods -n kube-system -o wide
kubectl get pods -n calico-system -o wide
kubectl get crd | grep -E 'operator.tigera.io|projectcalico.org'
```

---

## 6) Storage Checks (Stateful Workloads)

```bash
kubectl get storageclass
kubectl get pvc -A
kubectl describe pvc -n <namespace> <pvc-name>
```

---

## 7) Demo Commands (Stateless and Stateful)

Gatus:

```bash
kubectl apply -f demos/gatus-stateless/gatus-stateless-manifest.yaml
kubectl -n gatus-demo get pods -o wide
kubectl -n gatus-demo get svc
```

Uptime Kuma:

```bash
kubectl apply -f demos/uptime-kuma-stateful/uptime-kuma-stateful-manifest.yaml
kubectl -n uptime-kuma get pvc,pods -o wide
kubectl -n uptime-kuma get svc
```

---

## 8) Quick Watch Commands

```bash
watch -n 2 'kubectl get nodes -o wide'
watch -n 2 'kubectl get pods -A -o wide'
watch -n 2 'kubectl get nodes -o wide; echo; kubectl -n uptime-kuma get pods -o wide'
watch -n 2 'kubectl get nodes -o wide; echo; kubectl -n gatus-demo get pods -o wide'
```

---

## 9) Evidence Capture

```bash
mkdir -p ~/validation-evidence
TS=$(date +%Y%m%d-%H%M%S)
kubectl get nodes -o wide > ~/validation-evidence/${TS}-nodes.txt
kubectl get pods -A -o wide > ~/validation-evidence/${TS}-pods.txt
kubectl get events -A --sort-by=.lastTimestamp > ~/validation-evidence/${TS}-events.txt
```

