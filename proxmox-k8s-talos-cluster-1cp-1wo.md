
---

# Proxmox: 1-Control, 1-Worker Talos Linux Cluster Setup

## Infrastructure Specifications (Proxmox VMs)

| Setting | Control Plane Node (`talos-cp-01`) | Worker Node (`talos-worker-01`) |
| --- | --- | --- |
| **Machine Type** | `q35`, Qemu Agent: Enabled | `q35`, Qemu Agent: Enabled |
| **Disks** | SCSI (VirtIO SCSI single), 20GB+ | SCSI (VirtIO SCSI single), 40GB+ |
| **CPU** | 2+ Cores (Type: `host`) | 2+ Cores (Type: `host`) |
| **Memory** | 2048 MB (Minimum) | 2048 MB+ |
| **Network** | VirtIO (Bridged to LAN) | VirtIO (Bridged to LAN) |

---

## Phase 1: Cluster Generation & Deployment

### 1. Generate Configuration Templates

Boot the Control Plane VM with the Talos ISO attached. Identify its temporary DHCP IP address from the Proxmox console (e.g., `192.168.1.50`). On your local management machine, execute:

```bash
# Generate the YAML configuration files
talosctl gen config my-cluster https://192.168.1.50:6443 --output-dir .

# Configure your local environment to target the Control Plane node
talosctl config node 192.168.1.50
talosctl config endpoints 192.168.1.50
export TALOSCONFIG=./talosconfig

```

### 2. Apply Configurations to Nodes

Apply the generated configurations to the respective machines. Talos will automatically install to the local virtual disk, unmount the ISO, and reboot.

```bash
# Control Plane Node
talosctl apply-config --insecure -n 192.168.1.50 --file controlplane.yaml

# Worker Node (Boot VM, note its DHCP IP, e.g., 192.168.1.51)
talosctl apply-config --insecure -n 192.168.1.51 --file worker.yaml

```

### 3. Bootstrap Kubernetes

Initialize the Kubernetes control plane. This must be executed **exactly once**:

```bash
talosctl bootstrap -n 192.168.1.50

```

To monitor progress, use the visual CLI dashboard:

```bash
talosctl dashboard -n 192.168.1.50

```

### 4. Fetch Administrative Kubeconfig

Once the cluster is healthy, download the `kubeconfig` to interact with it using standard tools like `kubectl`:

```bash
talosctl kubeconfig ./kubeconfig
export KUBECONFIG=./kubeconfig
kubectl get nodes

```

---

## Phase 2: Core Cluster Infrastructure (Networking & Storage)

### 1. Load Balancer: Kube-VIP (Layer 2 Implementation)

Kube-VIP acts as a minimalist, native alternative to MetalLB, provisioning Layer 2 LAN IPs for Kubernetes `LoadBalancer` services.

```bash
# Install the Cloud Provider Controller
kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip-cloud-provider/main/manifest/kube-vip-cloud-controller.yaml

```

Create a global configuration map containing a dedicated range of IPs outside of your home router's DHCP pool (e.g., `kube-vip-config.yaml`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevip-config
  namespace: kube-system
data:
  range-global: "192.168.1.210-192.168.1.220" # Adjust to match your network topology

```

```bash
# Apply IP range configuration
kubectl apply -f kube-vip-config.yaml

# Deploy Kube-VIP DaemonSet
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kube-vip/kube-vip/main/yaml/daemonset/kube-vip-arp.yaml

```

### 2. Persistent Storage: Longhorn (Distributed Block Storage)

Longhorn offers rich features, graphical volume management, and snapshot workflows. Because Talos is immutable, specific host mounts and iSCSI hooks are necessary.

#### Step A: Configure Machine Bind Mounts

Add this block under the main `machine:` layer in both `controlplane.yaml` and `worker.yaml`:

```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - rw
          - rshared

```

#### Step B: Activate Node iSCSI Daemons

Execute via `talosctl` to ensure the underlying block services are running across the cluster nodes:

```bash
talosctl -n 192.168.1.50 service iscsid restart
talosctl -n 192.168.1.51 service iscsid restart

```

#### Step C: Install via Helm

Since this is a single-worker cluster topology, specify a single default replica layer to prevent persistent volumes from getting stuck in a pending state.

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set persistence.defaultClassReplicaCount=1

```

---

## Core Architectural Takeaways

> 📌 **Architectural Note:**
> * **Networking:** Kube-VIP is selected over MetalLB for its zero-overhead architecture and native configuration capabilities inside the Talos OS engine.
> * **Storage:** Longhorn is preferred over TopoLVM for homelab scaling out of the box (built-in graphical UI, simple backup paths to S3/NFS). Replica limits must be set to `1` until additional worker nodes are added to the cluster mesh.
> 
>