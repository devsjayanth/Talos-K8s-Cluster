# ⚓Talos Linux — Kubernetes
### 1 Control Plane + 2 Workers
Here is the complete, corrected, and optimized step-by-step guide for Talos v1.13.

### Prerequisites
*   Boot all 3 nodes from the Talos ISO. Note their **DHCP IPs**.
*   Define your variables:
    *   **`<DISK>`**: Install disk (e.g., `/dev/nvme0n1` or `/dev/sda`)
    *   **`<IFACE>`**: Network interface (e.g., `ens160`)
    *   **`<GW>`**: Gateway IP (e.g., `10.0.1.2`)
    *   **`<CP_IP>`**, **`<W1_IP>`**, **`<W2_IP>`**: Target static IPs

---

### Step 1: Discover Disk and Interface
Run these against the Control Plane's DHCP IP to confirm your disk and interface names:
```bash
talosctl get disks -n <CP_DHCP_IP> --insecure   # Note the main disk (ignore sr0)
talosctl get links -n <CP_DHCP_IP> --insecure   # Note the interface with state "up"
```

---

### Step 2: Create Patch Files
Because Talos v1.13 uses multi-document configs, we combine the `HostnameConfig` and `machine.network` into single patch files.

**Control Plane (`cp-patch.yaml`):**
```bash
cat <<'EOF' > cp-patch.yaml
---
apiVersion: v1alpha1
kind: HostnameConfig
auto: off
hostname: k8s-control-1
---
machine:
  network:
    nameservers:
      - <GW>
    interfaces:
      - interface: <IFACE>
        dhcp: false
        addresses:
          - <CP_IP>/24
        routes:
          - network: 0.0.0.0/0
            gateway: <GW>
EOF
```

**Worker 1 (`w1-patch.yaml`):**
```bash
cat <<'EOF' > w1-patch.yaml
---
apiVersion: v1alpha1
kind: HostnameConfig
auto: off
hostname: k8s-worker-1
---
machine:
  network:
    nameservers:
      - <GW>
    interfaces:
      - interface: <IFACE>
        dhcp: false
        addresses:
          - <W1_IP>/24
        routes:
          - network: 0.0.0.0/0
            gateway: <GW>
EOF
```

**Worker 2 (`w2-patch.yaml`):**
```bash
cat <<'EOF' > w2-patch.yaml
---
apiVersion: v1alpha1
kind: HostnameConfig
auto: off
hostname: k8s-worker-2
---
machine:
  network:
    nameservers:
      - <GW>
    interfaces:
      - interface: <IFACE>
        dhcp: false
        addresses:
          - <W2_IP>/24
        routes:
          - network: 0.0.0.0/0
            gateway: <GW>
EOF
```

---

### Step 3: Generate Base Configs
Generate the base configs with the install disk and image baked in.
```bash
talosctl gen config my-cluster https://<CP_IP>:6443 \
  --install-disk <DISK> \
  --install-image ghcr.io/siderolabs/installer:v1.13.4 \
  --force
```

---

### Step 4: Apply Configs
Apply the base config + patch to each node. **Detach the ISO from all VMs after applying.**

**1. Control Plane:**
```bash
talosctl apply-config --insecure -n <CP_DHCP_IP> --file controlplane.yaml --config-patch @cp-patch.yaml
```
*(Wait 3-5 minutes for it to reboot to `<CP_IP>`)*

**2. Worker 1:**
```bash
talosctl apply-config --insecure -n <W1_DHCP_IP> --file worker.yaml --config-patch @w1-patch.yaml
```

**3. Worker 2:**
```bash
talosctl apply-config --insecure -n <W2_DHCP_IP> --file worker.yaml --config-patch @w2-patch.yaml
```

---

### Step 5: Bootstrap and Get Kubeconfig
Once the Control Plane is back online at its static IP:

```bash
# 1. Merge client credentials
talosctl config merge ./talosconfig
talosctl config endpoints <CP_IP>
talosctl config nodes <CP_IP>

# 2. Bootstrap etcd (Run ONLY ONCE)
talosctl bootstrap -n <CP_IP> -e <CP_IP>

# 3. Download kubeconfig
talosctl kubeconfig -n <CP_IP> -e <CP_IP> -f
# This saves the config to ~/.kube/config and overwrites any existing file (-f).
```

---

### Step 6: Verify Cluster Health
Wait 2-3 minutes after bootstrap for the API server to fully initialize.

```bash
# 1. Check Talos OS level health
talosctl health -n <CP_IP> -e <CP_IP>
# (Wait until all checks say "OK")

# 2. Check Kubernetes nodes
kubectl get nodes
```
*Expected output: All 3 nodes should show `STATUS` as `Ready`.*

---
# 💾Volume Setup
Because Talos Linux is an immutable OS, you cannot SSH into the node to format the disk with `mkfs`. Instead, you must use a **CSI (Container Storage Interface) driver** that runs as a privileged pod, detects the raw blank disk, formats it, and mounts it for Kubernetes.

The best and easiest tool for raw local disks is **OpenEBS LocalPV**.

Here is the step-by-step guide to setting it up.

### Step 1: Verify the disk is visible to Talos
Check if Talos sees the new blank NVMe drive on Worker 2:
```bash
talosctl get disks -n <W2_STATIC_IP>
```
*Note the ID of the new blank drive (e.g., `nvme1n1`). Do not include `/dev/` in the next steps, just the ID.*

---

### Step 2: Install OpenEBS LocalPV Provisioner
If you don't have Helm installed, install it first:
```bash
export VERIFY_CHECKSUM=false
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Add the OpenEBS repo and install the LocalPV provisioner:
```bash
helm repo add openebs https://openebs.github.io/charts
helm repo update
```
Install OpenEBS with LocalPV Device support:
```
helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set localpv.enabled=true \
  --set localpv.device.enabled=true \
  --set openebs-ndm.enabled=true
```
Now verify everything is running:
1. Check pods:
```
kubectl get pods -n openebs
```
2. Check StorageClasses:
```
kubectl get sc
```
You should see "openebs-device" already created by OpenEBS.


---

### Step 3: Create a StorageClass
Create a StorageClass that tells Kubernetes to use the OpenEBS provisioner to format raw disks.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: local.csi.openebs.io
parameters:
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF
```

---

### Step 4: Create a Persistent Volume Claim (PVC)
Now, request storage from this new StorageClass. OpenEBS will automatically find the blank `nvme1n1` drive on Worker 2, format it to `ext4`, and bind it.
OpenEBS will use the entire NVMe drive regardless of what you request. The size only matters for Kubernetes scheduling decisions.

### Quick test workflow:

**1. Create the PVC:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nvme-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-nvme
  resources:
    requests:
      storage: 10Mi
EOF
```

**2. Check status:**
```bash
kubectl get pvc nvme-pvc
```
Wait for `STATUS: Bound` (should take 10-30 seconds).

**3. Verify which drive it claimed:**
```bash
kubectl get pv
kubectl describe pv $(kubectl get pvc nvme-pvc -o jsonpath='{.spec.volumeName}')
```
Look for the `Path` field - it will show `/dev/nvme1n1` (or whatever the blank drive was).
