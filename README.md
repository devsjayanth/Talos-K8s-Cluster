# Talos-K8s-Cluster
### Complete Talos Kubernetes Cluster Guide
**1 Control Plane + 2 Workers | Static IPs**

---

## How Talos Installation Works

Before you start, understand this flow — skipping steps here causes the stuck-on-DHCP problem:

```
Boot ISO → apply-config → Talos installs to disk → node auto-reboots → boots from disk with static IP → detach ISO
```

**The ISO is only needed for the very first boot.** `apply-config` is what triggers Talos to write itself to disk and reboot. Once that reboot completes, the node runs from disk and the ISO can be detached.

> ⚠️ **Critical:** If the ISO is attached and Talos is already on disk from a previous attempt, booting from the ISO will detect the existing install and skip the disk write. `apply-config` will return "Applied configuration without a reboot" and nothing changes. The fix is to detach the ISO, hard-reboot from the hypervisor, and re-run `apply-config` once the node comes back up.

---

## Prerequisites

### 1. Install CLI tools on your local machine

```bash
# Create bin directory if it doesn't exist
sudo mkdir -p /usr/local/bin

# Download and install talosctl v1.13.4
curl -sL https://github.com/siderolabs/talos/releases/download/v1.13.4/talosctl-linux-amd64 -o talosctl
chmod +x talosctl
sudo mv talosctl /usr/local/bin/

# Download and install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify both are installed
talosctl version --client
kubectl version --client
```

### 2. Prepare 3 VMs

Boot 3 VMs using the **Talos v1.13.4 ISO**. For each VM:

- At least 2 CPUs, 2 GB RAM (4 GB recommended for control plane)
- At least 20 GB disk attached — Talos installs itself here
- Network set to **bridged adapter** so it gets a real LAN IP via DHCP
- **Keep the ISO attached** for the first boot

Note the DHCP IP each VM receives. Check your router's lease table or watch the VM console — the IP is shown on the Talos dashboard screen.

| Role          | DHCP IP (initial) | Target Static IP |
|---------------|-------------------|------------------|
| Control Plane | `<CP_DHCP_IP>`    | `10.0.1.20`      |
| Worker 1      | `<W1_DHCP_IP>`    | `10.0.1.21`      |
| Worker 2      | `<W2_DHCP_IP>`    | `10.0.1.22`      |

> Adjust all IPs to match your network. Examples use `10.0.1.0/24` with gateway `10.0.1.2`.

---

## Phase 1: Install Talos & Bootstrap the Cluster

### Step 1 — Discover disk and interface names on each node

Run these against each node's DHCP IP **before** creating any patch files.

```bash
# Find the install disk name
talosctl get disks -n <CP_DHCP_IP> --insecure

# Find the network interface name
talosctl get links -n <CP_DHCP_IP> --insecure
```

**For the disk:** look for the largest non-USB block device (e.g. `/dev/sda`, `/dev/vda`, `/dev/nvme0n1`). Note the one marked `SYSTEM_DISK` if present.

**For the interface:** look for the entry that shows `up` under OPER STATE and `true` under LINK STATE — that is your active NIC (e.g. `ens160`, `eth0`, `enp1s0`).

Repeat for all 3 nodes. They will likely have the same values if the VMs are identically configured.

### Step 2 — Generate base configs

```bash
talosctl gen config my-talos-cluster https://10.0.1.20:6443
```

This produces three files:

- `controlplane.yaml` — applied to the control plane node
- `worker.yaml` — applied to worker nodes
- `talosconfig` — your local talosctl client config

### Step 3 — Create per-node patch files

Replace `ens160` with your actual interface name and `/dev/sda` with your actual disk device from Step 1.

**Control Plane (`patch-cp.yaml`):**

```bash
cat <<EOF > patch-cp.yaml
machine:
  install:
    disk: /dev/sda
    image: ghcr.io/siderolabs/installer:v1.13.4
    wipe: false
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        dhcp: false
        addresses:
          - 10.0.1.20/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

**Worker 1 (`patch-w1.yaml`):**

```bash
cat <<EOF > patch-w1.yaml
machine:
  install:
    disk: /dev/sda
    image: ghcr.io/siderolabs/installer:v1.13.4
    wipe: false
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        dhcp: false
        addresses:
          - 10.0.1.21/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

**Worker 2 (`patch-w2.yaml`):**

```bash
cat <<EOF > patch-w2.yaml
machine:
  install:
    disk: /dev/sda
    image: ghcr.io/siderolabs/installer:v1.13.4
    wipe: false
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        dhcp: false
        addresses:
          - 10.0.1.22/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

### Step 4 — Apply configs (this installs Talos to disk)

Apply to each node using its current **DHCP IP**. Each node will install Talos to disk and reboot automatically — the connection will drop mid-command, that is expected.

```bash
# Control plane
talosctl apply-config --insecure \
  -n <CP_DHCP_IP> -e <CP_DHCP_IP> \
  --file controlplane.yaml \
  --config-patch @patch-cp.yaml

# Worker 1
talosctl apply-config --insecure \
  -n <W1_DHCP_IP> -e <W1_DHCP_IP> \
  --file worker.yaml \
  --config-patch @patch-w1.yaml

# Worker 2
talosctl apply-config --insecure \
  -n <W2_DHCP_IP> -e <W2_DHCP_IP> \
  --file worker.yaml \
  --config-patch @patch-w2.yaml
```

You should see the connection hang or drop — this means the node is installing and rebooting. If it returns instantly with "Applied configuration without a reboot", see the Troubleshooting section.

### Step 5 — Wait for nodes to come up at their static IPs

Each node takes 2–4 minutes to install and reboot. Ping to confirm:

```bash
ping -c 3 10.0.1.20
ping -c 3 10.0.1.21
ping -c 3 10.0.1.22
```

All three should respond. Verify the addresses are correct:

```bash
talosctl get addresses -n 10.0.1.20 --insecure
talosctl get addresses -n 10.0.1.21 --insecure
talosctl get addresses -n 10.0.1.22 --insecure
```

Each should show its static IP (e.g. `10.0.1.20/24`) on `ens160`, not the old DHCP address.

### Step 6 — Configure talosctl to use the control plane static IP

```bash
talosctl config merge ./talosconfig
talosctl config endpoint 10.0.1.20
talosctl config node 10.0.1.20
```

### Step 7 — Bootstrap etcd

Run this **once only**, against the control plane:

```bash
talosctl bootstrap -n 10.0.1.20 -e 10.0.1.20
```

This starts etcd and brings up the Kubernetes control plane. It takes 2–3 minutes. You can watch progress with:

```bash
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20
```

### Step 8 — Fetch kubeconfig

```bash
talosctl kubeconfig ~/.kube/config -n 10.0.1.20 -e 10.0.1.20
```

### Step 9 — Verify all nodes are Ready

```bash
kubectl get nodes -o wide
```

Expected output:

```
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP
cp     Ready    control-plane   5m    v1.33.x   10.0.1.20
w1     Ready    <none>          4m    v1.33.x   10.0.1.21
w2     Ready    <none>          4m    v1.33.x   10.0.1.22
```

You can now safely detach the ISO from all VMs.

---

## Phase 2: Install MetalLB (L4 Load Balancer)

### Step 10 — Deploy MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### Step 11 — Create IP address pool

> ⚠️ The address range must be unused IPs on your LAN — outside your router's DHCP pool and not assigned to any device.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.1.200-10.0.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

---

## Phase 3: Install NGINX Ingress Controller (L7)

> ⚠️ The community `kubernetes/ingress-nginx` project was retired in March 2026. v1.15.1 is its final release with no further security updates. For new deployments consider [NGINX Gateway Fabric](https://github.com/nginxinc/nginx-gateway-fabric).

### Step 12 — Deploy ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Step 13 — Verify external IP

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

`EXTERNAL-IP` should show an IP from your MetalLB pool (e.g. `10.0.1.200`). If it shows `<pending>`, wait 30 seconds and try again.

---

## Phase 4: Install Cert-Manager (Automated TLS)

### Step 14 — Deploy cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml

kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

### Step 15 — Create Let's Encrypt ClusterIssuer

> Replace `your-email@example.com` with your real email address.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF
```

---

## Phase 5: Test Everything

### Step 16 — Deploy a sample app

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: nginxdemos/hello:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - hello.example.com
    secretName: hello-tls
  rules:
  - host: hello.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
EOF
```

### Step 17 — Final verification

```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl get ingress
kubectl get svc -A | grep LoadBalancer
```

---

## Operations Cheat Sheet

```bash
# Upgrade Talos OS on a node
talosctl upgrade \
  -n 10.0.1.20 -e 10.0.1.20 \
  --image ghcr.io/siderolabs/installer:v1.13.4

# Upgrade Kubernetes control plane
talosctl upgrade-k8s \
  -n 10.0.1.20 -e 10.0.1.20 \
  --to 1.33.1

# View logs for a service on a node
talosctl logs kubelet -n 10.0.1.20 -e 10.0.1.20

# List all services and their state
talosctl services -n 10.0.1.20 -e 10.0.1.20

# View kernel logs
talosctl dmesg -n 10.0.1.20 -e 10.0.1.20

# Open the live dashboard for a node
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20

# Reset a worker node to maintenance mode (keeps Talos on disk, wipes config)
talosctl reset \
  -n 10.0.1.21 -e 10.0.1.20 \
  --graceful=false \
  --system-labels-to-wipe STATE
```

---

## Troubleshooting

### "Applied configuration without a reboot"

**Cause:** The node booted from the ISO but Talos was already on disk from a previous apply. The ISO detects the existing install and skips the disk write entirely.

**Fix:**
1. Detach the ISO from the VM in your hypervisor
2. Hard-reboot the VM (power off → power on)
3. Wait for it to come up — it will boot from disk into maintenance mode at its DHCP IP
4. Re-run the `apply-config` command from Step 4

### Node won't boot after detaching ISO

**Cause:** Talos was never successfully installed to disk — the `apply-config` returned early before writing anything.

**Fix:** Re-attach the ISO, boot the VM, wait for maintenance mode, then re-run `apply-config`. The `install:` block in the patch is what tells Talos to write itself to disk. Do not detach the ISO until the node has come back up at its static IP.

### "x509: certificate signed by unknown authority"

**Cause:** You are trying to use a signed talosconfig to talk to a node that is still in maintenance mode (no PKI yet), or vice versa.

**Fix:** Use `--insecure` for any node that has not been bootstrapped yet. After bootstrapping, drop `--insecure` and use the merged talosconfig.

### "specified install disk does not exist"

**Cause:** The `disk:` value in the patch does not match the actual block device on the VM.

**Fix:** Run `talosctl get disks -n <IP> --insecure` and copy the exact device name into the patch.

### talosctl "unknown command" errors with IP addresses

**Cause:** Flags must come **after** the subcommand name, not before it.

```bash
# Wrong
talosctl --insecure --nodes 10.0.1.20 get links

# Correct
talosctl get links -n 10.0.1.20 --insecure
```

### Node stays on DHCP even after reboot

**Cause:** The patch is missing `dhcp: false`, so Talos runs both DHCP and the static config simultaneously. DHCP wins for routing.

**Fix:** Ensure `dhcp: false` is present in the interface block of your patch, as shown in Step 3.
