# Kubernetes Ultimate Build Guide (VM by VM)

<img src="./pictures/k8s%20topology.png" alt="Kubernetes HA Lab Topology" width="400">

This runbook builds the Kubernetes base layer of the lab from zero, using VirtualBox VMs and `kubeadm`.

This guide is intentionally practical:

- it is based on tested execution and recovery, not only install theory
- it documents what to do when real failures happen mid-build
- it is designed to be reusable by operators, not only lab authors

## Tested Environment

Primary validated environment for this guide:

- Host OS: `Windows 11 Pro`
- Virtualization: `VirtualBox 7.2.6 r172322 (Qt6.8.0 on Windows)`
- Server image: `ubuntu-22.04.5-live-server-amd64`

You can still adapt to nearby versions, but exact validation was done with the stack above.

## Stateful vs Stateless (Read First)

Kubernetes runs both workload types, but HA behavior is different.

### Stateless workloads
- app instances are interchangeable
- no critical data stored on pod-local disk
- scale horizontally with multiple replicas
- fastest failover and easiest rolling updates

### Stateful workloads
- app depends on durable data and often stable identity
- usually needs PVC-backed storage and ordered recovery behavior
- failover depends on storage attach/mount, quorum, and app startup time

Examples:

- Stateless: `nginx`, `traefik/whoami`, most web frontends, many API gateways, many alert/webhook processors.
- Stateful: PostgreSQL, MySQL, Redis (persistent mode), OpenSearch/Elasticsearch, Kafka, MinIO, Uptime Kuma with SQLite PVC, most SIEM data/index layers.

## HA Expectations and Production Reality

What a production-style Kubernetes environment typically includes:

- multiple control planes with stable API VIP
- multiple worker nodes with clear scheduling policy
- ingress layer for host/path routing (not per-app manual NodePort sprawl)
- replicated storage layer for stateful apps
- backup/restore process for persistent data
- monitoring, alerting, audit logging, and upgrade runbooks

What this build can do well:

- control-plane API high availability
- stateless app continuity during worker maintenance/failure when deployed with multi-replica + spread rules
- rolling changes with low service interruption
- stable access via VIP/domain instead of worker IPs

What this build does not guarantee by default:

- instant failover for every app
- zero downtime for single-replica stateful apps
- safe stateful recovery without proper storage design and backups
- automatic per-app traffic routing unless you add an ingress/reverse-proxy pattern

## Build Rules

- Build in this exact order: `lb01`, `lb02`, `cp01`, `cp02`, `cp03`, `wk01`, `wk02`.
- This runbook is two-phase by design.
- Phase 1: bootstrap control planes using temporary endpoint `10.10.10.12:6443`.
- Phase 2: mandatory cutover to stable endpoint `api:6443` (VIP) before any worker joins.
- Do not jump ahead if a validation step fails.
- Run commands on the VM named in the section header.
- If you already set hostname/static networking during OS install, skip only those specific duplicate commands.

## Paste Safety (Important)

- Paste one command at a time unless a full fenced block is explicitly provided.
- Do not paste partial lines from wrapped commands.
- If any command fails, stop and fix that command before continuing.
- For critical package install lines, use the exact single-line command from this guide.

## Address Plan

- `lb01`: WAN `10.0.0.30`, LAN `10.10.10.10`
- `lb02`: WAN `10.0.0.31`, LAN `10.10.10.11`
- `cp01`: WAN `10.0.0.32`, LAN `10.10.10.12`
- `cp02`: WAN `10.0.0.33`, LAN `10.10.10.13`
- `cp03`: WAN `10.0.0.34`, LAN `10.10.10.14`
- `wk01`: WAN `10.0.0.35`, LAN `10.10.10.15`
- `wk02`: WAN `10.0.0.36`, LAN `10.10.10.16`

## Kubernetes Baseline

- Packages track: `v1.35`
- Cluster version: `v1.35.3`
- CNI: Calico `v3.31.4`
- Pod CIDR: `192.168.0.0/16`
- Service CIDR: `10.96.0.0/12`

## VirtualBox VM Baseline

Use for every VM:

- Ubuntu Server `22.04`
- Adapter 1: Bridged (`10.0.0.x`)
- Adapter 2: Host-only/Internal (`10.10.10.x`)
- OpenSSH server enabled
- No optional stack features (do not preinstall MicroK8s, LXD, etc.)

## Shared Kubernetes Node Prep Block

Run this block on Kubernetes nodes only:

- `cp01`
- `cp02`
- `cp03`
- `wk01`
- `wk02`

Purpose:

- disable swap
- prepare kernel modules and sysctl
- configure `containerd` for systemd cgroups
- install/hold Kubernetes packages

```bash
sudo apt-get update
sudo apt-get install -y curl jq netcat-openbsd containerd apt-transport-https ca-certificates gpg

sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
if command -v ufw >/dev/null 2>&1; then sudo ufw disable || true; fi
sudo timedatectl set-ntp true

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable containerd
sudo systemctl restart containerd

# Kubernetes apt repo setup (network-sensitive step)
# If this section fails, rerun from "1) Reset Kubernetes repo state".
# 1) Reset Kubernetes repo state
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /tmp/kubernetes-release.key
sudo mkdir -p -m 755 /etc/apt/keyrings

# 2) Download signing key (with retries)
curl --retry 20 --retry-all-errors --retry-delay 2 --connect-timeout 10 --max-time 180 -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key -o /tmp/kubernetes-release.key
test -s /tmp/kubernetes-release.key || { echo "FATAL: Release.key download failed, do not continue."; exit 1; }

# 3) Import key and configure Kubernetes apt repo
sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg /tmp/kubernetes-release.key
rm -f /tmp/kubernetes-release.key
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null

# 4) Refresh apt index
sudo apt-get update

# 5) Clear stale package holds from previous failed attempts (safe on first run)
sudo apt-mark unhold kubelet kubeadm kubectl 2>/dev/null || true

# 6) Install Kubernetes node packages (keep this as ONE line when pasting)
sudo apt-get -o Acquire::Retries=10 -o Acquire::https::Timeout=30 -o Acquire::http::Timeout=30 install -y --allow-change-held-packages kubelet kubeadm kubectl

# 7) Validate install and lock package versions
dpkg -s kubelet kubeadm kubectl >/dev/null 2>&1 || { echo "FATAL: Kubernetes packages not installed."; exit 1; }
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

Optional verification:

```bash
ip -br a
swapon --show
kubeadm version -o short
grep -n "SystemdCgroup" /etc/containerd/config.toml
systemctl is-active containerd
systemctl is-enabled kubelet
```

Expected:

- `swapon --show` returns no active swap lines
- `SystemdCgroup = true` appears in containerd config
- `containerd` is active
- kubeadm reports `v1.35.x`

---

## VM 1: `lb01`

Purpose: create API load-balancer node #1.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname lb01
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.30/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.10/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

Optional verification:

```bash
hostname
ip -br a
ip route
```

### 2) Install and configure HAProxy

```bash
sudo apt-get update
sudo apt-get install -y haproxy

cat <<'EOF' | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 10s
    timeout client  1m
    timeout server  1m

frontend kubernetes_api_frontend
    bind *:6443
    default_backend kubernetes_control_planes

backend kubernetes_control_planes
    option tcp-check
    balance roundrobin
    server cp01 10.10.10.12:6443 check
    server cp02 10.10.10.13:6443 check
    server cp03 10.10.10.14:6443 check
EOF

sudo systemctl enable haproxy
sudo systemctl restart haproxy
```

Optional verification:

```bash
systemctl is-active haproxy
ss -lntp | grep 6443
sudo tail -n 30 /var/log/syslog | grep -i haproxy || true
```

---

## VM 2: `lb02`

Purpose: create API load-balancer node #2.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname lb02
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.31/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.11/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Install and configure HAProxy

```bash
sudo apt-get update
sudo apt-get install -y haproxy

cat <<'EOF' | sudo tee /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 10s
    timeout client  1m
    timeout server  1m

frontend kubernetes_api_frontend
    bind *:6443
    default_backend kubernetes_control_planes

backend kubernetes_control_planes
    option tcp-check
    balance roundrobin
    server cp01 10.10.10.12:6443 check
    server cp02 10.10.10.13:6443 check
    server cp03 10.10.10.14:6443 check
EOF

sudo systemctl enable haproxy
sudo systemctl restart haproxy
```

Optional verification:

```bash
hostname
ip -br a
systemctl is-active haproxy
ss -lntp | grep 6443
```

---

## VM 3: `cp01` (initial control plane bootstrap)

Purpose: bootstrap the cluster control plane and install Calico.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname cp01
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.32/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.12/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Host resolution

```bash
cat <<'EOF' | sudo tee /etc/hosts
127.0.0.1 localhost
127.0.1.1 cp01
10.10.10.10 lb01
10.10.10.11 lb02
10.10.10.9 api
10.10.10.12 cp01
10.10.10.13 cp02
10.10.10.14 cp03
10.10.10.15 wk01
10.10.10.16 wk02
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

### 3) Shared Kubernetes node prep

Run the full "Shared Kubernetes Node Prep Block" on `cp01`.

### 4) Create kubeadm init config

Important:

- `controlPlaneEndpoint: "10.10.10.12:6443"` below is temporary for bootstrap only.
- You will cut over to stable `api:6443` after `cp02` and `cp03` join.

```bash
cat <<'EOF' | sudo tee /root/kubeadm-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.12
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  name: cp01
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.12
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.3
controlPlaneEndpoint: "10.10.10.12:6443"
networking:
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
apiServer:
  certSANs:
    - "api"
    - "10.10.10.9"
    - "10.10.10.12"
    - "10.0.0.32"
    - "cp01"
    - "cp02"
    - "cp03"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

Optional verification:

```bash
sudo kubeadm config validate --config /root/kubeadm-init.yaml
```

### 5) Bootstrap cluster on cp01

```bash
sudo kubeadm config images pull --config /root/kubeadm-init.yaml
sudo kubeadm init --config /root/kubeadm-init.yaml

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
```

Post-init verification:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

### 6) Install Calico (CRDs first, then operator, then custom resources)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/operator-crds.yaml
kubectl wait --for=condition=Established --timeout=120s crd/installations.operator.tigera.io
kubectl wait --for=condition=Established --timeout=120s crd/apiservers.operator.tigera.io

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml

cat <<'EOF' > /tmp/calico-custom-resources.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - name: default-ipv4-ippool
        cidr: 192.168.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl apply -f /tmp/calico-custom-resources.yaml
```

Readiness watch:

```bash
watch -n 2 'kubectl get nodes -o wide; echo; kubectl get pods -n kube-system -o wide; echo; kubectl get pods -n calico-system -o wide; echo; kubectl get pods -n tigera-operator -o wide'
```

Gate to continue:

- Continue only after `cp01` is `Ready`.
- Continue only after Calico and kube-system pods are healthy.

---

## VM 4: `cp02` (control plane join)

Purpose: add second control-plane node.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname cp02
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.33/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.13/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Host resolution

```bash
cat <<'EOF' | sudo tee /etc/hosts
127.0.0.1 localhost
127.0.1.1 cp02
10.10.10.10 lb01
10.10.10.11 lb02
10.10.10.9 api
10.10.10.12 cp01
10.10.10.13 cp02
10.10.10.14 cp03
10.10.10.15 wk01
10.10.10.16 wk02
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

### 3) Shared Kubernetes node prep

Run the full "Shared Kubernetes Node Prep Block" on `cp02`.

### 4) Pre-join cleanliness check

```bash
if [ -d /etc/kubernetes/manifests ]; then sudo ls -l /etc/kubernetes/manifests; else echo "/etc/kubernetes/manifests not present yet (expected on fresh node)"; fi
if [ -d /var/lib/etcd ]; then sudo ls -l /var/lib/etcd; else echo "/var/lib/etcd not present yet (expected before control-plane join)"; fi
```

### 5) On cp01: generate fresh join values right before cp02 join

```bash
JOIN_CMD="$(sudo kubeadm token create --print-join-command)"
CERT_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
echo "$JOIN_CMD"
echo "CERT_KEY=$CERT_KEY"
```

From printed join command, copy:

- token value after `--token`
- hash value after `--discovery-token-ca-cert-hash`

### 6) On cp02: create join config and join

```bash
cat <<'EOF' > ~/kubeadm-join-cp02.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: REPLACE_TOKEN
    apiServerEndpoint: 10.10.10.12:6443
    caCertHashes:
      - "REPLACE_HASH"
controlPlane:
  certificateKey: REPLACE_CERT_KEY
  localAPIEndpoint:
    advertiseAddress: 10.10.10.13
    bindPort: 6443
nodeRegistration:
  name: cp02
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.13
EOF

sudo kubeadm join --config ~/kubeadm-join-cp02.yaml
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
```

Optional verification from cp01:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get pods -n calico-system -o wide
```

---

## VM 5: `cp03` (control plane join)

Purpose: add third control-plane node.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname cp03
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.34/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.14/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Host resolution

```bash
cat <<'EOF' | sudo tee /etc/hosts
127.0.0.1 localhost
127.0.1.1 cp03
10.10.10.10 lb01
10.10.10.11 lb02
10.10.10.9 api
10.10.10.12 cp01
10.10.10.13 cp02
10.10.10.14 cp03
10.10.10.15 wk01
10.10.10.16 wk02
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

### 3) Shared Kubernetes node prep

Run the full "Shared Kubernetes Node Prep Block" on `cp03`.

### 4) On cp01: regenerate fresh join values immediately before cp03 join

```bash
JOIN_CMD="$(sudo kubeadm token create --print-join-command)"
CERT_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
echo "$JOIN_CMD"
echo "CERT_KEY=$CERT_KEY"
```

### 5) On cp03: create join config and join

```bash
cat <<'EOF' > ~/kubeadm-join-cp03.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: REPLACE_TOKEN
    apiServerEndpoint: 10.10.10.12:6443
    caCertHashes:
      - "REPLACE_HASH"
controlPlane:
  certificateKey: REPLACE_CERT_KEY
  localAPIEndpoint:
    advertiseAddress: 10.10.10.14
    bindPort: 6443
nodeRegistration:
  name: cp03
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.14
EOF

sudo kubeadm join --config ~/kubeadm-join-cp03.yaml
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
```

Optional verification from cp01:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get pods -n calico-system -o wide
```

---

## Mandatory Phase 2: API VIP Cutover (Before Workers)

Purpose:

- move from temporary bootstrap endpoint (`10.10.10.12:6443`) to stable HA endpoint (`api:6443`)
- ensure worker joins and future operations are not tied to one control-plane IP

Run the full cutover procedure now:

- [API Endpoint Migration Runbook](./api-endpoint-migration-runbook.md)

Do not continue to `wk01`/`wk02` until all checks below pass on `cp01`:

```bash
getent hosts api
nc -vz api 6443
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep controlPlaneEndpoint
kubectl config view --minify | grep server
```

Expected:

- `api` resolves to VIP (`10.10.10.9`)
- TCP connect to `api:6443` succeeds
- kubeadm cluster config shows `controlPlaneEndpoint: api:6443`
- kubeconfig server shows `https://api:6443`

---

## VM 6: `wk01` (worker join)

Purpose: add worker node #1.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname wk01
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.35/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.15/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Host resolution

```bash
cat <<'EOF' | sudo tee /etc/hosts
127.0.0.1 localhost
127.0.1.1 wk01
10.10.10.10 lb01
10.10.10.11 lb02
10.10.10.9 api
10.10.10.12 cp01
10.10.10.13 cp02
10.10.10.14 cp03
10.10.10.15 wk01
10.10.10.16 wk02
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

### 3) Shared Kubernetes node prep

Run the full "Shared Kubernetes Node Prep Block" on `wk01`.

### 4) On cp01: generate fresh worker join values

```bash
JOIN_CMD="$(sudo kubeadm token create --print-join-command)"
echo "$JOIN_CMD"
```

From printed join command, copy:

- token value after `--token`
- hash value after `--discovery-token-ca-cert-hash`

### 5) On wk01: create join config and join

```bash
cat <<'EOF' | sudo tee /root/kubeadm-join-worker-wk01.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: REPLACE_TOKEN
    apiServerEndpoint: api:6443
    caCertHashes:
      - "REPLACE_HASH"
nodeRegistration:
  name: wk01
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.15
EOF

sudo kubeadm join --config /root/kubeadm-join-worker-wk01.yaml
```

Optional verification from cp01:

```bash
kubectl get nodes -o wide
kubectl get pods -n calico-system -o wide
```

---

## VM 7: `wk02` (worker join)

Purpose: add worker node #2.

### 1) Hostname and static network

```bash
sudo hostnamectl set-hostname wk02
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg >/dev/null

cat <<'EOF' | sudo tee /etc/netplan/00-k8s-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.0.0.36/24]
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses: [10.0.0.1,1.1.1.1]
    enp0s8:
      dhcp4: false
      addresses: [10.10.10.16/24]
EOF

sudo chmod 600 /etc/netplan/00-k8s-static.yaml
sudo rm -f /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan apply
```

### 2) Host resolution

```bash
cat <<'EOF' | sudo tee /etc/hosts
127.0.0.1 localhost
127.0.1.1 wk02
10.10.10.10 lb01
10.10.10.11 lb02
10.10.10.9 api
10.10.10.12 cp01
10.10.10.13 cp02
10.10.10.14 cp03
10.10.10.15 wk01
10.10.10.16 wk02
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

### 3) Shared Kubernetes node prep

Run the full "Shared Kubernetes Node Prep Block" on `wk02`.

### 4) On cp01: generate fresh worker join values

```bash
JOIN_CMD="$(sudo kubeadm token create --print-join-command)"
echo "$JOIN_CMD"
```

### 5) On wk02: create join config and join

```bash
cat <<'EOF' | sudo tee /root/kubeadm-join-worker-wk02.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    token: REPLACE_TOKEN
    apiServerEndpoint: api:6443
    caCertHashes:
      - "REPLACE_HASH"
nodeRegistration:
  name: wk02
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: node-ip
      value: 10.10.10.16
EOF

sudo kubeadm join --config /root/kubeadm-join-worker-wk02.yaml
```

Optional verification from cp01:

```bash
kubectl get nodes -o wide
kubectl get pods -n calico-system -o wide
```

---

## Final Validation (run on `cp01`)

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get pods -n calico-system -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -n 100
```

Expected:

- all 5 Kubernetes nodes (`cp01`, `cp02`, `cp03`, `wk01`, `wk02`) are `Ready`
- control-plane pods are healthy on `cp01`, `cp02`, `cp03`
- Calico pods are healthy across all nodes

---

## Next Steps

- [Situation-Based Validation Playbook](./validation-playbook.md)
- [Key Commands](./key-commands.md)
- [Shortcut Installer Script](./scripts/install-k8s-shortcuts.sh)
- [Scripts Notes](./scripts/README.md)
- [Gatus Demo (Stateless Fast-Failover)](./demos/gatus-stateless/gatus-stateless-runbook.md)
- [Uptime Kuma Demo](./demos/uptime-kuma-stateful/uptime-kuma-stateful-runbook.md)
- [API Endpoint Migration Runbook](./api-endpoint-migration-runbook.md) (use for recovery or revalidation)
- [FAQ](./faq.md)
- [IP Index](./ip-index.md)


