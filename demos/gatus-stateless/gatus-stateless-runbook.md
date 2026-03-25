# Gatus Demo (Stateless Fast-Failover)

Use this after the cluster build when you want a stateless resilience demo.

Why this demo:

- no PVC dependency
- fast recovery behavior
- clear self-heal and worker-failover visibility

Build prerequisite:

- complete [build-guide.md](../../build-guide.md)

Execution scope:

- run all `kubectl ...` commands on any control-plane node (`cp01`, `cp02`, or `cp03`)
- open UI from your host browser

---

## 1) Deploy

```bash
kubectl apply -f demos/gatus-stateless/gatus-stateless-manifest.yaml
kubectl -n gatus-demo rollout status deployment/gatus --timeout=300s
kubectl -n gatus-demo rollout status deployment/demo-web --timeout=300s
kubectl -n gatus-demo get pods -o wide
kubectl -n gatus-demo get svc
```

---

## 2) Open Gatus UI

Open:

- `http://<node-ip>:30082`

If NodePort is not reachable:

```bash
kubectl -n gatus-demo port-forward svc/gatus 8080:8080
```

Then open:

- `http://localhost:8080`

Expected:

- endpoint `demo-web` shows `UP`

---

## 3) Prove Resilience

Watch:

```bash
watch -n 2 'kubectl get nodes -o wide; echo; kubectl get pods -n gatus-demo -o wide; echo; kubectl get endpoints -n gatus-demo gatus demo-web'
```

Test A: self-heal one `demo-web` pod

```bash
POD=$(kubectl -n gatus-demo get pod -l app=demo-web -o jsonpath='{.items[0].metadata.name}')
kubectl -n gatus-demo delete pod "$POD"
```

Test B: rolling update

```bash
kubectl -n gatus-demo rollout restart deployment/demo-web
kubectl -n gatus-demo rollout status deployment/demo-web
```

Test C: worker maintenance

```bash
kubectl cordon wk01
kubectl drain wk01 --ignore-daemonsets --delete-emptydir-data --force
kubectl -n gatus-demo get pods -o wide
kubectl uncordon wk01
```

---

## 4) Cleanup (Optional)

```bash
kubectl delete namespace gatus-demo
```

---

## 5) Stateless vs Stateful

- `demos/gatus-stateless/gatus-stateless-runbook.md` = stateless, fast failover style behavior.
- `demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md` = stateful PVC workload behavior.

