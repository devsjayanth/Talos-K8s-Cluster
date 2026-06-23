# Talos Linux — Kubernetes Home Lab
### 1 Control Plane + 2 Workers | Static IPs

---

## Prerequisites

Install tools on your **local machine**:

```bash
# talosctl
curl -sL https://github.com/siderolabs/talos/releases/download/v1.13.4/talosctl-linux-amd64 -o talosctl
chmod +x talosctl && sudo mv talosctl /usr/local/bin/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# verify
talosctl version --client
kubectl version --client
```

---

## Step 1 — Boot nodes from ISO

Download the ISO and flash it to USB or attach as a VM optical drive:

```bash
curl -LO https://github.com/siderolabs/talos/releases/download/v1.13.4/metal-amd64.iso

# Bare metal — write to USB (replace /dev/sdX with your USB device)
sudo dd if=metal-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Boot all 3 machines from the ISO. Each will show a dashboard with its DHCP IP in the top-right corner. Note those IPs.

| Role          | DHCP IP (temporary) | Target Static IP |
|---------------|---------------------|------------------|
| Control Plane | `<CP_DHCP_IP>`      | `10.0.1.20`      |
| Worker 1      | `<W1_DHCP_IP>`      | `10.0.1.21`      |
| Worker 2      | `<W2_DHCP_IP>`      | `10.0.1.22`      |

> Adjust IPs to match your network. Examples use `10.0.1.0/24`, gateway `10.0.1.2`.

---

## Step 2 — Find disk and interface names

Run against each node's DHCP IP before generating any config:

```bash
# Disk name (look for the main block device — /dev/sda, /dev/vda, /dev/nvme0n1)
talosctl get disks -n <CP_DHCP_IP> --insecure

# Interface name (look for the one that is "up" and link state "true")
talosctl get links -n <CP_DHCP_IP> --insecure
```

Note the exact disk path and interface name — you'll use them in the next step.

---

## Step 3 — Generate configs with static IP and install disk baked in

Run this on your local machine. Replace:
- `10.0.1.20` → your control plane static IP
- `ens160` → your actual interface name from Step 2
- `/dev/sda` → your actual disk from Step 2
- `10.0.1.2` → your gateway/DNS

```bash
talosctl gen config my-talos-cluster https://10.0.1.20:6443 \
  --config-patch '[
    {
      "op": "add",
      "path": "/machine/install",
      "value": {
        "disk": "/dev/sda",
        "image": "ghcr.io/siderolabs/installer:v1.13.4",
        "wipe": false
      }
    },
    {
      "op": "add",
      "path": "/machine/network",
      "value": {
        "nameservers": ["10.0.1.2"],
        "interfaces": [
          {
            "interface": "ens160",
            "dhcp": false,
            "addresses": ["10.0.1.20/24"],
            "routes": [
              {"network": "0.0.0.0/0", "gateway": "10.0.1.2"}
            ]
          }
        ]
      }
    }
  ]' \
  --config-patch-worker '[
    {
      "op": "add",
      "path": "/machine/install",
      "value": {
        "disk": "/dev/sda",
        "image": "ghcr.io/siderolabs/installer:v1.13.4",
        "wipe": false
      }
    }
  ]'
```

This generates `controlplane.yaml`, `worker.yaml`, and `talosconfig` in your current directory.

> The control plane config now has the static IP and install disk baked in. Workers share the same disk config but get their IPs applied individually in Step 4.

---

## Step 4 — Apply configs to each node

Apply to each node using its current **DHCP IP**. The node installs Talos to disk and reboots automatically — the connection drops, that is normal.

```bash
# Control plane (uses the already-patched controlplane.yaml)
talosctl apply-config --insecure \
  -n <CP_DHCP_IP> -e <CP_DHCP_IP> \
  --file controlplane.yaml
```

For workers, inject each worker's static IP on the fly:

```bash
# Worker 1
talosctl apply-config --insecure \
  -n <W1_DHCP_IP> -e <W1_DHCP_IP> \
  --file worker.yaml \
  --config-patch '[{"op":"add","path":"/machine/network","value":{"nameservers":["10.0.1.2"],"interfaces":[{"interface":"ens160","dhcp":false,"addresses":["10.0.1.21/24"],"routes":[{"network":"0.0.0.0/0","gateway":"10.0.1.2"}]}]}}]'

# Worker 2
talosctl apply-config --insecure \
  -n <W2_DHCP_IP> -e <W2_DHCP_IP> \
  --file worker.yaml \
  --config-patch '[{"op":"add","path":"/machine/network","value":{"nameservers":["10.0.1.2"],"interfaces":[{"interface":"ens160","dhcp":false,"addresses":["10.0.1.22/24"],"routes":[{"network":"0.0.0.0/0","gateway":"10.0.1.2"}]}]}}]'
```

Wait 3–5 minutes for all nodes to install and reboot.

---

## Step 5 — Verify static IPs

```bash
ping -c 3 10.0.1.20
ping -c 3 10.0.1.21
ping -c 3 10.0.1.22
```

All three should respond. You can now detach the ISO from all VMs.

---

## Step 6 — Configure talosctl

```bash
talosctl config merge ./talosconfig
talosctl config endpoint 10.0.1.20
talosctl config node 10.0.1.20
```

---

## Step 7 — Bootstrap etcd

Run **once only** on the control plane:

```bash
talosctl bootstrap -n 10.0.1.20 -e 10.0.1.20
```

Watch progress:

```bash
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20
```

Wait until the dashboard shows stage `Running` (~3 minutes).

---

## Step 8 — Get kubeconfig

```bash
talosctl kubeconfig ~/.kube/config -n 10.0.1.20 -e 10.0.1.20
```

---

## Step 9 — Verify cluster

```bash
kubectl get nodes -o wide
```

Expected:

```
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP
cp     Ready    control-plane   5m    v1.33.x   10.0.1.20
w1     Ready    <none>          4m    v1.33.x   10.0.1.21
w2     Ready    <none>          4m    v1.33.x   10.0.1.22
```

---

## Step 10 — Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Create an IP pool from unused IPs on your LAN:

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

## Step 11 — Install NGINX Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Verify external IP was assigned by MetalLB
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

---

## Step 12 — Install Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml

kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

Create a Let's Encrypt issuer (replace the email):

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

## Step 13 — Deploy test app

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

Final check:

```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl get ingress
kubectl get svc -A | grep LoadBalancer
```

---

## Operations

```bash
# Live node dashboard
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20

# Service states
talosctl services -n 10.0.1.20 -e 10.0.1.20

# Kubelet logs
talosctl logs kubelet -n 10.0.1.20 -e 10.0.1.20

# Kernel logs
talosctl dmesg -n 10.0.1.20 -e 10.0.1.20

# Upgrade Talos OS
talosctl upgrade -n 10.0.1.20 -e 10.0.1.20 \
  --image ghcr.io/siderolabs/installer:v1.13.4

# Upgrade Kubernetes
talosctl upgrade-k8s -n 10.0.1.20 -e 10.0.1.20 --to 1.33.1

# Reset a worker (wipes config, keeps Talos on disk)
talosctl reset -n 10.0.1.21 -e 10.0.1.20 \
  --graceful=false --system-labels-to-wipe STATE
```

---

## Troubleshooting

**"Applied configuration without a reboot"**
The ISO is booted but Talos is already on disk from a previous attempt. Detach the ISO, hard-reboot the VM from the hypervisor, wait for it to come back up at its DHCP IP, then re-run the `apply-config` command.

**Node won't boot after removing ISO**
Talos was never written to disk. Re-attach the ISO, boot, and re-run `apply-config`. Only detach the ISO after the node is reachable at its static IP.

**Node still on DHCP after reboot**
Wrong interface name in the config. Run `talosctl get links -n <IP> --insecure` and confirm the name matches exactly what is in your config.

**"x509: certificate signed by unknown authority"**
Add `--insecure` when talking to a node that has not been bootstrapped yet. Drop it after bootstrapping.

**talosctl "unknown command" with an IP address**
Flags must come after the subcommand:
```bash
# Wrong
talosctl --insecure --nodes 10.0.1.20 get links
# Correct
talosctl get links -n 10.0.1.20 --insecure
```
