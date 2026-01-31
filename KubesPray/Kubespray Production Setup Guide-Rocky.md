# Kubespray Setup – 3 Rocky Masters + 4 Windows Workers (Step by Step)

This is a **step-by-step production-ready guide** for deploying Kubernetes using Kubespray with:
- **3 Rocky Linux Master Nodes (HA + etcd + system workloads)**
- **4 Windows Worker Nodes (application workloads only)**

---

## 1. Final Architecture
```
+-----------------------------+
| Rocky Linux Control Plane   |
| (3 Masters: API, etcd)      |
+-----------------------------+
            |
            v
+-----------------------------------+
| Windows Worker Nodes (4 Nodes)     |
| (Application workloads only)       |
+-----------------------------------+
```

---

## 2. Important Rules
- Kubespray manages **Linux masters only**
- Windows nodes are joined **manually**
- System workloads (CoreDNS, CNI, ingress, metrics-server) **run on masters**
- Masters **must be schedulable** (`remove_master_taints: true`)

---

## 3. Supported Operating Systems
- **Masters:** Rocky Linux 8/9
- **Workers:** Windows Server 2019 / 2022

---

## 4. Hardware Requirements
| Role | CPU | RAM | Disk |
|------|-----|-----|------|
| Master | 4+ | 8+ GB | 60 GB |
| Windows Worker | 4+ | 8 GB | 60 GB |

---

## 5. Prepare Rocky Linux Masters
```bash
sudo dnf update -y
sudo dnf install -y git curl python3 python3-pip chrony
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
vm.max_map_count = 262144
EOF
sudo sysctl --system
```

---

## 6. Ansible Control Node Setup
```bash
sudo dnf install -y python3-pip git
pip3 install --user ansible ruamel.yaml
ansible --version
```

---

## 7. Download Kubespray
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
pip3 install --user -r requirements.txt
```

---

## 8. Create Inventory
```bash
cp -rfp inventory/sample inventory/prod
```

### Inventory (`hosts.yaml`) – Linux masters only
```yaml
all:
  hosts:
    master-1:
      ansible_host: 192.168.1.10
    master-2:
      ansible_host: 192.168.1.11
    master-3:
      ansible_host: 192.168.1.12

  children:
    kube_control_plane:
      hosts:
        master-1:
        master-2:
        master-3:

    etcd:
      hosts:
        master-1:
        master-2:
        master-3:

    k8s_cluster:
      children:
        kube_control_plane:
```

---

## 9. Cluster Configuration (`k8s-cluster.yml`)
```yaml
kube_version: v1.29.0
kube_network_plugin: calico
kube_pods_subnet: 10.244.0.0/16
kube_service_addresses: 10.96.0.0/12
dns_mode: coredns
container_manager: containerd
containerd_version: 1.7.13
metrics_server_enabled: true
ingress_nginx_enabled: true
helm_enabled: true
remove_master_taints: true
```

---

## 10. Deploy Cluster
```bash
ansible-playbook -i inventory/prod/hosts.yaml \
  --become --become-user=root cluster.yml
```

---

## 11. Configure kubectl
```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
kubectl get nodes
```

---

## 12. Windows Worker Node Setup

### 12.1 Prepare Windows
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName containers -All
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Restart-Computer
```

### 12.2 Install containerd
```powershell
mkdir C:\containerd
cd C:\containerd
curl.exe -L https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-windows-amd64.tar.gz -o containerd.tar.gz
tar -xvf containerd.tar.gz
```

### 12.3 Install kubeadm, kubelet, kubectl
```powershell
curl.exe -L https://dl.k8s.io/v1.29.0/bin/windows/amd64/kubeadm.exe -o kubeadm.exe
curl.exe -L https://dl.k8s.io/v1.29.0/bin/windows/amd64/kubelet.exe -o kubelet.exe
curl.exe -L https://dl.k8s.io/v1.29.0/bin/windows/amd64/kubectl.exe -o kubectl.exe
setx PATH "$env:PATH;C:\k"
```

### 12.4 Join Windows Nodes
On a master:
```bash
kubeadm token create --print-join-command
```
On Windows:
```powershell
kubeadm join <API-VIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

### 12.5 Label Windows Nodes
```bash
kubectl label node win-worker-1 kubernetes.io/os=windows
kubectl label node win-worker-2 kubernetes.io/os=windows
kubectl label node win-worker-3 kubernetes.io/os=windows
kubectl label node win-worker-4 kubernetes.io/os=windows
```

---

## 13. Scheduling Rules
### Linux system workloads (system components on masters only)
```yaml
nodeSelector:
  kubernetes.io/os: linux
```
### Windows application workloads
```yaml
nodeSelector:
  kubernetes.io/os: windows
```

---

## 14. Production Checklist
✅ 3 masters (HA) with system workloads
✅ etcd quorum healthy
✅ NO Linux worker nodes
✅ Windows workers isolated for applications
✅ Monitoring and ingress on masters
✅ Masters properly sized to run all system pods

---

## 15. Notes
- Always ensure **masters have enough resources** to run control plane + system pods
- Upgrade nodes **carefully, one at a time**
- Backup etcd regularly
- Windows nodes run **applications only**

✅ **Cluster is production-ready with this configuration.**

