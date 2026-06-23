# Talos Linux — Kubernetes
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
