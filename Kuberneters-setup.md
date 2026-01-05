# Production-style shell-script–based setup to install a Kubernetes cluster on AWS EC2 Ubuntu with:

1 Control Plane (Master)

2 Worker Nodes

Kubernetes installed via scripts

containerd runtime

Calico CNI



##🧱 Architecture

EC2-1 → Control Plane (Master)

EC2-2 → Worker Node 1

EC2-3 → Worker Node 2

OS: Ubuntu 22.04

Instance type: t3.medium or higher (minimum 2 GB RAM)


🔑 Prerequisites (IMPORTANT)
✅ On ALL EC2 instances

Same VPC

Security Group allows:

SSH (22)

All traffic within SG

Hostnames set

Run as root user
sudo -i


📜 Script 1: Common Kubernetes Setup (Run on ALL Nodes)

Create file:

vi k8s-common.sh

Paste 👇

#!/bin/bash
set -e

echo "🚀 Disabling swap"
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo "📦 Installing dependencies"
apt update -y
apt install -y apt-transport-https ca-certificates curl gpg

echo "🐳 Installing containerd"
apt install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

echo "🔐 Loading kernel modules"
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

echo "🌐 Setting sysctl params"
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

echo "📥 Adding Kubernetes repo"
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
 | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| tee /etc/apt/sources.list.d/kubernetes.list

apt update -y

echo "📦 Installing kubeadm, kubelet, kubectl"
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

echo "✅ Common setup completed"

Run:

chmod +x k8s-common.sh
./k8s-common.sh


👉 Run this on master + both worker nodes

📜 Script 2: Initialize Control Plane (MASTER only)

Create:

vi master-init.sh

Paste 👇

#!/bin/bash
set -e

echo "🚀 Initializing Kubernetes Control Plane"
kubeadm init --pod-network-cidr=192.168.0.0/16

echo "📂 Configuring kubectl"
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

echo "🌐 Installing Calico CNI"
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

echo "✅ Master setup completed"


Run:

chmod +x master-init.sh
./master-init.sh


🔗 Join Worker Nodes to Cluster

After kubeadm init, you’ll see a join command like this:

kubeadm join <MASTER-IP>:6443 \
--token xxxxxx \
--discovery-token-ca-cert-hash sha256:xxxx

👉 Copy this command

👉 Run it on Node-1 and Node-2

✅ Verify Cluster (On Master)
kubectl get nodes


Expected output:

NAME        STATUS   ROLES           AGE   VERSION
master      Ready    control-plane   5m    v1.29.x
worker-1    Ready    <none>           2m    v1.29.x
worker-2    Ready    <none>           2m    v1.29.x
