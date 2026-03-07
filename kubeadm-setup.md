<div align="center">

# Kubernetes Multi-Node Cluster Setup Using kubeadm
## Google Cloud Platform (GCP) VM Deployment

<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/googlecloud/googlecloud-original.svg" width="110"/>
&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/kubernetes/kubernetes-plain.svg" width="110"/>
&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/docker/docker-original.svg" width="110"/>
&nbsp;&nbsp;
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/linux/linux-original.svg" width="110"/>

<br/>

###  Google Cloud •  Kubernetes •  containerd •  Ubuntu

</div>

---

#  Overview

This guide demonstrates how to deploy a **3-Node Kubernetes Cluster** using **kubeadm** on **Google Cloud Platform (GCP) Virtual Machines**.

The cluster consists of:

- **1 Control Plane Node**
- **2 Worker Nodes**
- **Container Runtime:** containerd
- **Network Plugin:** Calico
- **Metrics Monitoring:** Metrics Server

---

#  Cluster Architecture

| Node | Role | Internal IP |
|-----|-----|-----|
| k8s-master | Control Plane | 10.128.0.4 |
| k8s-worker-1 | Worker | 10.128.0.5 |
| k8s-worker-2 | Worker | 10.128.0.6 |

---

#  Step 1 — Create Virtual Machines

Create **3 Ubuntu virtual machines** in Google Cloud.

### Recommended Configuration

- OS: Ubuntu **22.04 LTS**
- Machine Type: **e2-medium**
- Disk: **20 GB**
- Network: **Default VPC**

### Instance Names

```
k8s-master
k8s-worker-1
k8s-worker-2
```

---

#  Step 2 — Configure Firewall Rules

Kubernetes nodes must communicate internally.

Create a firewall rule allowing traffic between nodes.

| Protocol | Port | Purpose |
|--------|--------|--------|
| TCP | 6443 | Kubernetes API |
| TCP | 2379-2380 | etcd |
| TCP | 10250 | kubelet |
| TCP | 179 | Calico BGP |
| UDP | 4789 | VXLAN |
| Other | 4 | Calico IPIP |

### Source Range

```
10.128.0.0/9
```

---

#  Step 3 — Clean Previous Kubernetes Installation (Optional)

Run on **all nodes**.

### Reset kubeadm

```bash
sudo kubeadm reset -f
```

### Remove old network files

```bash
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/kubelet
sudo rm -rf /etc/kubernetes
sudo rm -rf ~/.kube
```

### Flush iptables

```bash
sudo iptables -F
```

```bash
sudo iptables -t nat -F
```

```bash
sudo iptables -t mangle -F
```

```bash
sudo iptables -X
```

### Reboot node

```bash
sudo reboot
```

---

#  Step 4 — Disable Swap

Kubernetes requires swap to be disabled.

### Disable swap

```bash
sudo swapoff -a
```

### Make swap permanently disabled

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Verify

```bash
free -h
```

Swap should display:

```
0B
```

---

#  Step 5 — Load Kernel Modules

### Create configuration file

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

### Load modules

```bash
sudo modprobe overlay
```

```bash
sudo modprobe br_netfilter
```

### Verify modules

```bash
lsmod | grep overlay
```

```bash
lsmod | grep br_netfilter
```

---

#  Step 6 — Configure Sysctl Networking

### Create sysctl configuration

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

### Apply settings

```bash
sudo sysctl --system
```

### Verify

```bash
sysctl net.ipv4.ip_forward
```

Expected output:

```
net.ipv4.ip_forward = 1
```

---

# Step 7 — Install containerd

### Update system

```bash
sudo apt update
```

```bash
sudo apt install -y ca-certificates curl
```

### Add Docker repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Add repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install containerd

```bash
sudo apt update
```

```bash
sudo apt install -y containerd.io
```

---

# ⚙ Step 8 — Configure containerd

### Generate default configuration

```bash
sudo mkdir -p /etc/containerd
```

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

### Enable systemd cgroup

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### Restart containerd

```bash
sudo systemctl restart containerd
```

```bash
sudo systemctl enable containerd
```

### Verify

```bash
sudo systemctl status containerd
```

---

# Step 9 — Install Kubernetes Components

### Install dependencies

```bash
sudo apt update
```

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

### Add Kubernetes key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Add Kubernetes repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes tools

```bash
sudo apt update
```

```bash
sudo apt install -y kubelet kubeadm kubectl
```

### Lock versions

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# Step 10 — Initialize Kubernetes Control Plane

Run only on **master node**.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

After completion a **join command** will appear for worker nodes.

---

# Step 11 — Configure kubectl

Run on **master node**.

```bash
mkdir -p $HOME/.kube
```

```bash
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
```

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Verify

```bash
kubectl get nodes
```

Initially the master node may appear:

```
NotReady
```

---

# Step 12 — Install Calico Network Plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

### Verify

```bash
kubectl get pods -n kube-system
```

Wait until **Calico pods are Running**.

---

# Step 13 — Join Worker Nodes

Run on **worker nodes** using the command generated earlier.

Example:

```bash
sudo kubeadm join 10.128.0.4:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

---

# Step 14 — Verify Cluster

Run on **master node**.

```bash
kubectl get nodes
```

Expected output:

```
k8s-master     Ready
k8s-worker-1   Ready
k8s-worker-2   Ready
```

---

# Step 15 — Install Metrics Server

### Apply metrics server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Patch deployment

```bash
kubectl patch deployment metrics-server -n kube-system \
--type='json' \
-p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

### Verify

```bash
kubectl top nodes
```

Example output:

```
NAME           CPU(cores)   MEMORY(bytes)
k8s-master     110m         1050Mi
k8s-worker-1   90m          890Mi
k8s-worker-2   80m          870Mi
```

---

# Final Cluster Status

| Component | Status |
|--------|--------|
| Kubernetes Control Plane | Running |
| Worker Nodes | Connected |
| Calico Networking | Running |
| CoreDNS | Running |
| Metrics Server | Running |

---

# Author

**Md Majid**  
DevOps | Kubernetes | Cloud Infrastructure  
