# Talos Linux — Kubernetes
### 1 Control Plane + 2 Workers | Static IPs
Before generating or applying your configs, you need to boot the node from the Talos ISO. Once it boots and gets a DHCP IP, run these commands from your local machine to find the exact disk and network interface names.

*(Replace `<DHCP_IP>` with the IP shown on the node's screen, e.g., `10.0.1.132`)*

### 1. Find the Network Interface
```bash
talosctl get links -n <DHCP_IP> --insecure
```
**What to look for:**
Look for the interface where `OPER STATE` is **`up`** and `LINK STATE` is **`true`**. 
*   Ignore `lo` (loopback).
*   The name will be something like `ens160`, `eth0`, or `enp2s0`. **Write this down.**

### 2. Find the Install Disk
```bash
talosctl get disks -n <DHCP_IP> --insecure
```
**What to look for:**
Look at the `ID` column to find your main hard drive/SSD. 
*   **Ignore `sr0`** (this is the CD-ROM/ISO drive).
*   The main disk will be something like `sda`, `vda`, or `nvme0n1`. **Write this down.**
*   *Note: You must prepend `/dev/` to this name in your config (e.g., `/dev/sda` or `/dev/nvme0n1`).*

### 3. Verify the Current IP and Routes (Optional but recommended)
```bash
talosctl get addresses -n <DHCP_IP> --insecure
```
**What to look for:**
This confirms the node actually received a DHCP IP and shows you the current subnet and gateway, which helps you plan your static IP.

---

### Summary of what you need to write down:
1.  **DHCP IP:** (e.g., `10.0.1.132`)
2.  **Network Interface:** (e.g., `ens160`)
3.  **Disk Name:** (e.g., `/dev/nvme0n1`)
4.  **Gateway IP:** (Usually your router, e.g., `10.0.1.2`)

---
Talos will automatically match the `HostnameConfig` block to the hostname document and the `machine:` block to the main network document.

Here is the cleanest way to do it.

### 1. Create the Combined Patch Files

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

*(Create `w2-patch.yaml` for Worker 2 by changing the hostname to `k8s-worker-2` and IP to `10.0.1.22/24`)*

### 2. Generate Base Configs (No patches needed here)
```bash
talosctl gen config my-cluster https://10.0.1.20:6443 \
  --install-disk /dev/nvme0n1 \
  --install-image ghcr.io/siderolabs/installer:v1.13.4 \
  --force
```

### 3. Apply the Single Patch File to Each Node

**Control Plane:**
```bash
talosctl apply-config --insecure -n 10.0.1.133 --file controlplane.yaml --config-patch @cp-patch.yaml
```

**Worker 1:**
```bash
talosctl apply-config --insecure -n <W1_DHCP_IP> --file worker.yaml --config-patch @w1-patch.yaml
```

**Worker 2:**
```bash
talosctl apply-config --insecure -n <W2_DHCP_IP> --file worker.yaml --config-patch @w2-patch.yaml
```
