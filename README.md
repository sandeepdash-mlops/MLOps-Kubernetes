<div align="center">
<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0d1117,50:1a3a2a,100:16a34a&height=220&section=header&text=Devops-K8s&fontSize=90&fontColor=ffffff&fontAlignY=38&desc=Hybrid%20Multi-Region%20HA%20Kubernetes%20Cluster&descAlignY=66&descSize=22&descColor=86efac" alt="Devops-K8s Banner"/>

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.32-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Rocky Linux](https://img.shields.io/badge/Rocky_Linux-9.x-10B981?style=for-the-badge&logo=rockylinux&logoColor=white)](https://rockylinux.org/)
[![Cilium](https://img.shields.io/badge/Cilium-CNI-F8C517?style=for-the-badge&logo=cilium&logoColor=black)](https://cilium.io/)
[![Helm](https://img.shields.io/badge/Helm-3-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![HAProxy](https://img.shields.io/badge/HAProxy-Load_Balancer-FF6600?style=for-the-badge)](https://www.haproxy.org/)
[![etcd](https://img.shields.io/badge/etcd-Quorum-419EDA?style=for-the-badge)](https://etcd.io/)
[![CRI-O](https://img.shields.io/badge/CRI--O-Runtime-CC0000?style=for-the-badge)](https://cri-o.io/)
[![NFS](https://img.shields.io/badge/NFS-Persistent_Storage-6B7280?style=for-the-badge)](https://kubernetes.io/docs/concepts/storage/)

<br/>

> **End-to-end automation for a production-grade, highly available Kubernetes cluster that spans multiple regions, public cloud providers, and bank private on-prem networks. A Keepalived floating IP serves as the control plane advertise address, fronted by two HAProxy load balancers for failover. The setup covers etcd quorum management, persistent storage, and RBAC, with dedicated GPU instance pools for deploying ML applications, all designed for security and compliance in regulated hybrid enterprise environments.**

</div>

---

## 📌 What This Project Solves

Managing a **highly available Kubernetes cluster across multiple networks, regions, and cloud providers** is one of the hardest real-world infrastructure challenges. This repo documents and scripts the entire lifecycle — from bootstrapping nodes to recovering a broken etcd quorum — with production-grade tooling on **Rocky Linux 9**.

---

## 🏗️ Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │           HAProxy Load Balancer              │
                        │         stage-proxy-host (10.10.1.36)        │
                        │              Public IP + VPN                 │
                        └──────────────┬──────────────────────────────┘
                                       │ :6443
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
   ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌─────────▼───────────┐
   │  Control Plane 1    │  │  Control Plane 2    │  │  Control Plane 3   │
   │  isu-k8s-cp         │  │  stpi-k8s-master    │  │  gcp-k8s-master    │
   │  172.18.0.101       │  │  172.26.0.8         │  │  10.10.1.26        │
   │  (On-Prem / ISU)    │  │  (On-Prem / STPI)   │  │  (Google Cloud)    │
   └──────────┬──────────┘  └──────────┬──────────┘  └─────────┬──────────┘
              │                        │                        │
              └────────────────────────┼────────────────────────┘
                                       │  etcd quorum (cross-VPN)
                        ┌──────────────▼──────────────┐
                        │        Worker Nodes          │
                        │   (Multiple regions/zones)   │
                        └──────────────────────────────┘
```

> ⚡ All control planes are on **different networks**, connected via **VPN** — making this a true cross-network, hybrid HA setup.

---

## ✨ Key Features

| Feature | Detail |
|---|---|
| 🌐 **Multi-Region Control Plane** | 3 control planes across on-prem (ISU, STPI) and GCP — connected over VPN |
| ⚖️ **HAProxy Load Balancing** | TCP-mode round-robin with health checks on all API servers |
| 🧠 **etcd Quorum Management** | Handles cross-network etcd peer certs (SANs), stale member removal, clean rejoin |
| 🔒 **RBAC Automation** | Scripts for Linux user creation, Kubernetes user binding, and kubeconfig distribution |
| 📦 **Persistent Storage** | Supports both NFS (`nfs-subdir-external-provisioner`) and local-path provisioner |
| 🌿 **Cilium CNI** | eBPF-powered networking with kube-proxy replacement, Hubble observability, Gateway API |
| 🔁 **Full Recovery Playbooks** | Step-by-step recovery for etcd split-brain and hostname-change failures |
| 📖 **Production Reference Guide** | Complete manual setup guide with real commands, real errors, and real fixes |

---

## 📁 Repository Structure

```
Devops-K8s/
├── hybrid-multi-region-ha-cluster-setup.txt  # 📖 Complete manual setup reference guide
├── slave-nodes.sh                            # 👷 Worker node join helper script
├── RBAC-Kubernetes/
│   ├── create-linux-user.sh                  # 👤 Create OS-level user on node
│   ├── create-k8s-user.sh                    # 🔑 K8s user with cert-based auth + role binding
│   ├── move-kubeconfig.sh                    # 📤 Distribute kubeconfig to user
│   └── steps-to-execute.txt                  # 📋 RBAC execution order guide
└── README.md
```

---

## 🛠️ Setup Guide

The full step-by-step manual setup is documented in **`hybrid-multi-region-ha-cluster-setup.txt`**.

Below is a high-level summary of the setup phases:

### Phase 1 — HAProxy Node

```bash
dnf install haproxy -y
# Configure /etc/haproxy/haproxy.cfg with all 3 control-plane backends
# Add local DNS entry in /etc/hosts pointing to HAProxy internal IP
haproxy -c -V -f /etc/haproxy/haproxy.cfg     # validate config
systemctl enable --now haproxy.service
```

### Phase 2 — All Nodes (Prerequisites)

```bash
# Set hostname, /etc/hosts, disable swap, SELinux permissive
# Load kernel modules: overlay, br_netfilter
# Apply sysctl params for bridged traffic
# Install CRI-O container runtime
# Install kubelet, kubeadm, kubectl
```

### Phase 3 — Initialize First Control-Plane

```bash
sudo kubeadm init \
  --skip-phases=addon/kube-proxy \
  --control-plane-endpoint "stage.proxyhost.lan:6443" \
  --upload-certs
```

### Phase 4 — Join Additional Control-Planes & Workers

```bash
# Control-plane join (cp-2, cp-3)
kubeadm join stage.proxyhost.lan:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <hash> \
  --control-plane \
  --certificate-key <cert-key>

# Worker node join
kubeadm join stage.proxyhost.lan:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <hash>
```

### Phase 5 — Cilium CNI

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.18.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<CONTROL_PLANE_IP> \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true
```

### Phase 6 — Persistent Storage

```bash
# Local path provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# OR NFS provisioner via Helm
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=<NFS_SERVER_IP> \
  --set nfs.path=/nas \
  --set storageClass.name=stage-storage
```

> 📖 For complete commands, configs, and troubleshooting — refer to **`hybrid-multi-region-ha-cluster-setup.txt`**

---

## 🔒 RBAC — Kubernetes User Management

The `RBAC-Kubernetes/` module handles the full lifecycle of granting a developer or operator scoped cluster access.

```bash
# Step 1 — Create Linux user on the node
bash RBAC-Kubernetes/create-linux-user.sh <username>

# Step 2 — Create K8s user with certificate-based auth + role binding
bash RBAC-Kubernetes/create-k8s-user.sh <username> <namespace> <role>

# Step 3 — Deliver kubeconfig to the user
bash RBAC-Kubernetes/move-kubeconfig.sh <username>
```

> See `RBAC-Kubernetes/steps-to-execute.txt` for the full execution order and examples.

---

## 🔁 Failure Recovery Playbooks

### Case 1 — Stale etcd Member (Control-Plane Rejoin)

> When a control-plane is rebuilt or re-provisioned, the old etcd membership blocks the rejoin with `error: etcd cluster is not healthy`.

```bash
# Step 1 — On a healthy CP, list etcd members
kubectl -n kube-system exec -it etcd-<healthy-cp> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list

# Step 2 — Remove the stale member using its hex ID
etcdctl ... member remove <STALE_MEMBER_HEX_ID>

# Step 3 — On the broken node, clean state and rejoin
systemctl stop kubelet
rm -rf /etc/kubernetes /var/lib/etcd /var/lib/kubelet
kubeadm join stage.proxyhost.lan:6443 --token <token> \
  --discovery-token-ca-cert-hash <hash> \
  --control-plane --certificate-key <cert-key>
```

### Case 2 — Hostname Changed on Worker Node

> Kubelet fails with `node "old-name" not found` after a hostname change.

```bash
# Stop services and clean state
systemctl stop kubelet && systemctl stop crio
rm -rf /etc/kubernetes /var/lib/etcd /var/lib/kubelet

# Update /etc/hosts with the new hostname
# Generate a fresh join token from any healthy CP
kubeadm token create --print-join-command

# Rejoin with the new hostname
bash slave-nodes.sh
```

---

## 📐 Design Decisions

**Why HAProxy over a cloud LB?**
This cluster spans on-prem and cloud nodes — a cloud-native LB can't front all three. HAProxy runs on a dedicated proxy node with VPN access to all control-planes, making it the only viable cross-network solution.

**Why Cilium over flannel/calico?**
Cilium replaces kube-proxy entirely using eBPF, delivering better performance and built-in observability via Hubble — essential for cross-region traffic visibility.

**Why CRI-O over containerd/Docker?**
CRI-O is purpose-built for Kubernetes, lighter weight, and better aligned with OCI standards — reducing attack surface in compliance-sensitive environments.

---

## 💬 Connect

<div align="center">

📧 **Email:** sandeepdashmlops@gmail.com
&nbsp;&nbsp;|&nbsp;&nbsp;
💻 **GitHub:** [github.com/sandeepdash-mlops](https://github.com/sandeepdash-mlops)

</div>

---

<div align="center">
<img src="https://capsule-render.vercel.app/api?type=waving&color=0:16a34a,50:1a3a2a,100:0d1117&height=120&section=footer" alt="footer"/>

*Real infrastructure. Real problems. Real fixes.*

</div>
