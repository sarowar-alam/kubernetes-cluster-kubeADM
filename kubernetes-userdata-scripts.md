# Kubernetes Cluster Setup with AWS EC2 User Data
## Automated Installation for Ubuntu 24.04 LTS

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [User Data Script - Control Plane](#user-data-script---control-plane)
4. [User Data Script - Worker Nodes](#user-data-script---worker-nodes)
5. [Deployment Instructions](#deployment-instructions)
6. [Post-Deployment Steps](#post-deployment-steps)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)
9. [Advanced Configurations](#advanced-configurations)

---

## Overview

**What are AWS User Data Scripts?**

User Data scripts are bash/shell scripts that run automatically when an EC2 instance launches for the first time. They're perfect for automating infrastructure setup.

**Real-World Analogy:** Think of User Data as a "setup wizard" that runs automatically when you unbox a new computer. By the time you log in, everything is already installed and configured.

**This Guide Provides:**
- ✅ Fully automated Control Plane setup
- ✅ Fully automated Worker Node setup
- ✅ All prerequisites configured automatically
- ✅ containerd, kubeadm, kubelet, kubectl installed
- ✅ Cluster initialization (Control Plane only)
- ✅ Calico CNI installation (Control Plane only)
- ✅ Production-ready configurations

**Cluster Architecture:**
```
┌─────────────────────────────────────────────────────────────┐
│                         AWS VPC                             │
│                                                             │
│  ┌──────────────────────┐                                  │
│  │  Control Plane EC2   │                                  │
│  │  - User Data Script  │                                  │
│  │  - Auto-configures   │                                  │
│  │  - Initializes K8s   │                                  │
│  │  - Installs Calico   │                                  │
│  └──────────┬───────────┘                                  │
│             │                                               │
│             │ Join Token Generated                          │
│             │                                               │
│  ┌──────────▼───────────┐      ┌────────────────────┐     │
│  │  Worker Node 1 EC2   │      │  Worker Node 2 EC2 │     │
│  │  - User Data Script  │      │  - User Data Script│     │
│  │  - Auto-configures   │      │  - Auto-configures │     │
│  │  - Ready to join     │      │  - Ready to join   │     │
│  └──────────────────────┘      └────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### AWS Requirements

**1. EC2 Instance Specifications:**

| Component       | Instance Type | vCPU | RAM  | Storage |
|----------------|---------------|------|------|---------|
| Control Plane  | t3.medium     | 2    | 4 GB | 20 GB   |
| Worker Node    | t3.small      | 2    | 2 GB | 20 GB   |

**Note:** t3.medium recommended for Control Plane. t2.medium works but t3 has better burst credits.

**2. AMI:**
- **Ubuntu Server 24.04 LTS**
- **Architecture:** 64-bit (x86)

**3. Security Group Configuration:**

Create a security group named `k8s-cluster-sg` with these rules:

**Inbound Rules:**
```
Port Range    | Protocol | Source                | Description
--------------|----------|----------------------|----------------------------
22            | TCP      | Your IP/0.0.0.0/0    | SSH access
6443          | TCP      | Security Group ID    | Kubernetes API Server
2379-2380     | TCP      | Security Group ID    | etcd server client API
10250         | TCP      | Security Group ID    | Kubelet API
10259         | TCP      | Security Group ID    | kube-scheduler
10257         | TCP      | Security Group ID    | kube-controller-manager
179           | TCP      | Security Group ID    | Calico BGP
4789          | UDP      | Security Group ID    | Calico VXLAN
30000-32767   | TCP      | 0.0.0.0/0           | NodePort Services (optional)
```

**Outbound Rules:**
```
All traffic   | All      | 0.0.0.0/0           | Allow all outbound
```

**4. IAM Role (Optional but Recommended):**
- Attach `AmazonSSMManagedInstanceCore` for Systems Manager access
- Useful for troubleshooting without SSH

**5. Network Configuration:**
- Use **VPC with public subnets** (or private with NAT gateway)
- **Auto-assign Public IP:** Enabled
- All instances in the **same subnet** for simplicity

---

## User Data Script - Control Plane

**When to use:** Launch this script on your Control Plane (Master) node.

**What it does:**
1. Updates system packages
2. Disables swap permanently
3. Loads required kernel modules
4. Configures sysctl settings
5. Installs containerd with proper configuration
6. Installs kubeadm, kubelet, kubectl
7. Initializes Kubernetes cluster
8. Configures kubectl for ubuntu user
9. Installs Calico CNI
10. Outputs join command to file

### 📝 Control Plane User Data Script

```bash
#!/bin/bash

#############################################################
# Kubernetes Control Plane Setup - User Data Script
# OS: Ubuntu 24.04 LTS
# Container Runtime: containerd
# CNI: Calico
# Created: 2026
#############################################################

set -e  # Exit on any error

# Log all output to file for debugging
exec > >(tee /var/log/k8s-control-plane-setup.log)
exec 2>&1

echo "=========================================="
echo "Starting Kubernetes Control Plane Setup"
echo "Time: $(date)"
echo "=========================================="

#############################################################
# Phase 1: System Preparation
#############################################################

echo "[Phase 1] Updating system packages..."
apt-get update
apt-get upgrade -y

echo "[Phase 1] Setting hostname..."
hostnamectl set-hostname k8s-control-plane

# Add hostname to /etc/hosts
echo "127.0.1.1 k8s-control-plane" >> /etc/hosts

echo "[Phase 1] Disabling swap..."
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "[Phase 1] Loading kernel modules..."
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

echo "[Phase 1] Configuring sysctl..."
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

#############################################################
# Phase 2: Install containerd
#############################################################

echo "[Phase 2] Installing containerd prerequisites..."
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

echo "[Phase 2] Adding Docker repository..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update

echo "[Phase 2] Installing containerd..."
apt-get install -y containerd.io

echo "[Phase 2] Configuring containerd..."
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# Enable SystemdCgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd

#############################################################
# Phase 3: Install Kubernetes Components
#############################################################

echo "[Phase 3] Adding Kubernetes repository..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

echo "[Phase 3] Installing kubeadm, kubelet, kubectl..."
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet

#############################################################
# Phase 4: Initialize Kubernetes Cluster
#############################################################

echo "[Phase 4] Initializing Kubernetes cluster..."
echo "This may take 3-5 minutes..."

# Get private IP of this instance
PRIVATE_IP=$(hostname -I | awk '{print $1}')

kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=$PRIVATE_IP \
  --control-plane-endpoint=$PRIVATE_IP \
  --v=5 | tee /var/log/kubeadm-init.log

#############################################################
# Phase 5: Configure kubectl
#############################################################

echo "[Phase 5] Configuring kubectl for root user..."
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown root:root /root/.kube/config

echo "[Phase 5] Configuring kubectl for ubuntu user..."
mkdir -p /home/ubuntu/.kube
cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config
chown ubuntu:ubuntu /home/ubuntu/.kube/config

# Add kubectl bash completion
echo 'source <(kubectl completion bash)' >> /home/ubuntu/.bashrc
echo 'alias k=kubectl' >> /home/ubuntu/.bashrc
echo 'complete -o default -F __start_kubectl k' >> /home/ubuntu/.bashrc

#############################################################
# Phase 6: Install Calico CNI
#############################################################

echo "[Phase 6] Installing Calico CNI..."

# Wait for API server to be fully ready
echo "Waiting for API server to be ready..."
sleep 30

kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

#############################################################
# Phase 7: Generate and Save Join Command
#############################################################

echo "[Phase 7] Generating worker node join command..."
sleep 10

kubeadm token create --print-join-command > /home/ubuntu/kubeadm-join-command.sh
chmod +x /home/ubuntu/kubeadm-join-command.sh
chown ubuntu:ubuntu /home/ubuntu/kubeadm-join-command.sh

# Also save to root
cp /home/ubuntu/kubeadm-join-command.sh /root/kubeadm-join-command.sh

# Display join command in log
echo ""
echo "=========================================="
echo "WORKER NODE JOIN COMMAND:"
echo "=========================================="
cat /home/ubuntu/kubeadm-join-command.sh
echo "=========================================="

#############################################################
# Phase 8: Final Verification
#############################################################

echo "[Phase 8] Waiting for cluster to be ready..."
sleep 60

echo "Cluster status:"
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes

echo "Cluster pods:"
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods --all-namespaces

#############################################################
# Setup Complete
#############################################################

echo ""
echo "=========================================="
echo "Kubernetes Control Plane Setup COMPLETE!"
echo "Time: $(date)"
echo "=========================================="
echo ""
echo "Join command saved to: /home/ubuntu/kubeadm-join-command.sh"
echo ""
echo "To access cluster:"
echo "  ssh ubuntu@<this-instance-ip>"
echo "  kubectl get nodes"
echo ""
echo "To get join command:"
echo "  cat /home/ubuntu/kubeadm-join-command.sh"
echo ""
echo "Full log available at: /var/log/k8s-control-plane-setup.log"
echo "=========================================="

# Create status file
cat <<EOF > /home/ubuntu/cluster-info.txt
Kubernetes Control Plane Setup Complete
Setup Time: $(date)
Kubernetes Version: $(kubectl version --short 2>/dev/null || echo "Check manually")
Control Plane IP: $PRIVATE_IP

Join Command:
$(cat /home/ubuntu/kubeadm-join-command.sh)

To view cluster:
  kubectl get nodes
  kubectl get pods --all-namespaces

To get join command again:
  kubeadm token create --print-join-command
EOF

chown ubuntu:ubuntu /home/ubuntu/cluster-info.txt

echo "Setup complete! You can now join worker nodes."
```

---

## User Data Script - Worker Nodes

**When to use:** Launch this script on your Worker nodes.

**What it does:**
1. Updates system packages
2. Disables swap permanently
3. Loads required kernel modules
4. Configures sysctl settings
5. Installs containerd with proper configuration
6. Installs kubeadm, kubelet, kubectl
7. Prepares node to join cluster (manual join required)

**Note:** Worker nodes are prepared but NOT automatically joined. You must manually run the join command after Control Plane is ready.

### 📝 Worker Node User Data Script

```bash
#!/bin/bash

#############################################################
# Kubernetes Worker Node Setup - User Data Script
# OS: Ubuntu 24.04 LTS
# Container Runtime: containerd
# Created: 2026
#############################################################

set -e  # Exit on any error

# Log all output to file for debugging
exec > >(tee /var/log/k8s-worker-setup.log)
exec 2>&1

echo "=========================================="
echo "Starting Kubernetes Worker Node Setup"
echo "Time: $(date)"
echo "=========================================="

#############################################################
# Phase 1: System Preparation
#############################################################

echo "[Phase 1] Updating system packages..."
apt-get update
apt-get upgrade -y

echo "[Phase 1] Setting hostname..."
# Generate unique hostname based on instance ID
INSTANCE_ID=$(ec2-metadata --instance-id | cut -d " " -f 2)
hostnamectl set-hostname k8s-worker-${INSTANCE_ID}

# Add hostname to /etc/hosts
echo "127.0.1.1 k8s-worker-${INSTANCE_ID}" >> /etc/hosts

echo "[Phase 1] Disabling swap..."
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "[Phase 1] Loading kernel modules..."
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

echo "[Phase 1] Configuring sysctl..."
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

#############################################################
# Phase 2: Install containerd
#############################################################

echo "[Phase 2] Installing containerd prerequisites..."
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

echo "[Phase 2] Adding Docker repository..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update

echo "[Phase 2] Installing containerd..."
apt-get install -y containerd.io

echo "[Phase 2] Configuring containerd..."
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# Enable SystemdCgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd

#############################################################
# Phase 3: Install Kubernetes Components
#############################################################

echo "[Phase 3] Adding Kubernetes repository..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

echo "[Phase 3] Installing kubeadm, kubelet, kubectl..."
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet

#############################################################
# Phase 4: Prepare for Joining Cluster
#############################################################

echo "[Phase 4] Worker node setup complete. Ready to join cluster."

# Create instruction file for ubuntu user
cat <<EOF > /home/ubuntu/join-cluster-instructions.txt
========================================
Kubernetes Worker Node Setup Complete
Setup Time: $(date)
========================================

This worker node is now prepared to join a Kubernetes cluster.

NEXT STEPS:
-----------

1. SSH into your CONTROL PLANE node:
   $ ssh ubuntu@<control-plane-ip>

2. Get the join command:
   $ cat kubeadm-join-command.sh
   
   OR generate a new one:
   $ kubeadm token create --print-join-command

3. SSH into THIS worker node:
   $ ssh ubuntu@<this-worker-ip>

4. Run the join command with sudo:
   $ sudo kubeadm join <control-plane-ip>:6443 --token <token> \\
       --discovery-token-ca-cert-hash sha256:<hash>

5. Verify on control plane:
   $ kubectl get nodes

EXAMPLE JOIN COMMAND:
---------------------
sudo kubeadm join 172.31.10.10:6443 \\
  --token abcdef.0123456789abcdef \\
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...

TROUBLESHOOTING:
----------------
- Check logs: /var/log/k8s-worker-setup.log
- Verify containerd: sudo systemctl status containerd
- Verify kubelet: sudo systemctl status kubelet
- Check swap: free -h (should show 0 swap)
- Test connectivity to control plane: ping <control-plane-ip>

========================================
EOF

chown ubuntu:ubuntu /home/ubuntu/join-cluster-instructions.txt

#############################################################
# Setup Complete
#############################################################

echo ""
echo "=========================================="
echo "Kubernetes Worker Node Setup COMPLETE!"
echo "Time: $(date)"
echo "=========================================="
echo ""
echo "Instructions saved to: /home/ubuntu/join-cluster-instructions.txt"
echo ""
echo "Next: Get join command from control plane and run it here with sudo"
echo ""
echo "Full log available at: /var/log/k8s-worker-setup.log"
echo "=========================================="
```

---

## Deployment Instructions

### Step-by-Step EC2 Launch

#### Step 1: Launch Control Plane Instance

1. **Go to AWS EC2 Console**
2. **Click "Launch Instance"**
3. **Configure:**
   - **Name:** `k8s-control-plane`
   - **AMI:** Ubuntu Server 24.04 LTS
   - **Instance Type:** t3.medium (or t2.medium)
   - **Key Pair:** Select or create your SSH key
   - **Network:** 
     - VPC: Your VPC
     - Subnet: Public subnet
     - Auto-assign Public IP: **Enable**
   - **Security Group:** Select `k8s-cluster-sg` (created in prerequisites)
   - **Storage:** 20 GB gp3

4. **Expand "Advanced Details"**
5. **Scroll to "User Data"**
6. **Copy and paste the entire Control Plane User Data script**
7. **Click "Launch Instance"**

**⏱️ Wait Time:** 5-7 minutes for complete setup

---

#### Step 2: Launch Worker Node Instances

**Repeat for Worker Node 1:**

1. **Click "Launch Instance"**
2. **Configure:**
   - **Name:** `k8s-worker-1`
   - **AMI:** Ubuntu Server 24.04 LTS
   - **Instance Type:** t3.small (or t2.small)
   - **Key Pair:** Same as control plane
   - **Network:** Same VPC and subnet
   - **Security Group:** `k8s-cluster-sg`
   - **Storage:** 20 GB gp3

3. **In "Advanced Details" → "User Data":**
   - Copy and paste the Worker Node User Data script

4. **Click "Launch Instance"**

**Repeat for Worker Node 2:** (Same steps, name it `k8s-worker-2`)

**⏱️ Wait Time:** 3-4 minutes for each worker setup

---

#### Step 3: Verify Instances are Running

```bash
# In AWS Console, ensure all 3 instances show:
# - State: Running
# - Status Check: 2/2 checks passed (wait 2-3 minutes)
```

---

## Post-Deployment Steps

### Step 1: Verify Control Plane Setup

```bash
# SSH into Control Plane
ssh -i your-key.pem ubuntu@<control-plane-public-ip>

# Check setup log (IMPORTANT - verify success)
tail -50 /var/log/k8s-control-plane-setup.log

# Should show "Setup COMPLETE!" at the end

# Verify cluster
kubectl get nodes
# Should show control-plane in Ready state (wait 2-3 min if NotReady)

kubectl get pods -A
# All pods should be Running
```

---

### Step 2: Get Join Command

```bash
# On Control Plane node
cat ~/kubeadm-join-command.sh

# Output example:
# kubeadm join 172.31.10.10:6443 --token abc123.xyz789 \
#   --discovery-token-ca-cert-hash sha256:1234567890abcdef...

# Copy this entire command
```

**Alternative (if file missing):**
```bash
kubeadm token create --print-join-command
```

---

### Step 3: Join Worker Nodes

**For Each Worker Node:**

```bash
# SSH into Worker Node
ssh -i your-key.pem ubuntu@<worker-public-ip>

# Check setup log
tail -50 /var/log/k8s-worker-setup.log

# Should show "Setup COMPLETE!"

# Read instructions
cat ~/join-cluster-instructions.txt

# Run join command (paste from control plane)
sudo kubeadm join 172.31.10.10:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef...

# Should see:
# "This node has joined the cluster"
```

---

### Step 4: Verify Cluster from Control Plane

```bash
# On Control Plane
kubectl get nodes

# Expected output:
# NAME                              STATUS   ROLES           AGE
# k8s-control-plane                 Ready    control-plane   10m
# k8s-worker-i-0abcd1234           Ready    <none>          2m
# k8s-worker-i-0efgh5678           Ready    <none>          2m

# Check with more details
kubectl get nodes -o wide

# Verify all system pods
kubectl get pods -A
```

---

## Verification

### Test 1: Deploy Sample Application

```bash
# On Control Plane
kubectl create deployment nginx --image=nginx --replicas=3

# Wait 30 seconds
kubectl get pods -o wide
# Should see 3 pods distributed across worker nodes

kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
# Note the NodePort (e.g., 30XXX)

# Test from control plane
curl <worker-1-private-ip>:<NodePort>
# Should return nginx welcome page
```

---

### Test 2: Verify DNS

```bash
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

# Should resolve to 10.96.0.1
```

---

### Test 3: Verify Networking Between Pods

```bash
# Create 2 test pods
kubectl run pod1 --image=nginx
kubectl run pod2 --image=nginx

# Get pod IPs
kubectl get pods -o wide

# Test connectivity
kubectl exec pod1 -- curl <pod2-ip>
# Should succeed
```

---

## Troubleshooting

### Issue 1: Control Plane Setup Failed

**Symptoms:**
```bash
tail /var/log/k8s-control-plane-setup.log
# Shows errors
```

**Diagnosis:**
```bash
# Check specific logs
cat /var/log/kubeadm-init.log

# Check containerd
sudo systemctl status containerd

# Check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

**Common Causes:**

**A) Network/Internet Issues:**
```bash
# Test internet connectivity
ping -c 3 8.8.8.8
curl -I https://google.com

# Check DNS
cat /etc/resolv.conf
```

**Fix:** Ensure EC2 instance has internet access (public IP or NAT gateway).

**B) Port Conflicts:**
```bash
sudo netstat -tulpn | grep -E '6443|2379|2380|10250'
```

**Fix:** Ensure no other services using these ports.

**C) Insufficient Resources:**
```bash
free -h
df -h
```

**Fix:** Use t3.medium minimum for control plane.

---

### Issue 2: Worker Node Join Failed

**Symptoms:**
```bash
sudo kubeadm join ...
# Error: [ERROR FileAvailable--etc-kubernetes-kubelet.conf]
```

**Fix - Reset and Retry:**
```bash
# On worker node
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo systemctl restart kubelet

# Get fresh join command from control plane
# Run join command again
```

---

### Issue 3: Calico Pods Not Running

**Symptoms:**
```bash
kubectl get pods -n calico-system
# Shows CrashLoopBackOff or Pending
```

**Diagnosis:**
```bash
kubectl describe pod -n calico-system <calico-pod-name>
kubectl logs -n calico-system <calico-pod-name>
```

**Common Fixes:**

**A) Network Policy Issue:**
```bash
# On control plane, verify IP forwarding
sysctl net.ipv4.ip_forward
# Should be 1

# Restart Calico
kubectl delete pod -n calico-system --all
# Pods will be recreated
```

**B) Security Group Issue:**
```bash
# Ensure security group allows:
# - Port 179 (BGP)
# - Port 4789 (VXLAN)
# - All traffic between cluster nodes
```

---

### Issue 4: Nodes Show NotReady

**Symptoms:**
```bash
kubectl get nodes
# Shows NotReady
```

**Diagnosis:**
```bash
kubectl describe node <node-name>
# Look at Conditions section

# Check CNI
kubectl get pods -n calico-system
```

**Fix:**
```bash
# Usually CNI issue
# Wait 2-3 minutes for Calico to fully initialize

# If still NotReady, restart Calico
kubectl delete pod -n calico-system --all

# On problematic node, restart services
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

---

### Issue 5: Cannot Access Cluster (kubectl)

**Symptoms:**
```bash
kubectl get nodes
# Error: The connection to the server localhost:8080 was refused
```

**Fix:**
```bash
# Verify kubeconfig exists
ls -la ~/.kube/config

# If missing, copy from admin.conf
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Test again
kubectl get nodes
```

---

### Issue 6: Token Expired

**Symptoms:**
```bash
sudo kubeadm join ...
# Error: token is not valid
```

**Fix:**
```bash
# On control plane, generate new token
kubeadm token create --print-join-command

# Use new command on worker
```

---

## Advanced Configurations

### Configuration 1: Add More Worker Nodes Later

```bash
# On Control Plane:
kubeadm token create --print-join-command

# Launch new EC2 with Worker User Data script
# After setup completes, SSH and run join command
```

---

### Configuration 2: Label Worker Nodes

```bash
# On Control Plane
kubectl label node k8s-worker-i-0abcd1234 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-i-0efgh5678 node-role.kubernetes.io/worker=worker

# Verify
kubectl get nodes
# Should show role "worker" instead of "<none>"
```

---

### Configuration 3: Custom Hostnames in User Data

**Modify Worker User Data:**

Replace the hostname section:
```bash
# OLD:
INSTANCE_ID=$(ec2-metadata --instance-id | cut -d " " -f 2)
hostnamectl set-hostname k8s-worker-${INSTANCE_ID}

# NEW (for worker 1):
hostnamectl set-hostname k8s-worker-1
echo "127.0.1.1 k8s-worker-1" >> /etc/hosts

# NEW (for worker 2):
hostnamectl set-hostname k8s-worker-2
echo "127.0.1.1 k8s-worker-2" >> /etc/hosts
```

---

### Configuration 4: Install kubectl Autocomplete

Already included in scripts! Test:
```bash
# Start new shell or:
source ~/.bashrc

# Try autocomplete:
kubectl get n<TAB>
# Should complete to "nodes"

# Use alias:
k get nodes
```

---

### Configuration 5: Access Cluster from Local Machine

```bash
# On Control Plane:
cat ~/.kube/config

# Copy entire content

# On your local machine:
mkdir -p ~/.kube
nano ~/.kube/config
# Paste content

# Modify server IP to control plane PUBLIC IP:
# Change:
#   server: https://172.31.10.10:6443
# To:
#   server: https://<control-plane-public-ip>:6443

# Test:
kubectl get nodes
```

**Note:** Ensure security group allows your IP to access port 6443.

---

### Configuration 6: Enable Metrics Server (Optional)

```bash
# On Control Plane
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait 1 minute, then test:
kubectl top nodes
kubectl top pods
```

---

## Best Practices

### 1. Tag Your Resources

In AWS Console, add tags to all instances:
```
Key: Environment  |  Value: kubernetes-lab
Key: Cluster      |  Value: k8s-cluster-01
Key: ManagedBy    |  Value: userdata-automation
```

---

### 2. Use Elastic IPs for Production

- Attach Elastic IP to Control Plane
- Update kubeconfig with stable IP
- Workers can use dynamic IPs

---

### 3. Create AMI After Setup

After confirming cluster works:
```bash
# Stop an instance
# Create AMI from it
# Use AMI for faster future deployments
```

---

### 4. Backup etcd Regularly

```bash
# On Control Plane
sudo ETCDCTL_API=3 etcdctl snapshot save /home/ubuntu/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Upload to S3
aws s3 cp /home/ubuntu/etcd-backup.db s3://your-bucket/k8s-backups/
```

---

### 5. Use Launch Templates

Instead of copying User Data each time:
1. Create Launch Template with User Data
2. Launch instances from template
3. Faster and less error-prone

---

### 6. Monitor Setup Logs

```bash
# Watch User Data execution in real-time
ssh ubuntu@<instance-ip>
tail -f /var/log/k8s-control-plane-setup.log
# or
tail -f /var/log/k8s-worker-setup.log
```

---

### 7. Cleanup After Testing

```bash
# Terminate all EC2 instances
# Release Elastic IPs if used
# Delete security groups
# Remove key pairs if not needed
```

**Cost Savings:** Stop instances when not in use (not recommended for production).

---

## Quick Reference

### Control Plane Commands

```bash
# Get join command
kubeadm token create --print-join-command

# View cluster
kubectl get nodes -o wide
kubectl get pods -A

# Check logs
tail -50 /var/log/k8s-control-plane-setup.log

# Restart component
sudo systemctl restart kubelet
```

---

### Worker Node Commands

```bash
# Check setup
tail -50 /var/log/k8s-worker-setup.log

# Join cluster (example)
sudo kubeadm join 172.31.10.10:6443 --token abc.xyz \
  --discovery-token-ca-cert-hash sha256:123...

# Reset if needed
sudo kubeadm reset -f
```

---

### Verification Commands

```bash
# Cluster health
kubectl get componentstatuses
kubectl get nodes
kubectl get pods -A

# Deploy test app
kubectl create deployment test --image=nginx
kubectl get pods -o wide

# Check networking
kubectl run test --image=busybox --rm -it -- sh
```

---

## Summary

**What You Get:**
- ✅ Fully automated Kubernetes cluster setup
- ✅ No manual commands required during instance launch
- ✅ Production-ready configuration
- ✅ Proper security settings (swap off, sysctl, modules)
- ✅ containerd as runtime
- ✅ Calico CNI for networking
- ✅ kubectl configured with autocomplete
- ✅ Complete logging for debugging

**Total Setup Time:**
- Control Plane: 5-7 minutes
- Worker Nodes: 3-4 minutes each
- Join Process: 1 minute per worker
- **Total: ~15 minutes for full 3-node cluster**

**Cost Estimate (as of 2026):**
- t3.medium (Control): ~$0.0416/hour
- t3.small x2 (Workers): ~$0.0208/hour each
- **Total: ~$0.0832/hour or $2/day if running 24/7**

**Stop instances when not in use to save costs!**

---

## Next Steps for Students

1. **Practice deploying applications**
2. **Learn Services (ClusterIP, NodePort, LoadBalancer)**
3. **Explore ConfigMaps and Secrets**
4. **Study Ingress controllers (nginx-ingress)**
5. **Implement Persistent Volumes**
6. **Practice RBAC (Role-Based Access Control)**
7. **Learn Helm for package management**
8. **Implement monitoring (Prometheus/Grafana)**
9. **Practice cluster upgrades**
10. **Study disaster recovery (etcd backup/restore)**

---

## Additional Resources

- **AWS EC2 User Data Docs:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html
- **Kubernetes Documentation:** https://kubernetes.io/docs/
- **kubeadm Setup Guide:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- **Calico Documentation:** https://docs.tigera.io/calico/

---

**Happy Learning! 🚀**

*Remember: Infrastructure as Code (IaC) is the modern way. These User Data scripts are your first step into automated infrastructure deployment!*

---

## 🧑‍💻 Author

*Md. Sarowar Alam*  
Lead DevOps Engineer, Hogarth Worldwide  
📧 Email: sarowar@hotmail.com  
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/

---
