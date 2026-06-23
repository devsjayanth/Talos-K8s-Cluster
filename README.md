# Talos-K8s-Cluster
### Complete Talos Kubernetes Cluster Guide
**1 Control Plane + 2 Workers | Static IPs**

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
*⚠️ Change `ens160` to your actual interface name. Update IPs, CIDR (`/24`), and Gateway to match your network.*

#### 1. Create the Patch Files

**Control Plane (`patch-cp.yaml`):**
```bash
cat <<EOF > patch-cp.yaml
machine:
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        addresses:
          - 10.0.1.20/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

**Worker 1 (`patch-w1.yaml`):** *(Change `10.0.1.21` to your desired Worker 1 IP)*
```bash
cat <<EOF > patch-w1.yaml
machine:
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        addresses:
          - 10.0.1.21/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

**Worker 2 (`patch-w2.yaml`):** *(Change `10.0.1.22` to your desired Worker 2 IP)*
```bash
cat <<EOF > patch-w2.yaml
machine:
  network:
    nameservers:
      - 10.0.1.2
    interfaces:
      - interface: ens160
        addresses:
          - 10.0.1.22/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.0.1.2
EOF
```

#### 2. Apply the Configs

Apply the configs to each node using its **current DHCP IP** and the correct patch file. The nodes will reboot into their static IPs.

```bash
# Control Plane
talosctl apply-config --insecure --nodes <CP_DHCP_IP> --file controlplane.yaml --config-patch @patch-cp.yaml

# Workers (replace with their actual current DHCP IPs)
talosctl apply-config --insecure --nodes <W1_DHCP_IP> --file worker.yaml --config-patch @patch-w1.yaml
talosctl apply-config --insecure --nodes <W2_DHCP_IP> --file worker.yaml --config-patch @patch-w2.yaml
```

*Wait ~2-3 minutes for all nodes to reboot and acquire their static IPs before continuing.*

### Step 3 — Configure Local `talosctl` & Bootstrap

```bash
# Merge talosconfig into ~/.talos/config and point to the new static CP IP
talosctl config merge ./talosconfig
talosctl config endpoints 192.168.1.10
talosctl config nodes 192.168.1.10

# Bootstrap etcd (run once, on the control plane node only)
talosctl bootstrap

# Fetch kubeconfig and merge it into ~/.kube/config
talosctl kubeconfig ~/.kube/config
```

### Step 4 — Verify Nodes
```bash
kubectl get nodes
```
*Expected: 3 nodes, all `Ready`.*

---

## Phase 2: Install MetalLB (L4 Load Balancer)

### Step 5 — Deploy MetalLB
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### Step 6 — Create IP Pool
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

> ⚠️ **Note:** The community `ingress-nginx` project (kubernetes/ingress-nginx) was retired in March 2026. Version v1.15.1 is its final release and receives no further security updates. For new deployments consider [NGINX Gateway Fabric](https://github.com/nginxinc/nginx-gateway-fabric) (Gateway API) or another actively maintained ingress controller.

### Step 7 — Deploy Ingress-NGINX
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Step 8 — Verify External IP
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```
*Expected: `EXTERNAL-IP` shows an IP from your MetalLB pool (e.g., `192.168.1.240`).*

---

## Phase 4: Install Cert-Manager (Automated TLS)

### Step 9 — Deploy Cert-Manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

### Step 10 — Create Let's Encrypt ClusterIssuer
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

### Step 11 — Deploy a Sample App
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

### Step 12 — Final Verification
```bash
kubectl get pods -A
kubectl get ingress
kubectl get svc -A | grep LoadBalancer
```

---

## Operations Cheat Sheet

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
