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
# 💾Volume Setup- Talos Storage Preparation & OpenEBS

Because Talos is an immutable OS, you must explicitly allow Kubelet to mount host directories for `openebs-hostpath`, and label namespaces to allow OpenEBS to run privileged init pods.

### Step 1: Patch Talos Machine Config for HostPath
Run this to allow OpenEBS to write to `/var/openebs/local`. **Repeat for all 3 nodes** (Control Plane and both Workers).

```bash
cat <<EOF > openebs-mount-patch.yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/openebs/local
        type: bind
        source: /var/openebs/local
        options: [ bind, rshared, rw ]
EOF

# Apply to Control Plane
talosctl patch mc -n <CP_IP> --patch @openebs-mount-patch.yaml

# Apply to Workers
talosctl patch mc -n <W1_IP> --patch @openebs-mount-patch.yaml
talosctl patch mc -n <W2_IP> --patch @openebs-mount-patch.yaml
```
*(Wait ~30-60 seconds for the kubelets to restart on all nodes).*

### Step 2: Install OpenEBS
Install the OpenEBS Helm chart to provision both host-path (folders) and device (raw disks) storage.

```bash
helm repo add openebs https://openebs.github.io/charts
helm repo update

helm install openebs openebs/openebs --namespace openebs --create-namespace \
  --set localpv.enabled=true \
  --set localpv.device.enabled=true \
  --set openebs-ndm.enabled=true
```

### Step 3: Label Namespaces for Pod Security
Talos blocks privileged pods by default. We must allow them in `openebs` (for raw disk formatting).

```bash
  kubectl label --overwrite ns openebs \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/warn=privileged \
    pod-security.kubernetes.io/audit=privileged
```

### Step 4: Verify StorageClasses
OpenEBS automatically creates the required StorageClasses. You do **not** need to create them manually.

```bash
kubectl get sc
```
**Expected Output:**
*   **`openebs-hostpath`**: Creates folders on the node's root disk (Best for small apps like Grafana/Prometheus).
*   **`openebs-device`**: Formats and claims entire raw blank disks (Best for databases/large data on your extra NVMe).
