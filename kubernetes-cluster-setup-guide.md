# Kubernetes Cluster Setup Guide with kubeadm
## Production-Ready Setup for Ubuntu 22.04 on AWS EC2

---

## 📋 Table of Contents

1. [Cluster Overview](#cluster-overview)
2. [Prerequisites](#prerequisites)
3. [Step-by-Step Setup](#step-by-step-setup)
4. [Kubernetes Architecture Deep Dive](#kubernetes-architecture-deep-dive)
5. [Verification & Testing](#verification--testing)
6. [Troubleshooting](#troubleshooting)
7. [Common Mistakes & Best Practices](#common-mistakes--best-practices)

---

## Cluster Overview

**Setup Details:**
- **Platform:** AWS EC2
- **OS:** Ubuntu 22.04 LTS
- **Nodes:** 3 Total (1 Control Plane + 2 Workers)
- **Cluster Tool:** kubeadm
- **Container Runtime:** containerd
- **CNI Plugin:** Calico
- **Kubernetes Version:** 1.28+ (latest stable)

**Infrastructure:**

```
┌─────────────────────────────────────────────────────────┐
│                    AWS VPC / Cloud                      │
│                                                         │
│  ┌──────────────────┐       ┌────────────────────┐    │
│  │  Control Plane   │       │   Worker Node 1    │    │
│  │   (Master Node)  │◄─────►│                    │    │
│  │  - API Server    │       │  - kubelet         │    │
│  │  - Scheduler     │       │  - kube-proxy      │    │
│  │  - Controller Mgr│       │  - containerd      │    │
│  │  - etcd          │       │  - Pods            │    │
│  └──────────────────┘       └────────────────────┘    │
│           ▲                                            │
│           │                                            │
│           │                 ┌────────────────────┐    │
│           └────────────────►│   Worker Node 2    │    │
│                             │                    │    │
│                             │  - kubelet         │    │
│                             │  - kube-proxy      │    │
│                             │  - containerd      │    │
│                             │  - Pods            │    │
│                             └────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### AWS EC2 Instance Requirements

**Minimum Specifications:**
- **Control Plane:** t2.medium (2 vCPU, 4GB RAM)
- **Worker Nodes:** t2.small or t2.medium (1-2 vCPU, 2-4GB RAM)
- **Storage:** 20GB+ root volume for each node
- **Security Group:** Allow ports (see below)

**Required Ports:**

**Control Plane Node:**
```
6443      - Kubernetes API Server
2379-2380 - etcd server client API
10250     - Kubelet API
10259     - kube-scheduler
10257     - kube-controller-manager
```

**Worker Nodes:**
```
10250     - Kubelet API
30000-32767 - NodePort Services (optional, for external access)
```

**All Nodes (for Calico CNI):**
```
179       - BGP (Calico networking)
4789      - VXLAN (Calico overlay)
```

**SSH Access:**
```
22        - SSH access from your IP
```

---

## Step-by-Step Setup

### Phase 1: Prepare ALL Nodes (Control Plane + Workers)

> **WHY?** Kubernetes requires a consistent environment across all nodes. This includes system updates, kernel modules, network settings, and swap disabled.

#### Step 1.1: Update System Packages

```bash
# Run on ALL nodes
sudo apt-get update
sudo apt-get upgrade -y
```

**Verification:**
```bash
cat /etc/os-release
# Should show Ubuntu 22.04
```

**WHY:** Ensures all security patches and system libraries are up to date.

---

#### Step 1.2: Set Hostnames (Important for Node Identification)

```bash
# On Control Plane node:
sudo hostnamectl set-hostname k8s-control-plane

# On Worker Node 1:
sudo hostnamectl set-hostname k8s-worker-1

# On Worker Node 2:
sudo hostnamectl set-hostname k8s-worker-2
```

**Add entries to /etc/hosts on ALL nodes:**
```bash
sudo nano /etc/hosts
```

Add these lines (replace with your actual private IPs):
```
172.31.10.10   k8s-control-plane
172.31.10.11   k8s-worker-1
172.31.10.12   k8s-worker-2
```

**Verification:**
```bash
hostname
# Should show the hostname you set
ping k8s-control-plane -c 2
```

**WHY:** Kubernetes identifies nodes by hostname. Proper DNS/host resolution prevents communication issues.

**Common Mistake:** Forgetting to add IP mappings causes node join failures.

---

#### Step 1.3: Disable Swap (CRITICAL)

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Verification:**
```bash
free -h
# The 'Swap' line should show 0B total

cat /etc/fstab | grep swap
# Should be commented out with #
```

**WHY:** Kubernetes scheduler assumes memory management by the kernel. Swap can cause unpredictable pod behavior and performance issues. kubelet will refuse to start if swap is enabled.

**Real-World Analogy:** Think of Kubernetes as a strict traffic controller. It needs to know exactly how many parking spots (memory) are available. Swap is like "temporary parking" on the street - it confuses the controller and creates chaos.

**Common Mistake:** Swap re-enables on reboot if /etc/fstab isn't updated.

---

#### Step 1.4: Load Required Kernel Modules

```bash
# Create configuration file
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter
```

**Verification:**
```bash
lsmod | grep overlay
lsmod | grep br_netfilter
# Both should show output
```

**WHY:**
- **overlay:** Required for containerd's overlay filesystem (efficient container layer storage)
- **br_netfilter:** Enables iptables to see bridged network traffic (critical for pod networking)

**Real-World Analogy:** These modules are like security cameras at a bridge. Without them, Kubernetes can't "see" traffic between containers and apply network policies.

---

#### Step 1.5: Configure sysctl Network Settings

```bash
# Create sysctl configuration
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings immediately
sudo sysctl --system
```

**Verification:**
```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
# All three should return = 1
```

**WHY:**
- **bridge-nf-call-iptables:** Allows iptables to process packets traversing a bridge (needed for Kubernetes network policies and service routing)
- **ip_forward:** Enables the Linux kernel to forward packets between network interfaces (essential for pod-to-pod communication across nodes)

**Common Mistake:** Skipping sysctl apply causes CNI plugins to fail silently.

---

### Phase 2: Install containerd (ALL Nodes)

> **WHY containerd?** It's the industry-standard container runtime, lightweight, and officially supported by Kubernetes. Docker is deprecated in Kubernetes.

#### Step 2.1: Install containerd Prerequisites

```bash
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

**WHY:** These packages enable secure package downloads over HTTPS and repository management.

---

#### Step 2.2: Install containerd

```bash
# Add Docker's official GPG key (containerd is maintained by Docker)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt-get update

# Install containerd
sudo apt-get install -y containerd.io
```

**Verification:**
```bash
systemctl status containerd
# Should show 'active (running)'

containerd --version
# Should show version 1.6+
```

---

#### Step 2.3: Configure containerd for Kubernetes

```bash
# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (CRITICAL for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Verification:**
```bash
sudo systemctl status containerd
# Should still be active after restart

# Verify SystemdCgroup is enabled
grep SystemdCgroup /etc/containerd/config.toml
# Should show: SystemdCgroup = true
```

**WHY SystemdCgroup?** Kubernetes and containerd must use the same cgroup driver (systemd) to manage resource limits. Mismatch causes kubelet to fail.

**Real-World Analogy:** Cgroup driver is like currency. If Kubernetes uses dollars and containerd uses euros without conversion, transactions fail.

**Common Mistake:** Forgetting to enable SystemdCgroup causes cryptic kubelet errors like "failed to create pod sandbox."

---

### Phase 3: Install kubeadm, kubelet, kubectl (ALL Nodes)

> **WHY these three tools?**
> - **kubeadm:** Bootstraps the cluster (like an installer)
> - **kubelet:** Node agent that runs on every node (like a worker that executes tasks)
> - **kubectl:** Command-line tool to interact with the cluster (like a remote control)

#### Step 3.1: Add Kubernetes Repository

```bash
# Add Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index
sudo apt-get update
```

---

#### Step 3.2: Install Kubernetes Tools

```bash
# Install kubeadm, kubelet, kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent automatic updates (version lock)
sudo apt-mark hold kubelet kubeadm kubectl
```

**Verification:**
```bash
kubeadm version
kubectl version --client
kubelet --version
# All should show version 1.28.x
```

**WHY hold packages?** Prevents accidental upgrades that can break cluster compatibility. Kubernetes upgrades must be done deliberately with proper planning.

---

#### Step 3.3: Enable kubelet

```bash
sudo systemctl enable kubelet
# Note: kubelet will be in a crash loop until cluster is initialized - this is NORMAL
```

**Verification:**
```bash
systemctl status kubelet
# Will show "activating" or "failed" - this is expected before cluster init
```

**WHY is kubelet failing?** It's waiting for kubeadm to provide cluster configuration. This is normal and expected.

---

### Phase 4: Initialize Control Plane (Master Node ONLY)

> **This is the most critical step.** This creates the Kubernetes control plane - the brain of your cluster.

#### Step 4.1: Initialize the Cluster

```bash
# Run ONLY on Control Plane node
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=$(hostname -i) \
  --control-plane-endpoint=$(hostname -i)
```

**Parameter Explanation:**
- `--pod-network-cidr=192.168.0.0/16`: IP range for pod network (required by Calico)
- `--apiserver-advertise-address`: IP address the API server advertises to cluster members
- `--control-plane-endpoint`: IP for accessing the control plane (use LB IP in HA setups)

**What happens during init:**
1. Preflight checks (swap, ports, etc.)
2. Downloads container images for control plane components
3. Generates certificates for secure communication
4. Starts etcd (database)
5. Starts API server, scheduler, controller manager
6. Generates bootstrap tokens for worker nodes

**Expected Output:**
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

...

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.10.10:6443 --token abc123.xyz... \
    --discovery-token-ca-cert-hash sha256:abcdef...
```

**⚠️ IMPORTANT:** Save the `kubeadm join` command - you'll need it for worker nodes!

**Troubleshooting:**
- **Error: port 6443 in use:** Another process is using the port. Check with `sudo netstat -tulpn | grep 6443`
- **Error: swap is on:** Run `sudo swapoff -a` again
- **Error: container runtime not running:** Check containerd: `sudo systemctl status containerd`

---

#### Step 4.2: Configure kubectl for Regular User

```bash
# Run as regular user (ubuntu)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**WHY:** admin.conf contains credentials to authenticate with the API server. Moving it to ~/.kube/config allows kubectl to work without sudo.

**Verification:**
```bash
kubectl get nodes
# Should show control-plane node in 'NotReady' state (CNI not installed yet)

kubectl get pods -n kube-system
# Should show coredns pods in 'Pending' state (normal without CNI)
```

---

### Phase 5: Install CNI Plugin - Calico (Control Plane Node)

> **WHY CNI?** Container Network Interface (CNI) plugins enable pod-to-pod networking. Without CNI, pods can't communicate, and nodes remain NotReady.

**Real-World Analogy:** CNI is like the postal service. Control plane (government) is running, but without a postal service, citizens (pods) can't send mail (packets) to each other.

#### Step 5.1: Install Calico

```bash
# Install Calico operator and custom resources
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Download custom resources manifest
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml -O

# Apply Calico configuration
kubectl create -f custom-resources.yaml
```

**What Calico does:**
- Creates virtual network between all pods
- Implements NetworkPolicy for security
- Handles IP address management (IPAM)
- Routes traffic between nodes using BGP

**Verification (wait 2-3 minutes):**
```bash
# Watch Calico pods start
kubectl get pods -n calico-system --watch
# Press Ctrl+C when all pods are Running

# Check node status
kubectl get nodes
# Should now show 'Ready'

# Verify Calico installation
kubectl get pods -n calico-system
# All pods should be 'Running'
```

**Common Issues:**
- **Pods stuck in Init:** Check network connectivity between nodes
- **ImagePullBackOff:** Nodes can't reach internet/registry. Check security groups and NAT gateway
- **CrashLoopBackOff:** Check pod logs: `kubectl logs -n calico-system <pod-name>`

---

### Phase 6: Join Worker Nodes (Worker Nodes ONLY)

> **This connects your worker nodes to the control plane.**

#### Step 6.1: Run Join Command

```bash
# On Worker Node 1 and Worker Node 2, run the join command from Step 4.1 output
# Example (use YOUR token and hash):
sudo kubeadm join 172.31.10.10:6443 \
    --token abc123.xyz789abc123 \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**Expected Output:**
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

---

#### Step 6.2: Verify Worker Nodes Joined

```bash
# On Control Plane node
kubectl get nodes
# Should show all 3 nodes in 'Ready' state

kubectl get nodes -o wide
# Shows more details including internal IPs and OS versions
```

**Expected Output:**
```
NAME                 STATUS   ROLES           AGE   VERSION
k8s-control-plane    Ready    control-plane   10m   v1.28.x
k8s-worker-1         Ready    <none>          2m    v1.28.x
k8s-worker-2         Ready    <none>          2m    v1.28.x
```

---

#### Step 6.3: Lost Join Token? Generate a New One

```bash
# On Control Plane node
kubeadm token create --print-join-command
# Outputs complete join command with new token
```

**WHY:** Tokens expire after 24 hours for security. This regenerates a valid join command.

---

### Phase 7: Cluster Verification

#### Test 7.1: Deploy Test Application

```bash
# Create nginx deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Verify pods are distributed across workers
kubectl get pods -o wide
# Should see pods on different nodes

# Expose as service
kubectl expose deployment nginx --port=80 --type=NodePort

# Get service details
kubectl get svc nginx
# Note the NodePort (30000-32767)
```

**Access Test:**
```bash
# From any node, test service (replace 30XXX with actual NodePort)
curl localhost:30XXX
# Should return nginx welcome page
```

---

#### Test 7.2: Verify DNS

```bash
# Create test pod
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
# Should resolve to cluster IP (10.96.0.1)
```

---

#### Test 7.3: Check Cluster Health

```bash
# Component status
kubectl get componentstatuses

# All system pods
kubectl get pods -n kube-system
# All should be Running

# Node resources
kubectl top nodes
# Requires metrics-server (optional)
```

---

## Kubernetes Architecture Deep Dive

### Overview: The Orchestra Analogy

Think of Kubernetes as a symphony orchestra:
- **Control Plane** = Conductor + Music Library
- **Worker Nodes** = Musicians
- **Pods** = Musical pieces being performed
- **etcd** = Sheet music archive
- **Scheduler** = Seating arranger (assigns musicians to seats)

---

### Control Plane Components (Master Node)

```
┌──────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE NODE                        │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              kube-apiserver                         │    │
│  │  • REST API frontend                                │    │
│  │  • Authentication & Authorization                   │    │
│  │  • All communication goes through here              │    │
│  │  • Port: 6443                                       │    │
│  └───────────┬──────────────────────────────┬─────────┘    │
│              │                              │               │
│    ┌─────────▼─────────┐         ┌─────────▼──────────┐   │
│    │                   │         │                     │   │
│    │  kube-scheduler   │         │ kube-controller-    │   │
│    │                   │         │     manager         │   │
│    │  • Assigns pods   │         │                     │   │
│    │    to nodes       │         │ • Node Controller   │   │
│    │  • Watches for    │         │ • Replication Ctrl  │   │
│    │    unscheduled    │         │ • Endpoints Ctrl    │   │
│    │    pods           │         │ • Service Account   │   │
│    │                   │         │   & Token Ctrl      │   │
│    └───────────────────┘         └─────────────────────┘   │
│                                                              │
│              ┌───────────────────────────┐                  │
│              │         etcd              │                  │
│              │  • Key-value store        │                  │
│              │  • Cluster state database │                  │
│              │  • Source of truth        │                  │
│              │  • Ports: 2379-2380       │                  │
│              └───────────────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

---

#### 1. kube-apiserver

**Role:** The front door of Kubernetes. Every request (kubectl commands, other components) goes through the API server.

**What it does:**
- Validates and processes REST requests
- Authenticates users and service accounts
- Authorizes operations (RBAC)
- Reads/writes cluster state to etcd
- Serves as the only component that talks to etcd directly

**Real-World Analogy:** The receptionist at a hospital. Everyone (doctors, nurses, patients) must go through reception. Reception checks your ID, verifies appointments, and updates medical records.

**Port:** 6443 (HTTPS)

**Example Flow:**
```
kubectl get pods → API Server → Authenticates → Authorizes → Queries etcd → Returns data
```

---

#### 2. etcd

**Role:** The cluster's database. Stores all cluster configuration, state, and metadata.

**What it does:**
- Stores current state: which pods run on which nodes
- Stores desired state: deployment specifications
- Distributed, consistent, high-availability key-value store
- Uses Raft consensus algorithm for consistency

**Real-World Analogy:** Library archive. Contains all books (configurations) and checkout records (current state). If someone asks "Which book is checked out?", you consult the archive.

**Why it's critical:** If etcd fails, you lose cluster state. Always back up etcd in production.

**Data stored in etcd:**
- Pod definitions
- Service definitions
- ConfigMaps, Secrets
- Node registration
- And more...

**Port:** 2379 (client), 2380 (peer communication)

---

#### 3. kube-scheduler

**Role:** Decides which node should run each newly created pod.

**What it does:**
- Watches API server for unscheduled pods
- Evaluates all nodes based on:
  - Resource availability (CPU, memory)
  - Node selectors and affinity rules
  - Taints and tolerations
  - Resource requests and limits
- Assigns pod to best-fit node
- Updates API server with decision

**Real-World Analogy:** Hotel front desk assigning rooms. When guest (pod) arrives, front desk checks available rooms (nodes), considers preferences (affinity), and assigns best room based on availability and requirements.

**Does NOT run pods:** Scheduler only makes decisions. Kubelet on the worker node actually runs the pod.

**Scheduling Flow:**
```
1. User creates pod → API server
2. Scheduler watches, sees unscheduled pod
3. Scheduler evaluates nodes (filtering + scoring)
4. Scheduler writes decision to API server
5. Kubelet on chosen node pulls pod and runs it
```

---

#### 4. kube-controller-manager

**Role:** Runs controller processes that regulate cluster state.

**What it does:**
- Continuously monitors cluster state via API server
- Detects differences between current state and desired state
- Takes action to reconcile (fix) differences
- Houses multiple controllers in one binary

**Key Controllers:**
- **Node Controller:** Monitors node health, marks nodes as unhealthy
- **ReplicaSet Controller:** Ensures desired number of pod replicas are running
- **Endpoints Controller:** Populates Endpoints objects (maps Services to Pods)
- **ServiceAccount & Token Controllers:** Creates default accounts and API access tokens for new namespaces

**Real-World Analogy:** Building maintenance crew. They continuously patrol the building (cluster). If they see broken lights (failed pods), they replace them. If temperature is off (not enough replicas), they adjust the thermostat.

**Reconciliation Loop Example:**
```
Desired State: 3 nginx pods
Current State: 2 nginx pods (1 crashed)

Controller sees difference → Creates 1 new pod → Reconciled
```

---

### Worker Node Components

```
┌──────────────────────────────────────────────────────────────┐
│                      WORKER NODE                             │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              kubelet                                │    │
│  │  • Node agent                                       │    │
│  │  • Talks to API server                              │    │
│  │  • Manages pods on this node                        │    │
│  │  • Reports node status                              │    │
│  │  • Port: 10250                                      │    │
│  └───────┬────────────────────────────────────────────┘    │
│          │                                                   │
│          │    ┌──────────────────────────────┐             │
│          │    │   Container Runtime          │             │
│          └───►│   (containerd)               │             │
│               │                              │             │
│               │  • Pulls images              │             │
│               │  • Starts/stops containers   │             │
│               │  • Manages container lifecycle│            │
│               └──────────────────────────────┘             │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              kube-proxy                             │    │
│  │  • Network proxy                                    │    │
│  │  • Manages iptables/IPVS rules                      │    │
│  │  • Enables Service networking                       │    │
│  │  • Routes traffic to correct pods                   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    PODS                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │Container │  │Container │  │Container │          │   │
│  │  │   App1   │  │   App2   │  │   App3   │   ...    │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

#### 1. kubelet

**Role:** The primary node agent. Ensures containers are running on the node.

**What it does:**
- Registers node with API server
- Watches API server for pod assignments to this node
- Instructs container runtime to pull images and run containers
- Monitors pod and container health
- Reports pod status back to API server
- Executes container lifecycle hooks

**Real-World Analogy:** Factory floor supervisor. Receives work orders from headquarters (control plane), assigns tasks to workers (containers), monitors progress, and reports back.

**Port:** 10250

**Important:** Kubelet does NOT manage containers that weren't created by Kubernetes.

**Kubelet Flow:**
```
1. API server assigns pod to node
2. Kubelet on node sees assignment
3. Kubelet tells containerd: "Pull nginx:latest"
4. Kubelet tells containerd: "Start container"
5. Kubelet monitors container
6. Kubelet reports status to API server
```

---

#### 2. kube-proxy

**Role:** Network proxy that maintains network rules on nodes.

**What it does:**
- Implements Kubernetes Service concept
- Maintains iptables/IPVS rules for forwarding traffic to pods
- Enables load balancing for Services
- Allows pods to communicate with Services via cluster IP

**Real-World Analogy:** Post office sorter. When mail (network packet) arrives addressed to "Customer Service" (Service name), the sorter looks up which specific person (pod) should receive it and redirects accordingly.

**How it works:**
```
Request to Service IP (10.96.0.50:80)
    ↓
kube-proxy iptables rules
    ↓
Randomly selects one backend pod (192.168.1.10:8080)
    ↓
Forwards traffic to pod
```

**Modes:**
- **iptables** (default): Uses iptables rules for routing
- **IPVS**: More efficient for large clusters

---

#### 3. Container Runtime (containerd)

**Role:** Software responsible for running containers.

**What it does:**
- Pulls container images from registries
- Extracts and manages images
- Creates and starts containers
- Stops and removes containers
- Manages container resources (CPU, memory)

**Why containerd?** Lightweight, secure, industry-standard. Docker support was removed in Kubernetes 1.24.

**Real-World Analogy:** The stagehand in a theater. Doesn't write the play (image), but sets up the stage (container) for actors to perform.

**Interaction with kubelet:**
```
kubelet: "Start nginx container"
containerd: "Pulling nginx:latest..."
containerd: "Extracting layers..."
containerd: "Container running with PID 12345"
kubelet: "Container started successfully"
```

---

### Complete Request Flow Example

**Scenario:** User runs `kubectl run nginx --image=nginx`

```
1. kubectl → API Server (authenticates, validates request)
2. API Server → etcd (writes pod definition)
3. Scheduler watches API server → sees new unscheduled pod
4. Scheduler evaluates nodes → selects k8s-worker-1
5. Scheduler → API Server (updates pod with node assignment)
6. API Server → etcd (saves assignment)
7. kubelet on k8s-worker-1 watches API server → sees pod assigned
8. kubelet → containerd ("pull and run nginx")
9. containerd → Docker Hub (pulls nginx image)
10. containerd creates and starts container
11. kubelet monitors container → reports status to API Server
12. API Server → etcd (saves pod status: Running)
13. kubectl get pods → API Server → Shows "nginx Running"
```

---

### Communication Patterns

```
┌──────────────────────────────────────────────────────────┐
│             CLUSTER COMMUNICATION FLOW                   │
│                                                          │
│  kubectl                                                 │
│     │                                                    │
│     │ HTTPS (6443)                                       │
│     ▼                                                    │
│  API Server ◄────────── kubelet (10250)                 │
│     │                                                    │
│     │                                                    │
│     ▼                                                    │
│   etcd (2379)                                            │
│                                                          │
│  API Server ◄────────── kube-proxy                      │
│     │                                                    │
│     ▼                                                    │
│  Scheduler                                               │
│                                                          │
│  API Server ◄────────── Controller Manager              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Key Principle:** ALL communication flows through API Server. No component talks directly to etcd except API Server.

---

## Verification & Testing

### Health Checks

```bash
# Check all nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check Calico
kubectl get pods -n calico-system

# Detailed node info
kubectl describe node k8s-worker-1

# Cluster info
kubectl cluster-info

# Component health
kubectl get --raw='/readyz?verbose'
```

---

### Deploy Multi-Container Test Application

```bash
# Create namespace
kubectl create namespace demo

# Deploy application with resource requests
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: demo
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
EOF
```

**Verify:**
```bash
kubectl get all -n demo
kubectl get pods -n demo -o wide
# Pods should be distributed across worker nodes
```

---

### Test Pod-to-Pod Communication

```bash
# Get pod IPs
kubectl get pods -n demo -o wide

# Test from one pod to another (replace pod names)
kubectl exec -n demo <pod-1> -- curl <pod-2-IP>
```

---

### Test Service DNS Resolution

```bash
# Create test pod
kubectl run test --image=busybox -it --rm --restart=Never -n demo -- sh

# Inside pod:
nslookup web-service
# Should resolve to Service ClusterIP

wget -O- web-service
# Should return nginx homepage

exit
```

---

## Troubleshooting

### Issue 1: Nodes Showing "NotReady"

**Symptoms:**
```bash
kubectl get nodes
# Shows NotReady
```

**Diagnosis:**
```bash
# Check node details
kubectl describe node <node-name>

# Check kubelet logs
sudo journalctl -u kubelet -f

# Check container runtime
sudo systemctl status containerd
```

**Common Causes & Fixes:**

**A) CNI Not Installed or Failing**
```bash
kubectl get pods -n calico-system
# If pods not running or missing, reinstall Calico

kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

**B) Kubelet Not Running**
```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

**C) Container Runtime Issues**
```bash
# Restart containerd
sudo systemctl restart containerd

# Check runtime endpoint
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
```

**D) Swap Enabled**
```bash
free -h
# If swap is active:
sudo swapoff -a
sudo systemctl restart kubelet
```

---

### Issue 2: Pods Stuck in "Pending"

**Symptoms:**
```bash
kubectl get pods
# Shows Pending
```

**Diagnosis:**
```bash
kubectl describe pod <pod-name>
# Look at Events section
```

**Common Causes & Fixes:**

**A) No nodes available / Resource constraints**
```bash
kubectl get nodes
# Ensure nodes are Ready

kubectl describe node <node-name>
# Check Allocatable resources
```

**Fix:** Add more worker nodes or reduce resource requests.

**B) Taints preventing scheduling**
```bash
kubectl describe node <node-name> | grep Taints

# Remove taint (if appropriate)
kubectl taint nodes <node-name> node.kubernetes.io/not-ready:NoSchedule-
```

**C) PersistentVolumeClaim not bound**
```bash
kubectl get pvc
# Check status

kubectl describe pvc <pvc-name>
```

---

### Issue 3: Pods Stuck in "ContainerCreating"

**Symptoms:**
```bash
kubectl get pods
# Shows ContainerCreating for extended period
```

**Diagnosis:**
```bash
kubectl describe pod <pod-name>
# Check Events

# On the worker node running the pod:
sudo journalctl -u kubelet | tail -50
```

**Common Causes & Fixes:**

**A) CNI Network Issues**
```bash
# Check Calico
kubectl get pods -n calico-system

# Restart Calico node pod on problematic worker
kubectl delete pod -n calico-system calico-node-<xxxx>
```

**B) Image Pull Issues**
```bash
kubectl describe pod <pod-name>
# Look for ImagePullBackOff or ErrImagePull

# Test image pull on node:
sudo crictl pull <image-name>
```

**Fix:** Ensure worker nodes have internet access or private registry credentials.

**C) Volume Mount Issues**
```bash
kubectl describe pod <pod-name>
# Look for mount-related errors

# Check if ConfigMap/Secret exists
kubectl get configmaps,secrets
```

---

### Issue 4: "Unable to connect to server"

**Symptoms:**
```bash
kubectl get nodes
# Error: Unable to connect to the server: dial tcp: lookup...
```

**Diagnosis:**
```bash
# Check API server
sudo systemctl status kubelet
kubectl get pods -n kube-system | grep apiserver

# Check ports
sudo netstat -tulpn | grep 6443
```

**Common Causes & Fixes:**

**A) API Server Not Running**
```bash
# Check control plane pods
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

# Check manifests
ls -la /etc/kubernetes/manifests/

# Restart kubelet (it will restart static pods)
sudo systemctl restart kubelet
```

**B) Kubeconfig Issues**
```bash
# Verify kubeconfig exists
cat ~/.kube/config

# Use correct context
kubectl config get-contexts
kubectl config use-context kubernetes-admin@kubernetes
```

**C) Network/Firewall Issues**
```bash
# Test API server connectivity
curl -k https://localhost:6443

# Check security group allows port 6443
```

---

### Issue 5: etcd Connection Problems

**Symptoms:**
```bash
kubectl get nodes
# Error: etcdserver: request timed out
```

**Diagnosis:**
```bash
# Check etcd pod
kubectl get pods -n kube-system | grep etcd

# Check etcd logs
kubectl logs -n kube-system etcd-<control-plane-name>

# On control plane node:
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock logs <etcd-container-id>
```

**Common Causes & Fixes:**

**A) etcd Certificate Issues**
```bash
# Check certificate validity
sudo kubeadm certs check-expiration

# Renew certificates if needed
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

**B) Disk Space Full**
```bash
df -h
# etcd is sensitive to disk I/O

# Clean up if needed
sudo apt-get clean
```

**C) Port Conflicts**
```bash
sudo netstat -tulpn | grep -E '2379|2380'
# Ensure only etcd is using these ports
```

---

### Issue 6: Worker Node Join Fails

**Symptoms:**
```bash
sudo kubeadm join ...
# Error: [ERROR FileAvailable--etc-kubernetes-kubelet.conf]
```

**Common Causes & Fixes:**

**A) Node Already Initialized**
```bash
# Reset node completely
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config

# Rejoin
sudo kubeadm join ...
```

**B) Token Expired**
```bash
# On control plane, generate new token
kubeadm token create --print-join-command

# Use new command on worker
```

**C) Control Plane Not Reachable**
```bash
# From worker, test connectivity
ping k8s-control-plane
telnet k8s-control-plane 6443

# Fix /etc/hosts if needed
```

**D) Port Already In Use**
```bash
# Check if kubelet is already running
sudo systemctl stop kubelet
sudo kubeadm join ...
```

---

### Issue 7: Calico Pods CrashLoopBackOff

**Symptoms:**
```bash
kubectl get pods -n calico-system
# Shows CrashLoopBackOff
```

**Diagnosis:**
```bash
kubectl logs -n calico-system <calico-pod> --previous
# Check crash logs

kubectl describe pod -n calico-system <calico-pod>
```

**Common Causes & Fixes:**

**A) IP Forwarding Not Enabled**
```bash
sysctl net.ipv4.ip_forward
# Should be 1

# If 0:
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

**B) Kernel Modules Not Loaded**
```bash
lsmod | grep -E 'overlay|br_netfilter'

# Load if missing:
sudo modprobe overlay
sudo modprobe br_netfilter
```

**C) CIDR Mismatch**
```bash
# Verify pod CIDR matches Calico configuration
kubectl get installation default -o yaml | grep cidr

# Should match --pod-network-cidr used in kubeadm init
```

**Fix:** If mismatch, edit installation:
```bash
kubectl edit installation default
# Change cidr to match (192.168.0.0/16 for default Calico)
```

---

### General Troubleshooting Commands

```bash
# Get detailed pod info
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container instance

# Get logs for multi-container pod
kubectl logs <pod-name> -c <container-name>

# Exec into pod
kubectl exec -it <pod-name> -- /bin/sh

# Get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n kube-system --sort-by='.lastTimestamp'

# Check resource usage (requires metrics-server)
kubectl top nodes
kubectl top pods

# Node logs
sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f

# Container runtime debugging
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock images
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock logs <container-id>

# Network troubleshooting from pod
kubectl run debug --image=nicolaka/netshoot -it --rm
# Inside pod: ping, curl, nslookup, traceroute available
```

---

## Common Mistakes & Best Practices

### Mistakes to Avoid

1. **Forgetting to disable swap permanently**
   - Symptom: kubelet fails after reboot
   - Fix: Always edit `/etc/fstab`

2. **Not setting SystemdCgroup in containerd config**
   - Symptom: Cryptic pod creation failures
   - Fix: `SystemdCgroup = true` in `/etc/containerd/config.toml`

3. **Wrong pod CIDR in kubeadm init**
   - Symptom: Calico networking fails
   - Fix: Use `--pod-network-cidr=192.168.0.0/16` for Calico

4. **Not saving join token**
   - Fix: `kubeadm token create --print-join-command`

5. **Security group ports not open**
   - Symptom: Nodes can't communicate
   - Fix: Review AWS security group rules

6. **Mixing cgroup drivers (systemd vs cgroupfs)**
   - Symptom: kubelet/containerd conflicts
   - Fix: Both must use systemd

7. **Running kubeadm init multiple times without reset**
   - Fix: `sudo kubeadm reset -f` before re-initializing

8. **Not checking logs when troubleshooting**
   - Always check: `journalctl -u kubelet`, `kubectl describe`, `kubectl logs`

---

### Best Practices

1. **Version Lock Kubernetes Packages**
   ```bash
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

2. **Use Labels for Organization**
   ```bash
   kubectl label nodes k8s-worker-1 node-role.kubernetes.io/worker=worker
   ```

3. **Set Resource Requests and Limits**
   - Prevents resource starvation
   - Enables proper scheduling

4. **Use Namespaces for Isolation**
   ```bash
   kubectl create namespace dev
   kubectl create namespace prod
   ```

5. **Monitor Cluster Health**
   - Install metrics-server
   - Use Prometheus + Grafana in production

6. **Backup etcd Regularly**
   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot save backup.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```

7. **Use ConfigMaps and Secrets**
   - Don't hardcode configuration in images

8. **Implement RBAC**
   - Principle of least privilege
   - Create service accounts for apps

9. **Use Readiness and Liveness Probes**
   - Kubernetes knows when pods are healthy

10. **Document Your Setup**
    - Keep track of CNI plugin version
    - Note custom configurations

---

## Quick Reference Commands

### Cluster Management
```bash
# View cluster info
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces

# Drain node (maintenance)
kubectl drain <node-name> --ignore-daemonsets

# Uncordon node
kubectl uncordon <node-name>

# Delete node from cluster
kubectl delete node <node-name>

# Reset node (on the node itself)
sudo kubeadm reset -f
```

### Pod Management
```bash
# Create pod
kubectl run nginx --image=nginx

# Get pods
kubectl get pods -o wide

# Delete pod
kubectl delete pod <pod-name>

# Force delete
kubectl delete pod <pod-name> --force --grace-period=0

# Exec into pod
kubectl exec -it <pod-name> -- bash

# Port forward
kubectl port-forward <pod-name> 8080:80
```

### Debugging
```bash
# Describe resource
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Logs
kubectl logs <pod-name> -f
kubectl logs <pod-name> --previous

# Events
kubectl get events --sort-by='.lastTimestamp'

# Top resources
kubectl top nodes
kubectl top pods
```

### Networking
```bash
# Get services
kubectl get svc

# Get endpoints
kubectl get endpoints

# Test DNS
kubectl run test --image=busybox -it --rm -- nslookup kubernetes.default
```

---

## Additional Resources

### Official Documentation
- Kubernetes Docs: https://kubernetes.io/docs/
- kubeadm Setup: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- Calico Docs: https://docs.tigera.io/calico/

### Useful Tools
- **k9s:** Terminal UI for Kubernetes
- **kubectx/kubens:** Switch contexts/namespaces easily
- **stern:** Multi-pod log tailing
- **helm:** Package manager for Kubernetes

### Install kubectl autocompletion
```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

---

## Summary

You've successfully set up a production-style Kubernetes cluster with:
- ✅ 3-node cluster (1 control plane + 2 workers)
- ✅ containerd as container runtime
- ✅ kubeadm for cluster bootstrapping
- ✅ Calico CNI for networking
- ✅ Proper system configuration (swap, sysctl, modules)
- ✅ Understanding of Kubernetes architecture
- ✅ Troubleshooting knowledge

**Next Steps for Students:**
1. Deploy sample applications
2. Learn about Services (ClusterIP, NodePort, LoadBalancer)
3. Explore ConfigMaps and Secrets
4. Study Ingress controllers
5. Learn about StatefulSets and DaemonSets
6. Implement monitoring (Prometheus/Grafana)
7. Practice RBAC and network policies
8. Learn cluster upgrades with kubeadm

---

**Remember:** Kubernetes is complex, but each component has a clear purpose. Understand the "why" behind each step, and troubleshooting becomes logical rather than magical.

Happy Learning! 🚀

---

## 🧑‍💻 Author

*Md. Sarowar Alam*  
Lead DevOps Engineer, Hogarth Worldwide  
📧 Email: sarowar@hotmail.com  
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/

---
