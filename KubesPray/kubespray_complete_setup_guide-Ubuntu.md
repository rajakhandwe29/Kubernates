# Kubespray – Complete End‑to‑End Setup Documentation

This document is a **full, practical guide** to installing a production‑ready Kubernetes cluster using **Kubespray**.

It is written so that you can:
- Follow it step by step
- Copy/paste commands
- Understand *why* each step exists

---

## 1. What is Kubespray?

**Kubespray** is an open‑source project that deploys Kubernetes clusters using **Ansible**.

Internally it uses:
- kubeadm
- containerd / CRI‑O
- Ansible roles and playbooks

### When to use Kubespray
- Multi‑node clusters
- Production or staging environments
- Bare metal or VM‑based clusters
- Need repeatability, upgrades, HA

### When NOT to use Kubespray
- Local laptop testing → minikube / kind / Docker Desktop
- Single‑node learning setups

---

## 2. Architecture Overview

```
Ansible Control Node
        |
        | (SSH + Ansible)
        v
+-------------------+
| Control Plane     |
| (API, etcd)       |
+-------------------+
        |
        v
+-------------------+
| Worker Nodes      |
| (kubelet, pods)   |
+-------------------+
```

---

## 3. Prerequisites

### 3.1 Hardware Requirements

| Role | CPU | RAM | Disk |
|----|----|----|----|
| Control Plane | 2 | 4 GB | 40 GB |
| Worker | 2 | 4 GB | 40 GB |

### 3.2 Operating System

Supported (recommended):
- Ubuntu 20.04 / 22.04
- CentOS Stream 8/9
- Rocky Linux

⚠️ All nodes **must use the same OS version**

---

## 4. Network Requirements

- All nodes can reach each other
- Required ports open (or firewalls disabled initially)

Important ports:
- 6443 (Kubernetes API)
- 2379–2380 (etcd)
- 10250–10255 (kubelet)

---

## 5. Ansible Control Node Setup

### 5.1 Install Dependencies

```bash
sudo apt update
sudo apt install -y python3 python3-pip git
pip3 install --user ansible
```

Verify:
```bash
ansible --version
```

---

## 6. Download Kubespray

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

Install Python requirements:
```bash
pip3 install --user -r requirements.txt
```

---

## 7. Inventory Creation

### 7.1 Create Inventory Folder

```bash
cp -rfp inventory/sample inventory/mycluster
```

### 7.2 Generate Inventory Automatically

```bash
pip3 install --user ruamel.yaml

declare -a IPS=(192.168.1.10 192.168.1.11 192.168.1.12)
CONFIG_FILE=inventory/mycluster/hosts.yaml \
python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

---

## 8. Inventory File (hosts.yaml)

```yaml
all:
  hosts:
    master-1:
      ansible_host: 192.168.1.10
      ip: 192.168.1.10
      access_ip: 192.168.1.10

    worker-1:
      ansible_host: 192.168.1.11
      ip: 192.168.1.11

    worker-2:
      ansible_host: 192.168.1.12
      ip: 192.168.1.12

  children:
    kube_control_plane:
      hosts:
        master-1:

    kube_node:
      hosts:
        worker-1:
        worker-2:

    etcd:
      hosts:
        master-1:

    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
```

---

## 9. Cluster Configuration

### 9.1 Kubernetes Version & Network

**File:** `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`

```yaml
kube_version: v1.29.0
kube_network_plugin: calico
kube_pods_subnet: 10.244.0.0/16
kube_service_addresses: 10.96.0.0/12
dns_mode: coredns
```

---

## 10. Container Runtime Configuration

**File:** `inventory/mycluster/group_vars/all/containerd.yml`

```yaml
container_manager: containerd
containerd_version: 1.7.13
```

---

## 11. Add-ons Configuration

**File:** `inventory/mycluster/group_vars/k8s_cluster/addons.yml`

```yaml
dashboard_enabled: true
metrics_server_enabled: true
ingress_nginx_enabled: true
helm_enabled: true
```

---

## 12. Ansible Global Settings

**File:** `inventory/mycluster/group_vars/all/all.yml`

```yaml
ansible_user: ubuntu
ansible_become: true
swap_file_state: absent
```

---

## 13. SSH Connectivity Test

```bash
ansible -i inventory/mycluster/hosts.yaml all -m ping
```

If this fails, **do not proceed**.

---

## 14. Deploy Kubernetes Cluster

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  --become --become-user=root \
  cluster.yml
```

Expected time: **15–30 minutes**

---

## 15. Configure kubectl Access

On control plane node:

```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Verify:
```bash
kubectl get nodes
kubectl get pods -A
```

---

## 16. Day‑2 Operations

### Add Node
- Add node IP to inventory
- Re-run `cluster.yml`

### Upgrade Cluster
```bash
ansible-playbook upgrade-cluster.yml
```

### Reset Cluster (DESTRUCTIVE)
```bash
ansible-playbook reset.yml
```

---

## 17. Common Errors & Fixes

| Problem | Fix |
|------|----|
| SSH unreachable | Fix key / user |
| Swap enabled | Disable swap |
| Pods stuck Pending | Check CNI |
| etcd failures | Time sync / ports |

---

## 18. Best Practices

- Use **3 control plane nodes** for production
- Use **separate etcd nodes** for large clusters
- Enable monitoring (Prometheus)
- Backup etcd regularly

---

## 19. Useful Commands

```bash
kubectl get nodes
kubectl describe node <node>
kubectl logs -n kube-system <pod>
```

---

## 20. Official Resources

- Kubespray GitHub
- Kubernetes Documentation
- Ansible Documentation

---

✅ You now have a **complete Kubespray setup guide** suitable for production use.

