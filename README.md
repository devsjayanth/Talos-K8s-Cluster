# Talos-K8s-Cluster

**1 Control Plane + 2 Workers | June 2026**

---

## Prerequisites

Install on your **local machine**:

```bash
# talosctl
curl -sL https://github.com/siderolabs/talos/releases/download/v1.13.4/talosctl-$(uname -s | tr '[:upper:]' '[:lower:]')-amd64 -o /usr/local/bin/talosctl
chmod +x /usr/local/bin/talosctl

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/$(uname -s | tr '[:upper:]' '[:lower:]')/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

Prepare **3 machines** (VMs or bare metal) booted into the **Talos v1.13.4 ISO** (Maintenance Mode). Note their IPs:

| Role | Variable |
|---|---|
| Control Plane | `<CP_IP>` |
| Worker 1 | `<W1_IP>` |
| Worker 2 | `<W2_IP>` |

---

## Phase 1: Bootstrap Talos Cluster

### Step 1 — Generate Configs

```bash
talosctl gen config my-talos-cluster https://<CP_IP>:6443
```

This creates:
- `controlplane.yaml`
- `worker.yaml`
- `talosconfig`

### Step 2 — Apply Configs to Nodes

```bash
# Control Plane
talosctl apply-config --insecure --nodes <CP_IP> --file controlplane.yaml

# Workers
talosctl apply-config --insecure --nodes <W1_IP> --file worker.yaml
talosctl apply-config --insecure --nodes <W2_IP> --file worker.yaml
```

Nodes will reboot into Talos OS.

### Step 3 — Configure Local talosctl

```bash
talosctl config merge ./talosconfig
talosctl config endpoints <CP_IP>
talosctl config nodes <CP_IP>
```

### Step 4 — Bootstrap etcd

```bash
talosctl bootstrap
```

### Step 5 — Retrieve Kubeconfig

```bash
talosctl kubeconfig ~/.kube
```

### Step 6 — Verify Nodes

```bash
kubectl get nodes
```

Expected output: **3 nodes, all `Ready`**.

---

## Phase 2: Install MetalLB (L4 Load Balancer)

### Step 7 — Deploy MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```

Wait for pods to be ready:

```bash
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
```

Wait for readiness:

```bash
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```

### Step 10 — Verify External IP Assigned

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

Expected: `EXTERNAL-IP` shows an IP from your MetalLB pool (e.g., `192.168.1.240`).

---

## Phase 4: Install Cert-Manager (Optional — Automated TLS)

### Step 11 — Deploy Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
```

Wait for readiness:

```bash
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
    cert-manager.io/cluster-issuer: letsencrypt-prod  # Remove if no cert-manager
spec:
  ingressClassName: nginx
  tls:                                                  # Remove if no cert-manager
  - hosts:                                              # Remove if no cert-manager
    - hello.example.com                                 # Remove if no cert-manager
    secretName: hello-tls                               # Remove if no cert-manager
  rules:
  - host: hello.example.com                             # Change to your domain or remove host for IP access
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
# Check all pods across namespaces
kubectl get pods -A

# Check ingress got an address
kubectl get ingress

# Check MetalLB assigned IPs
kubectl get svc -A | grep LoadBalancer
```

---

## Summary of Components

| Component | Version | Purpose |
|---|---|---|
| **Talos Linux** | v1.13.4 | Immutable Kubernetes OS |
| **Flannel** | Built-in | Pod networking (CNI) |
| **MetalLB** | v0.16.1 | L4 Load Balancer (assigns IPs) |
| **Ingress-NGINX** | v1.15.1 | L7 HTTP/HTTPS routing |
| **Cert-Manager** | v1.20.2 | Automated TLS certificates |

---

## Useful Day-2 Commands

```bash
# Upgrade Talos on a node
talosctl upgrade --nodes <NODE_IP> --image ghcr.io/siderolabs/installer:v1.13.4

# Upgrade Kubernetes
talosctl upgrade-k8s --to 1.33.1

# View Talos service logs
talosctl logs kubelet

# Enter a node's shell
talosctl --nodes <NODE_IP> shell
```
