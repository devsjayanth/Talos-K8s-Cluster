# Talos Linux — Kubernetes
### 1 Control Plane + 2 Workers | Static IPs

---

## Before You Start

Talos Linux is a minimal OS built only for Kubernetes. There is no SSH, no shell, no package manager.
Everything is managed through `talosctl` from your local machine.

**How it works:**
```
Boot ISO → node gets DHCP IP → apply-config → Talos installs to disk → node reboots → comes up at static IP
```

The ISO is only needed for the first boot. Once Talos installs to disk and the node reboots, you can detach the ISO.

---

## What You Need

- 3 machines (VMs or bare metal) with Ethernet — **Wi-Fi is not supported**
- Your local Linux/Mac machine with internet access
- Your router's gateway IP and a free IP range for static addresses

---

## Step 1 — Install talosctl and kubectl on your local machine

```bash
# Install talosctl
curl -sL https://github.com/siderolabs/talos/releases/download/v1.13.4/talosctl-linux-amd64 -o talosctl
chmod +x talosctl
sudo mv talosctl /usr/local/bin/

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify both are installed
talosctl version --client
kubectl version --client
```

---

## Step 2 — Download the Talos ISO and boot your nodes

```bash
# Download the ISO
curl -LO https://github.com/siderolabs/talos/releases/download/v1.13.4/metal-amd64.iso
```

**Bare metal:** Flash to a USB drive (replace `/dev/sdX` with your USB device):
```bash
# Find your USB device first
lsblk

# Flash the ISO
sudo dd if=metal-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

**VM (VMware / Proxmox / VirtualBox / KVM):** Attach `metal-amd64.iso` as the CD/DVD drive. Set it as the first boot device. Create one VM per node with at least:
- 2 CPUs, 2 GB RAM (4 GB for control plane)
- 20 GB disk
- Network adapter set to **bridged**

Boot all 3 VMs/machines from the ISO. Each will show a dashboard screen. Look at the top-right corner — it shows the DHCP IP assigned by your router. Write those down.

```
Control Plane DHCP IP:  _______________
Worker 1 DHCP IP:       _______________
Worker 2 DHCP IP:       _______________
```

---

## Step 3 — Find the install disk name on each node

Run this from your local machine against each node's DHCP IP.
You need the exact disk device path to tell Talos where to install itself.

```bash
# Control plane
talosctl get disks -n <CP_DHCP_IP> --insecure

# Worker 1
talosctl get disks -n <W1_DHCP_IP> --insecure

# Worker 2
talosctl get disks -n <W2_DHCP_IP> --insecure
```

**Example output:**
```
NODE   NAMESPACE   TYPE   ID    SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL
       runtime     Disk   sda   50 GB    false        sata        false               VMware Virtual S
       runtime     Disk   sr0   4.7 GB   false        ata         true                VMware IDE CDR10
```

**Pick the main disk** (not `sr0` — that is the CD-ROM). The `ID` column gives you the name.
Prepend `/dev/` to get the full path:

| What you see in ID column | Full disk path to use |
|---------------------------|-----------------------|
| `sda`                     | `/dev/sda`            |
| `vda`                     | `/dev/vda`            |
| `nvme0n1`                 | `/dev/nvme0n1`        |
| `sdb`                     | `/dev/sdb`            |

```
Write down your disk path: /dev/_______________
```

---

## Step 4 — Find the network interface name on each node

Run this from your local machine against each node's DHCP IP.
You need the exact interface name to configure the static IP.

```bash
# Control plane
talosctl get links -n <CP_DHCP_IP> --insecure

# Worker 1
talosctl get links -n <W1_DHCP_IP> --insecure

# Worker 2
talosctl get links -n <W2_DHCP_IP> --insecure
```

**Example output:**
```
NODE   NAMESPACE   TYPE         ID        VERSION   ALIAS   TYPE    KIND   HW ADDR            OPER STATE   LINK STATE
       network     LinkStatus   bond0     1                 ether   bond   3e:6a:d9:05:c6:c9   down        false
       network     LinkStatus   ens160    3                 ether          00:0c:29:a2:07:6c   up          true
       network     LinkStatus   lo        1                 loopback       00:00:00:00:00:00   unknown     true
```

**Pick the interface** where OPER STATE is `up` and LINK STATE is `true`. Ignore `lo` (loopback) and anything that is `down`.

```
Write down your interface name: _______________
```

Common interface names by platform:

| Platform         | Typical interface name |
|------------------|------------------------|
| VMware           | `ens160` or `ens192`   |
| VirtualBox       | `eth0` or `enp0s3`     |
| KVM/QEMU         | `eth0` or `enp1s0`     |
| Proxmox (VirtIO) | `eth0` or `ens18`      |
| Bare metal       | `enp2s0` or `eno1`     |

---

## Step 5 — Plan your static IPs

Fill in your values before running any commands. Use IPs that are not in your router's DHCP pool.

```
Your gateway IP:            10.0.1.___
Your DNS server IP:         10.0.1.___  (usually same as gateway)

Control plane static IP:    10.0.1.___
Worker 1 static IP:         10.0.1.___
Worker 2 static IP:         10.0.1.___

MetalLB IP range (unused):  10.0.1.___ - 10.0.1.___
```

---

## Step 6 — Generate cluster configs

Run this on your local machine. Substitute your values from Steps 3, 4, and 5.

In the command below, replace:
- `10.0.1.20` → your control plane static IP (appears twice)
- `/dev/sda` → your disk path from Step 3
- `ens160` → your interface name from Step 4
- `10.0.1.2` → your gateway IP
- `10.0.1.20/24` → your control plane static IP with `/24` subnet

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

This generates three files in your current directory:
```
controlplane.yaml   ← config for the control plane node (has static IP baked in)
worker.yaml         ← base config for worker nodes
talosconfig         ← your talosctl client credentials
```

---

## Step 7 — Apply config to the control plane node

Replace `<CP_DHCP_IP>` with the DHCP IP of your control plane from Step 2.

```bash
talosctl apply-config --insecure \
  -n <CP_DHCP_IP> -e <CP_DHCP_IP> \
  --file controlplane.yaml
```

The command will hang for a few seconds, then the connection drops — **that is correct**. The node is installing Talos to disk and rebooting. Do not interrupt it.

---

## Step 8 — Apply config to Worker 1

Replace:
- `<W1_DHCP_IP>` → Worker 1's DHCP IP from Step 2
- `ens160` → your interface name from Step 4
- `10.0.1.21/24` → Worker 1's target static IP
- `10.0.1.2` → your gateway IP (appears twice)

```bash
talosctl apply-config --insecure \
  -n <W1_DHCP_IP> -e <W1_DHCP_IP> \
  --file worker.yaml \
  --config-patch '[
    {
      "op": "add",
      "path": "/machine/network",
      "value": {
        "nameservers": ["10.0.1.2"],
        "interfaces": [
          {
            "interface": "ens160",
            "dhcp": false,
            "addresses": ["10.0.1.21/24"],
            "routes": [
              {"network": "0.0.0.0/0", "gateway": "10.0.1.2"}
            ]
          }
        ]
      }
    }
  ]'
```

---

## Step 9 — Apply config to Worker 2

Replace:
- `<W2_DHCP_IP>` → Worker 2's DHCP IP from Step 2
- `ens160` → your interface name from Step 4
- `10.0.1.22/24` → Worker 2's target static IP
- `10.0.1.2` → your gateway IP (appears twice)

```bash
talosctl apply-config --insecure \
  -n <W2_DHCP_IP> -e <W2_DHCP_IP> \
  --file worker.yaml \
  --config-patch '[
    {
      "op": "add",
      "path": "/machine/network",
      "value": {
        "nameservers": ["10.0.1.2"],
        "interfaces": [
          {
            "interface": "ens160",
            "dhcp": false,
            "addresses": ["10.0.1.22/24"],
            "routes": [
              {"network": "0.0.0.0/0", "gateway": "10.0.1.2"}
            ]
          }
        ]
      }
    }
  ]'
```

---

## Step 10 — Wait for nodes to come up at their static IPs

All 3 nodes are now installing Talos to disk and rebooting. This takes **3–5 minutes**.

Once they are back up, ping each static IP to confirm:

```bash
ping -c 3 10.0.1.20
ping -c 3 10.0.1.21
ping -c 3 10.0.1.22
```

All three should respond. If a node doesn't respond, see the Troubleshooting section.

Confirm addresses look correct (no leftover DHCP address):

```bash
talosctl get addresses -n 10.0.1.20 --insecure
talosctl get addresses -n 10.0.1.21 --insecure
talosctl get addresses -n 10.0.1.22 --insecure
```

Each should show only its static IP on your interface (e.g. `10.0.1.20/24` on `ens160`).

**You can now detach the ISO from all VMs.**

---

## Step 11 — Configure talosctl

```bash
talosctl config merge ./talosconfig
talosctl config endpoint 10.0.1.20
talosctl config node 10.0.1.20
```

Test the connection (no `--insecure` needed after this point):

```bash
talosctl version -n 10.0.1.20 -e 10.0.1.20
```

---

## Step 12 — Bootstrap etcd

Run **once only** on the control plane. This starts etcd and brings up Kubernetes:

```bash
talosctl bootstrap -n 10.0.1.20 -e 10.0.1.20
```

Watch the cluster come up (takes ~3 minutes):

```bash
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20
```

Wait until the stage shows `Running` before continuing.

---

## Step 13 — Download kubeconfig

```bash
talosctl kubeconfig ~/.kube/config -n 10.0.1.20 -e 10.0.1.20
```

---

## Step 14 — Verify all nodes are Ready

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

Your Kubernetes cluster is running.

---

## Step 15 — Install MetalLB (Load Balancer)

MetalLB provides `LoadBalancer` services on bare metal / home lab networks.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Create an IP pool. Replace the range with unused IPs on your LAN (not in your router's DHCP range):

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

## Step 16 — Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Verify MetalLB assigned it an external IP:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

`EXTERNAL-IP` should show an IP from your MetalLB pool (e.g. `10.0.1.200`). If it shows `<pending>`, wait 30 seconds and retry.

---

## Step 17 — Install Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml

kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

Create a Let's Encrypt issuer — replace `your-email@example.com` with your real email:

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

## Step 18 — Deploy a test app

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

Final verification:

```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl get ingress
kubectl get svc -A | grep LoadBalancer
```

---

## Operations

```bash
# Live dashboard for a node
talosctl dashboard -n 10.0.1.20 -e 10.0.1.20

# View all service states on a node
talosctl services -n 10.0.1.20 -e 10.0.1.20

# View kubelet logs
talosctl logs kubelet -n 10.0.1.20 -e 10.0.1.20

# View kernel logs
talosctl dmesg -n 10.0.1.20 -e 10.0.1.20

# Upgrade Talos OS (one node at a time)
talosctl upgrade -n 10.0.1.20 -e 10.0.1.20 \
  --image ghcr.io/siderolabs/installer:v1.13.4

# Upgrade Kubernetes version
talosctl upgrade-k8s -n 10.0.1.20 -e 10.0.1.20 --to 1.33.1

# Reset a worker node (wipes config, keeps Talos on disk)
talosctl reset -n 10.0.1.21 -e 10.0.1.20 \
  --graceful=false --system-labels-to-wipe STATE
```

---

## Troubleshooting

### "Applied configuration without a reboot"

The ISO is attached and Talos detected an existing install on disk from a previous attempt. It stages config but does not reboot.

**Fix:**
1. Detach the ISO from the VM in your hypervisor
2. Hard-reboot the VM (power off → power on)
3. Wait for it to come back up at its DHCP IP
4. Re-run the `apply-config` command for that node

### Node won't boot after detaching ISO

Talos was never written to disk — the `apply-config` returned early before finishing. Re-attach the ISO and re-run `apply-config`. Only detach the ISO after the node is reachable at its static IP.

### Node still responds only on its DHCP IP after reboot

Two possible causes:
1. **Wrong interface name** — the config targeted a different interface than what actually exists. Run `talosctl get links -n <IP> --insecure` and check the name matches exactly.
2. **`dhcp: false` missing** — DHCP and static run at the same time and DHCP wins. The commands in this guide already include `dhcp: false`, so this only happens if the patch was modified.

### "x509: certificate signed by unknown authority"

You are talking to a node that hasn't been bootstrapped yet without `--insecure`. Always use `--insecure` for nodes in maintenance mode (Steps 3, 4, 7, 8, 9, 10).

### talosctl "unknown command" errors when passing an IP

Flags must come **after** the subcommand:

```bash
# Wrong
talosctl --insecure --nodes 10.0.1.20 get links

# Correct
talosctl get links -n 10.0.1.20 --insecure
```

### Certificates expire after 1 year

The `talosconfig` and `kubeconfig` client files expire after 1 year. Regenerate before they expire:

```bash
talosctl config regenerate -n 10.0.1.20 -e 10.0.1.20
talosctl kubeconfig ~/.kube/config -n 10.0.1.20 -e 10.0.1.20
```
