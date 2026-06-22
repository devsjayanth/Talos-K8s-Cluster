# Talos-K8s-Cluster
### Complete Talos Kubernetes Cluster Guide
**1 Control Plane + 2 Workers | Static IPs | June 2026**

---

## Prerequisites

**1. Install CLI tools on your local machine:**
```bash
# 1. Ensure the directory exists
sudo mkdir -p /usr/local/bin

# 2. Download and install talosctl
curl -sL https://github.com/siderolabs/talos/releases/download/v1.13.4/talosctl-linux-amd64 -o talosctl
chmod +x talosctl
sudo mv talosctl /usr/local/bin/

# 3. Download and install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 4. Verify they are installed correctly
talosctl version --client
kubectl version --client
```

**2. Prepare 3 Machines:**
Boot 3 VMs/bare-metal machines using the **Talos v1.13.4 ISO**. 
Note their **initial DHCP IPs** (assigned by your router) and their **network interface names** (e.g., `eth0`, `ens192`):
*   Control Plane: `<DHCP_CP>` -> Target Static: `192.168.1.10`
*   Worker 1: `<DHCP_W1>` -> Target Static: `192.168.1.11`
*   Worker 2: `<DHCP_W2>` -> Target Static: `192.168.1.12`

---

## Phase 1: Bootstrap Talos Cluster (Static IPs)

### Step 1 — Generate Base Configs
Generate configs using the **target static IP** of the control plane:
```bash
talosctl gen config my-talos-cluster https://192.168.1.10:6443
```

### Step 2 — Create Static IP Patches
*⚠️ Change `eth0` to your actual interface name. Update IPs, CIDR (`/24`), and Gateway to match your network.*

```bash
# Control Plane Patch
cat <<EOF > patch-cp.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses: ["192.168.1.10/24"]
        routes: [{"network": "0.0.0.0/0", "gateway": "192.168.1.1"}]
EOF

# Worker 1 Patch
cat <<EOF > patch-w1.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses: ["192.168.1.11/24"]
        routes: [{"network": "0.0.0.0/0", "gateway": "192.168.1.1"}]
EOF

# Worker 2 Patch
cat <<EOF > patch-w2.yaml
machine:
  network:
    interfaces:
      - interface: eth0
        addresses: ["192.168.1.12/24"]
        routes: [{"network": "0.0.0.0/0", "gateway": "192.168.1.1"}]
EOF
```

### Step 3 — Apply Patches to Configs
```bash
talosctl config patch controlplane.yaml --file patch-cp.yaml
talosctl config patch worker.yaml --file patch-w1.yaml --output worker1.yaml
talosctl config patch worker.yaml --file patch-w2.yaml --output worker2.yaml
```

### Step 4 — Apply Configs to Nodes
*Note: We use the **initial DHCP IPs** here. The nodes will reboot and switch to the static IPs defined in the patches.*

```bash
# Control Plane
talosctl apply-config --insecure --nodes <DHCP_CP> --file controlplane.yaml

# Workers
talosctl apply-config --insecure --nodes <DHCP_W1> --file worker1.yaml
talosctl apply-config --insecure --nodes <DHCP_W2> --file worker2.yaml
```

### Step 5 — Configure Local `talosctl` & Bootstrap
*Wait ~2-3 minutes for the nodes to reboot and acquire their new static IPs.*

```bash
# Merge config and point to the NEW static IPs
talosctl config merge ./talosconfig
talosctl config endpoints 192.168.1.10
talosctl config nodes 192.168.1.10

# Bootstrap etcd and get kubeconfig
talosctl bootstrap
talosctl kubeconfig ~/.kube
```

### Step 6 — Verify Nodes
```bash
kubectl get nodes
```
*Expected: 3 nodes, all `Ready`.*

---

## Phase 2: Install MetalLB (L4 Load Balancer)

### Step 7 — Deploy MetalLB
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
```

### Step 8 — Create IP Pool
> ⚠️ Change the `addresses` range to **unused IPs** in your local network.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
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

### Step 9 — Deploy Ingress-NGINX
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```

### Step 10 — Verify External IP
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```
*Expected: `EXTERNAL-IP` shows an IP from your MetalLB pool (e.g., `192.168.1.240`).*

---

## Phase 4: Install Cert-Manager (Automated TLS)

### Step 11 — Deploy Cert-Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
kubectl wait --namespace cert-manager --for=condition=ready pod --selector=app.kubernetes.io/instance=cert-manager --timeout=120s
```

### Step 12 — Create Let's Encrypt ClusterIssuer
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
          class: nginx
EOF
```

---

## Phase 5: Test Everything

### Step 13 — Deploy a Sample App
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

### Step 14 — Final Verification
```bash
kubectl get pods -A
kubectl get ingress
kubectl get svc -A | grep LoadBalancer
```

---

## Day-2 Operations Cheat Sheet

```bash
# Upgrade Talos OS on a specific node
talosctl upgrade --nodes 192.168.1.10 --image ghcr.io/siderolabs/installer:v1.13.4

# Upgrade Kubernetes control plane
talosctl upgrade-k8s --to 1.33.1

# View kubelet logs on a node
talosctl --nodes 192.168.1.10 logs kubelet

# Reset a node to maintenance mode (wipes data)
talosctl reset --nodes 192.168.1.11 --graceful=false
```
