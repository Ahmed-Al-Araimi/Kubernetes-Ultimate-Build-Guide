# Uptime Kuma Demo (Resilience + Continuity)

Use this after you finish the cluster build.

What this demo proves:

- a real app (`uptime-kuma`) running on your cluster
- service continuity on replica-based workload (`demo-web`)
- self-healing, rolling updates, and worker maintenance behavior

If you want a stateless fast-failover demo first, run:

- [Gatus Demo (Stateless)](../gatus-stateless/gatus-stateless-runbook.md)

Build prerequisite:

- complete [build-guide.md](../../build-guide.md)

Execution scope:

- run all `kubectl ...` commands on any control-plane node (`cp01`, `cp02`, or `cp03`)
- run browser/UI steps from your host machine

---

## 1) Storage Prerequisite (stateful apps only)

Core Kubernetes build does not require storage.
This demo does, because `uptime-kuma` uses a PVC.

Check storage classes:

```bash
kubectl get storageclass
```

If no default storage class exists, use one of these paths.

### Option A (recommended): dynamic local-path provisioner

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass
```

### Option B (fallback): static PV for this demo

```bash
cat <<'EOF' > /tmp/uptime-kuma-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: uptime-kuma-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /var/lib/uptime-kuma-data
    type: DirectoryOrCreate
EOF

kubectl apply -f /tmp/uptime-kuma-pv.yaml
kubectl get pv uptime-kuma-pv
```

---

## 2) Deploy

```bash
kubectl apply -f demos/uptime-kuma-stateful/uptime-kuma-stateful-manifest.yaml
kubectl -n uptime-kuma get pvc
kubectl -n uptime-kuma rollout status deployment/uptime-kuma --timeout=300s
kubectl -n uptime-kuma rollout status deployment/demo-web --timeout=300s
kubectl -n uptime-kuma get pods -o wide
kubectl -n uptime-kuma get svc
```

---

## 3) Open UI and Add Monitor

Open in browser:

- `http://<node-ip>:30081`

If NodePort is not reachable:

```bash
kubectl -n uptime-kuma port-forward svc/uptime-kuma 3001:3001
```

Then open:

- `http://localhost:3001`

In Uptime Kuma, create monitor:

- Type: `HTTP(s)`
- Name: `demo-web`
- URL: `http://demo-web` (or `http://demo-web.uptime-kuma.svc.cluster.local`)
- Heartbeat: `5s`

---

## 4) Prove Resilience

Watch:

```bash
watch -n 2 'kubectl get nodes -o wide; echo; kubectl get pods -n uptime-kuma -o wide; echo; kubectl get endpoints -n uptime-kuma demo-web'
```

Test A: self-heal

```bash
POD=$(kubectl -n uptime-kuma get pod -l app=demo-web -o jsonpath='{.items[0].metadata.name}')
kubectl -n uptime-kuma delete pod "$POD"
```

Test B: rolling update

```bash
kubectl -n uptime-kuma rollout restart deployment/demo-web
kubectl -n uptime-kuma rollout status deployment/demo-web
```

Test C: worker maintenance

```bash
kubectl cordon wk01
kubectl drain wk01 --ignore-daemonsets --delete-emptydir-data --force
kubectl -n uptime-kuma get pods -o wide
kubectl uncordon wk01
```

---

## 5) Cleanup (Optional)

```bash
kubectl delete namespace uptime-kuma
rm -f /tmp/uptime-kuma-pv.yaml
```

Notes:

- `uptime-kuma` is single replica in this quick lab demo.
- continuity proof is primarily the `demo-web` service staying available.

