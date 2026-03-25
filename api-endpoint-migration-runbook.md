# API Endpoint Migration Runbook (Name-Based)

Use this after `cp01`, `cp02`, and `cp03` are healthy.

When to use this document:

- you already have a running kubeadm control plane
- you want to migrate API access to a stable name (`api`) fronted by LB VIP
- you want future joins and kubeconfigs decoupled from single-node IPs

Goal:

- Keep VIP as network address: `10.10.10.9`
- Use simple API name everywhere: `api`
- Move Kubernetes endpoint usage from `10.10.10.12:6443` to `api:6443`

## Fast Path For Fresh Builds From This Repo

If you built control planes using the current `build-guide.md`, API SANs may already include `api` and `10.10.10.9`.

Check on `cp01`, `cp02`, and `cp03`:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 "Subject Alternative Name"
```

If SAN already includes both `DNS:api` and `IP Address:10.10.10.9` on all three control planes:

- run Phase 1 (keepalived)
- run Phase 2 (hosts mapping)
- run Phase 3 (kubeadm endpoint update)
- skip Phase 4 (cert regeneration)
- run Phase 5 (kubeconfig switch)
- confirm Phase 6 behavior for future joins

If SAN is missing `api` or `10.10.10.9` on any control plane, use full flow including Phase 4.

## 60-second triage (if cutover seems broken)

Run on `cp01`:

```bash
getent hosts api
nc -vz api 6443
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep controlPlaneEndpoint
```

Healthy target:

- `api` resolves to `10.10.10.9`
- TCP `6443` is reachable
- control planes are healthy enough for quorum
- kubeadm config shows `controlPlaneEndpoint: api:6443`

## Naming and VIP

- API name: `api`
- API VIP: `10.10.10.9`

All kube configs and join configs should use the name `api`, not the VIP IP.

Why name-based endpoint helps:

- if LB/VIP IP changes later, update name mapping instead of rewriting every join/kubeconfig
- reduces tight coupling to one control-plane IP
- makes HA migration and operations cleaner

---

## Files Touched Per VM

`lb01`:

- `/etc/keepalived/keepalived.conf`
- `/etc/hosts`

`lb02`:

- `/etc/keepalived/keepalived.conf`
- `/etc/hosts`

`cp01`:

- `/etc/hosts`
- `/root/kubeadm-lb-migration.yaml`
- `/root/kubeadm-lb-cp01.yaml`
- `/etc/kubernetes/pki/apiserver.crt` (regenerated)
- `/etc/kubernetes/pki/apiserver.key` (regenerated)
- `/etc/kubernetes/manifests/kube-apiserver.yaml` (move out/in to restart static pod)
- `~/.kube/config`
- optional: `/etc/kubernetes/admin.conf`

`cp02`:

- `/etc/hosts`
- `/root/kubeadm-lb-cp02.yaml`
- `/etc/kubernetes/pki/apiserver.crt` (regenerated)
- `/etc/kubernetes/pki/apiserver.key` (regenerated)
- `/etc/kubernetes/manifests/kube-apiserver.yaml` (move out/in)
- `~/.kube/config`
- optional: `/etc/kubernetes/admin.conf`

`cp03`:

- `/etc/hosts`
- `/root/kubeadm-lb-cp03.yaml`
- `/etc/kubernetes/pki/apiserver.crt` (regenerated)
- `/etc/kubernetes/pki/apiserver.key` (regenerated)
- `/etc/kubernetes/manifests/kube-apiserver.yaml` (move out/in)
- `~/.kube/config`
- optional: `/etc/kubernetes/admin.conf`

`wk01`:

- `/etc/hosts`

`wk02`:

- `/etc/hosts`

---

## Phase 0: Preconditions

Run on `cp01`:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

Continue only if cluster is healthy.

Check VIP is free:

```bash
ping -c 2 10.10.10.9 || true
```

---

## Phase 1: Keepalived On `lb01` and `lb02`

Before configuring keepalived, confirm LB LAN interface name (examples use `enp0s8`):

```bash
ip -br a
```

If interface differs, replace `enp0s8` in configs below.

### `lb01`

```bash
sudo apt-get update
sudo apt-get install -y keepalived

cat <<'EOF' | sudo tee /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
  script "pidof haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_K8S_API {
  state MASTER
  interface enp0s8
  virtual_router_id 51
  priority 201
  advert_int 1
  authentication {
    auth_type PASS
    # Use a strong unique value for your environment.
    auth_pass k8sApiVip
  }
  unicast_src_ip 10.10.10.10
  unicast_peer {
    10.10.10.11
  }
  virtual_ipaddress {
    10.10.10.9/24 dev enp0s8
  }
  track_script {
    chk_haproxy
  }
}
EOF

sudo systemctl enable keepalived
sudo systemctl restart keepalived
```

### `lb02`

```bash
sudo apt-get update
sudo apt-get install -y keepalived

cat <<'EOF' | sudo tee /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
  script "pidof haproxy"
  interval 2
  weight 2
}

vrrp_instance VI_K8S_API {
  state BACKUP
  interface enp0s8
  virtual_router_id 51
  priority 200
  advert_int 1
  authentication {
    auth_type PASS
    # Use a strong unique value for your environment.
    auth_pass k8sApiVip
  }
  unicast_src_ip 10.10.10.11
  unicast_peer {
    10.10.10.10
  }
  virtual_ipaddress {
    10.10.10.9/24 dev enp0s8
  }
  track_script {
    chk_haproxy
  }
}
EOF

sudo systemctl enable keepalived
sudo systemctl restart keepalived
```

Verify on both LBs:

```bash
ip -br a show enp0s8
systemctl is-active keepalived
systemctl is-active haproxy
```

Expected:

- VIP `10.10.10.9` is present on one LB.
- Both services are `active`.

---

## Phase 2: Add `api` Name Mapping On All Nodes

Run on `lb01`, `lb02`, `cp01`, `cp02`, `cp03`, `wk01`, and `wk02`:

```bash
grep -qE '^[[:space:]]*10\.10\.10\.9[[:space:]]+api([[:space:]]|$)' /etc/hosts || echo '10.10.10.9 api' | sudo tee -a /etc/hosts
```

Verify from `cp01`:

```bash
getent hosts api
nc -vz api 6443
```

Expected:

- `api` resolves to `10.10.10.9`
- TCP connect on `6443` succeeds

---

## Phase 3: Update kubeadm Cluster Endpoint To Name

On `cp01`:

```bash
cat <<'EOF' | sudo tee /root/kubeadm-lb-migration.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: "api:6443"
apiServer:
  certSANs:
    - "api"
    - "10.10.10.9"
    - "10.10.10.12"
    - "10.0.0.32"
    - "cp01"
    - "cp02"
    - "cp03"
networking:
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
EOF
```

---

## Phase 4: Renew API Server Certs On Control Planes

### 4.1 Node-specific config files

On `cp01`:

```bash
cat <<'EOF' | sudo tee /root/kubeadm-lb-cp01.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.12
  bindPort: 6443
nodeRegistration:
  name: cp01
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: "api:6443"
apiServer:
  certSANs:
    - "api"
    - "10.10.10.9"
    - "10.10.10.12"
    - "10.0.0.32"
    - "cp01"
    - "cp02"
    - "cp03"
EOF
```

On `cp02`:

```bash
cat <<'EOF' | sudo tee /root/kubeadm-lb-cp02.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.13
  bindPort: 6443
nodeRegistration:
  name: cp02
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: "api:6443"
apiServer:
  certSANs:
    - "api"
    - "10.10.10.9"
    - "10.10.10.13"
    - "10.0.0.33"
    - "cp01"
    - "cp02"
    - "cp03"
EOF
```

On `cp03`:

```bash
cat <<'EOF' | sudo tee /root/kubeadm-lb-cp03.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.14
  bindPort: 6443
nodeRegistration:
  name: cp03
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: "api:6443"
apiServer:
  certSANs:
    - "api"
    - "10.10.10.9"
    - "10.10.10.14"
    - "10.0.0.34"
    - "cp01"
    - "cp02"
    - "cp03"
EOF
```

### 4.2 Force-regenerate and restart apiserver static pod (guaranteed path)

Run in this order: `cp02` -> `cp03` -> `cp01`.

On `cp02`:

```bash
TS=$(date +%Y%m%d-%H%M%S)
sudo cp /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.bak-$TS
sudo cp /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.bak-$TS
sudo rm -f /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.key
sudo kubeadm init phase certs apiserver --config /root/kubeadm-lb-cp02.yaml
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml
sleep 20
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
```

On `cp03`:

```bash
TS=$(date +%Y%m%d-%H%M%S)
sudo cp /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.bak-$TS
sudo cp /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.bak-$TS
sudo rm -f /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.key
sudo kubeadm init phase certs apiserver --config /root/kubeadm-lb-cp03.yaml
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml
sleep 20
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
```

On `cp01`:

```bash
TS=$(date +%Y%m%d-%H%M%S)
sudo cp /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.bak-$TS
sudo cp /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.bak-$TS
sudo rm -f /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.key
sudo kubeadm init phase certs apiserver --config /root/kubeadm-lb-cp01.yaml
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml
sleep 20
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
```

Verify SAN on each CP:

```bash
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 "Subject Alternative Name"
```

Expected SAN includes `DNS:api` and `IP Address:10.10.10.9`.

### 4.3 Upload updated kubeadm config

On `cp01`:

```bash
sudo kubeadm init phase upload-config kubeadm --config /root/kubeadm-lb-migration.yaml
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep controlPlaneEndpoint
```

Expected output includes `controlPlaneEndpoint: api:6443`.

---

## Phase 5: Switch kubeconfig To Name

On each control plane user account that uses `kubectl`:

```bash
kubectl config set-cluster kubernetes --server=https://api:6443
```

Optional (root kubeconfig on each CP):

```bash
sudo sed -i 's#https://10.10.10.12:6443#https://api:6443#g' /etc/kubernetes/admin.conf
```

Verify from `cp01`:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

---

## Phase 6: Future Join Endpoint

From now on, new join configs should use:

- `apiServerEndpoint: api:6443`

Do not use `10.10.10.12:6443` in future join files.

---

## Quick Rollback

Temporary fallback for your current kubeconfig:

```bash
kubectl config set-cluster kubernetes --server=https://10.10.10.12:6443
```

---

## Common Issue: `kubeadm token create` still tries `10.10.10.12:6443`

Symptom:

```text
error: ... Post "https://10.10.10.12:6443/...": context deadline exceeded
error: ... dial tcp 10.10.10.12:6443: connect: connection refused
```

Why it happens:

- `kubeadm token create` uses `/etc/kubernetes/admin.conf` by default.
- If that file still points to `10.10.10.12:6443`, token creation depends on local cp01 API.
- During/after cutover, local cp01 API may be restarting or temporarily unavailable.

Fix:

1. Point root kubeconfig to name endpoint:

```bash
sudo sed -i 's#https://10.10.10.12:6443#https://api:6443#g' /etc/kubernetes/admin.conf
```

2. Retry token creation:

```bash
sudo kubeadm token create --print-join-command
```

3. If still failing, force kubeconfig explicitly:

```bash
sudo kubeadm token create --print-join-command --kubeconfig /home/cp01/.kube/config
```

4. If command still hangs/fails, check cp01 local API/kubelet health:

```bash
kubectl get nodes -o wide
sudo systemctl is-active kubelet
sudo crictl ps -a | grep kube-apiserver
sudo journalctl -u kubelet -n 120 --no-pager
```
