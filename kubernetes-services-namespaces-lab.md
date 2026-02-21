# Kubernetes Services & Namespaces: Real-World Lab
## Hands-On Tutorial for 3-Node kubeadm Cluster on AWS EC2

---

## 📋 Table of Contents

1. [Kubernetes Services - Networking Fundamentals](#kubernetes-services---networking-fundamentals)
2. [Service Types Deep Dive](#service-types-deep-dive)
   - [ClusterIP Service](#clusterip-service)
   - [NodePort Service](#nodeport-service)
   - [LoadBalancer Service](#loadbalancer-service)
3. [Namespaces for Resource Isolation](#namespaces-for-resource-isolation)
4. [Mini Real-World Project](#mini-real-world-project)
5. [Production Best Practices](#production-best-practices)
6. [Common Interview Questions](#common-interview-questions)

---

## Kubernetes Services - Networking Fundamentals

### The Problem Services Solve

**Without Services:**

```
Frontend Pod (192.168.1.5) → Backend Pod (192.168.2.10)
                              ↓
                        Pod dies, new pod created
                        New IP: 192.168.2.15
                              ↓
                        Frontend can't find backend!
```

**Challenges:**
- ❌ Pod IP addresses are ephemeral (change on restart)
- ❌ Multiple backend pods - which one to call?
- ❌ No load balancing between pods
- ❌ External access to cluster is complex

**With Services:**

```
Frontend Pod → Service (Stable IP: 10.96.100.50) → Backend Pods
               (kubernetes.default.svc.cluster.local)    ↓
                                                   Load balances to:
                                                   - Pod 1 (192.168.2.10)
                                                   - Pod 2 (192.168.2.11)
                                                   - Pod 3 (192.168.2.12)
```

**Solutions:**
- ✅ Stable IP address and DNS name
- ✅ Load balancing across multiple pods
- ✅ Service discovery (find services by name)
- ✅ External access (NodePort, LoadBalancer)

---

### What is a Kubernetes Service?

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them. Services enable loose coupling between microservices.

**Key Concepts:**
- **Stable Endpoint:** Services provide a stable IP/DNS name
- **Label Selector:** Services find pods using labels
- **Load Balancing:** Distributes traffic across healthy pods
- **Service Discovery:** Pods can find services via DNS

---

### Service Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      SERVICE                                │
│                                                             │
│  Name: backend-service                                      │
│  ClusterIP: 10.96.100.50                                    │
│  DNS: backend-service.default.svc.cluster.local             │
│  Selector: app=backend                                      │
│                                                             │
│  ┌───────────────────────────────────────────────────┐    │
│  │              Endpoints                             │    │
│  │  (Pods matching selector)                          │    │
│  │                                                    │    │
│  │  Pod 1: 192.168.1.10:8080 ◄─┐                    │    │
│  │  Pod 2: 192.168.1.11:8080   ├─ Load Balanced     │    │
│  │  Pod 3: 192.168.1.12:8080 ◄─┘                    │    │
│  └───────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

### Real-World Analogy: Hotel Reception

Think of a **Service** as a hotel reception desk:

- **Service** = Reception desk (stable location)
- **Pods** = Hotel staff members (can change shifts)
- **Guests (clients)** always go to reception desk, don't need to know which specific staff member is working
- Reception desk routes requests to available staff
- Staff members can be replaced, but reception desk location stays the same

---

### Service Types Overview

| Type           | Purpose                              | Access From         | Use Case                    |
|----------------|--------------------------------------|---------------------|-----------------------------|
| **ClusterIP**  | Internal cluster access only         | Within cluster      | Backend APIs, Databases     |
| **NodePort**   | External access via Node IP:Port     | External (limited)  | Development, Testing        |
| **LoadBalancer**| External access via cloud LB        | External (production)| Production web apps        |
| **ExternalName**| Maps service to external DNS        | Within cluster      | External databases          |

---

## Service Types Deep Dive

### ClusterIP Service

**ClusterIP** is the default service type. It exposes the service on an internal IP within the cluster.

**Characteristics:**
- Accessible only from within the cluster
- Gets a stable virtual IP (ClusterIP)
- Has DNS name: `<service-name>.<namespace>.svc.cluster.local`
- Used for internal microservice communication

**Network Flow:**

```
┌──────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                    │
│                                                          │
│  ┌────────────────┐                                     │
│  │  Frontend Pod  │                                     │
│  │                │                                     │
│  │  curl backend- │                                     │
│  │  service:8080  │                                     │
│  └────────┬───────┘                                     │
│           │                                             │
│           ▼                                             │
│  ┌────────────────────────┐                            │
│  │  ClusterIP Service     │                            │
│  │  backend-service       │                            │
│  │  10.96.100.50:8080     │                            │
│  └───────┬────────────────┘                            │
│          │                                              │
│          │ Load Balances                                │
│          ▼                                              │
│  ┌──────────────────────────────────────┐             │
│  │  Backend Pods (Deployment)           │             │
│  │  - Pod 1: 192.168.1.10:8080          │             │
│  │  - Pod 2: 192.168.1.11:8080          │             │
│  │  - Pod 3: 192.168.1.12:8080          │             │
│  └──────────────────────────────────────┘             │
│                                                          │
│  ❌ External access: NOT POSSIBLE                       │
└──────────────────────────────────────────────────────────┘
```

---

#### Demo 1: ClusterIP Service

**Step 1: Create Backend Deployment**

Create `backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          cat > server.js << 'EOF'
          const http = require('http');
          const os = require('os');
          
          const server = http.createServer((req, res) => {
            const response = {
              message: 'Hello from Backend API',
              pod: os.hostname(),
              timestamp: new Date().toISOString(),
              path: req.url
            };
            res.writeHead(200, {'Content-Type': 'application/json'});
            res.end(JSON.stringify(response, null, 2));
          });
          
          server.listen(8080);
          console.log('Backend server running on port 8080');
          EOF
          node server.js
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**Step 2: Create ClusterIP Service**

Create `backend-clusterip-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  type: ClusterIP  # Default, can be omitted
  selector:
    app: backend    # Must match pod labels
  ports:
  - port: 8080         # Service port
    targetPort: 8080   # Container port
    protocol: TCP
    name: http
```

**Service YAML Breakdown:**

| Field           | Purpose                                           |
|-----------------|---------------------------------------------------|
| `type: ClusterIP` | Service type (internal only)                    |
| `selector`      | Which pods this service routes traffic to         |
| `port`          | Port on the service (what clients connect to)     |
| `targetPort`    | Port on the pod (where container is listening)    |
| `protocol`      | TCP or UDP                                        |

**Step 3: Apply Resources**

```bash
# Create deployment
kubectl apply -f backend-deployment.yaml

# Verify pods
kubectl get pods -l app=backend

# Create service
kubectl apply -f backend-clusterip-service.yaml

# Verify service
kubectl get svc backend-service

# Get detailed service info
kubectl describe svc backend-service
```

**Output:**
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
backend-service   ClusterIP   10.96.100.50    <none>        8080/TCP   10s
```

**Step 4: Verify Endpoints**

```bash
# Check endpoints (should show 3 pod IPs)
kubectl get endpoints backend-service

# Describe endpoints
kubectl describe endpoints backend-service
```

**Output:**
```
NAME              ENDPOINTS                                         AGE
backend-service   192.168.1.10:8080,192.168.1.11:8080,192.168.1.12:8080   1m
```

**Step 5: Test Internal Access**

```bash
# Create test pod to access service
kubectl run test-pod --image=busybox:1.35 --rm -it --restart=Never -- sh

# Inside test pod:
wget -qO- http://backend-service:8080
# Should return JSON response

# Test load balancing - call multiple times
for i in 1 2 3 4 5; do
  echo "Request $i:"
  wget -qO- http://backend-service:8080 | grep pod
  echo ""
done
# You'll see different pod names (load balancing in action!)

# Test DNS resolution
nslookup backend-service
# Should resolve to ClusterIP

# Test FQDN (Fully Qualified Domain Name)
wget -qO- http://backend-service.default.svc.cluster.local:8080

exit
```

**Step 6: Test from Another Pod**

```bash
# Create nginx pod
kubectl run nginx-test --image=nginx:1.21

# Exec into nginx pod
kubectl exec -it nginx-test -- bash

# Inside nginx pod:
curl http://backend-service:8080
# Should work!

# Install tools if needed
apt-get update && apt-get install -y curl dnsutils

# Check DNS
nslookup backend-service

exit
```

---

#### Understanding Service DNS

Kubernetes provides automatic DNS for services:

**DNS Format:**
```
<service-name>.<namespace>.svc.<cluster-domain>

Examples:
- Short form: backend-service
- Namespaced: backend-service.default
- FQDN: backend-service.default.svc.cluster.local
```

**When to use which:**
- **Same namespace:** `backend-service`
- **Different namespace:** `backend-service.production`
- **FQDN:** When you need absolute certainty

---

### NodePort Service

**NodePort** exposes the service on each Node's IP at a static port (30000-32767 range).

**Characteristics:**
- Accessible from outside the cluster
- Uses Node IP + NodePort
- Automatically creates ClusterIP service
- Port range: 30000-32767 (configurable)

**Network Flow:**

```
┌────────────────────────────────────────────────────────────┐
│                   EXTERNAL ACCESS                          │
│                                                            │
│  Internet/User                                             │
│       │                                                    │
│       │ http://<node-ip>:30080                             │
│       ▼                                                    │
│  ┌────────────────────────────────────────────────┐      │
│  │         AWS Security Group                     │      │
│  │  - Allow port 30080 from your IP               │      │
│  └────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
                          ▼
┌────────────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER                            │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐   ┌──────────────┐│
│  │ Control Plane│    │  Worker 1    │   │  Worker 2    ││
│  │ Port: 30080  │    │ Port: 30080  │   │ Port: 30080  ││
│  └──────┬───────┘    └──────┬───────┘   └──────┬───────┘│
│         │                   │                   │         │
│         └───────────────────┴───────────────────┘         │
│                             ▼                             │
│                  ┌────────────────────┐                   │
│                  │  NodePort Service  │                   │
│                  │  Port: 8080        │                   │
│                  │  NodePort: 30080   │                   │
│                  └─────────┬──────────┘                   │
│                            │                              │
│                  ┌─────────▼──────────┐                   │
│                  │   ClusterIP        │                   │
│                  │   10.96.100.50     │                   │
│                  └─────────┬──────────┘                   │
│                            │ Load Balances                │
│                            ▼                              │
│                  ┌─────────────────────────┐             │
│                  │  Backend Pods           │             │
│                  │  - Pod 1: .1.10:8080    │             │
│                  │  - Pod 2: .1.11:8080    │             │
│                  │  - Pod 3: .1.12:8080    │             │
│                  └─────────────────────────┘             │
└────────────────────────────────────────────────────────────┘
```

---

#### Demo 2: NodePort Service

**Step 1: Create NodePort Service**

Create `backend-nodeport-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport-service
  labels:
    app: backend
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 8080         # ClusterIP port
    targetPort: 8080   # Pod port
    nodePort: 30080    # Node port (optional, auto-assigned if omitted)
    protocol: TCP
    name: http
```

**NodePort YAML Breakdown:**

| Field        | Purpose                                        |
|--------------|------------------------------------------------|
| `type: NodePort` | Makes service accessible via Node IP       |
| `port`       | Internal cluster port                          |
| `targetPort` | Pod container port                             |
| `nodePort`   | External port on nodes (30000-32767)          |

**Step 2: Apply Service**

```bash
# Create NodePort service (keeping existing ClusterIP service too)
kubectl apply -f backend-nodeport-service.yaml

# Get service details
kubectl get svc backend-nodeport-service

# Describe service
kubectl describe svc backend-nodeport-service
```

**Output:**
```
NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
backend-nodeport-service   NodePort   10.96.100.51    <none>        8080:30080/TCP   10s
```

**Note:** Shows both ClusterIP (10.96.100.51) and NodePort (30080)

---

**Step 3: Configure AWS Security Group**

```bash
# In AWS Console:
# 1. Go to EC2 → Security Groups
# 2. Select your k8s-cluster-sg security group
# 3. Add Inbound Rule:
#    - Type: Custom TCP
#    - Port: 30080
#    - Source: My IP (or 0.0.0.0/0 for testing, but not recommended)
```

**Security Consideration:** Always restrict source IP for NodePort access in production!

---

**Step 4: Get Node IPs**

```bash
# Get node external IPs
kubectl get nodes -o wide

# Or get public IP from AWS Console
# Or use AWS CLI:
# aws ec2 describe-instances --filters "Name=tag:Name,Values=k8s-worker-*" --query 'Reservations[*].Instances[*].[PublicIpAddress]' --output text
```

---

**Step 5: Access Service Externally**

```bash
# From your local machine (or anywhere with internet access)
curl http://<node-public-ip>:30080

# Example:
curl http://18.123.45.67:30080

# Or in browser:
http://18.123.45.67:30080
```

**Response:**
```json
{
  "message": "Hello from Backend API",
  "pod": "backend-deployment-abc123",
  "timestamp": "2026-02-21T10:30:00.000Z",
  "path": "/"
}
```

**Test Load Balancing:**
```bash
# Call multiple times - see different pods respond
for i in {1..10}; do
  curl http://18.123.45.67:30080 | grep pod
done
```

---

**Step 6: Access via ANY Node**

**Important:** NodePort makes service accessible on ALL nodes, even if pods aren't on that node.

```bash
# Get all node IPs
kubectl get nodes -o wide

# Try accessing via control plane IP
curl http://<control-plane-public-ip>:30080
# Works!

# Try accessing via worker-1 IP
curl http://<worker-1-public-ip>:30080
# Works!

# Try accessing via worker-2 IP
curl http://<worker-2-public-ip>:30080
# Works!
```

**How?** kube-proxy on each node forwards traffic to the service, which load-balances to pods.

---

**Step 7: Check Endpoints**

```bash
# Endpoints show actual pod IPs
kubectl get endpoints backend-nodeport-service

# Test scaling
kubectl scale deployment backend-deployment --replicas=5

# Wait 10 seconds, check endpoints again
kubectl get endpoints backend-nodeport-service
# Should show 5 endpoints now!

# Test external access still works
curl http://<node-ip>:30080
```

---

#### NodePort Use Cases

**✅ Good for:**
- Development and testing
- Small deployments
- POC/demos
- Direct access when LoadBalancer isn't available

**❌ Not ideal for:**
- Production (use LoadBalancer instead)
- Many services (port exhaustion)
- Security-sensitive apps (exposes on fixed ports)

---

### LoadBalancer Service

**LoadBalancer** provisions an external load balancer from the cloud provider (AWS, GCP, Azure).

**Characteristics:**
- Automatically provisions cloud load balancer (AWS ELB/ALB/NLB)
- Provides external IP address
- Automatically creates NodePort and ClusterIP
- Production-grade external access

**Network Flow:**

```
┌────────────────────────────────────────────────────────────┐
│                   EXTERNAL ACCESS                          │
│                                                            │
│  Internet/Users                                            │
│       │                                                    │
│       │ http://a1234567890.us-east-1.elb.amazonaws.com    │
│       │ (AWS Load Balancer DNS)                            │
│       ▼                                                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │         AWS Elastic Load Balancer                │    │
│  │  - Health checks                                 │    │
│  │  - TLS termination                               │    │
│  │  - Distributes traffic across nodes              │    │
│  └────────────────┬─────────────────────────────────┘    │
└───────────────────┼──────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER (AWS)                      │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐   ┌──────────────┐│
│  │ Worker 1     │    │  Worker 2    │   │  Worker 3    ││
│  │ NodePort     │    │ NodePort     │   │ NodePort     ││
│  └──────┬───────┘    └──────┬───────┘   └──────┬───────┘│
│         │                   │                   │         │
│         └───────────────────┴───────────────────┘         │
│                             ▼                             │
│                  ┌────────────────────┐                   │
│                  │ LoadBalancer Svc   │                   │
│                  │ (Creates NodePort) │                   │
│                  └─────────┬──────────┘                   │
│                            │                              │
│                  ┌─────────▼──────────┐                   │
│                  │   ClusterIP        │                   │
│                  └─────────┬──────────┘                   │
│                            │                              │
│                            ▼                              │
│                  ┌─────────────────────────┐             │
│                  │  Backend Pods           │             │
│                  └─────────────────────────┘             │
└────────────────────────────────────────────────────────────┘
```

---

#### Demo 3: LoadBalancer Service (AWS Behavior)

**Step 1: Create LoadBalancer Service**

Create `backend-loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-loadbalancer-service
  labels:
    app: backend
  annotations:
    # AWS-specific annotations (optional)
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Network Load Balancer
    # service.beta.kubernetes.io/aws-load-balancer-type: "elb"  # Classic Load Balancer (default)
    # service.beta.kubernetes.io/aws-load-balancer-internal: "true"  # Internal LB
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80           # Load balancer port
    targetPort: 8080   # Pod port
    protocol: TCP
    name: http
```

**Step 2: Apply Service**

```bash
# Create service
kubectl apply -f backend-loadbalancer-service.yaml

# Watch service creation (wait for EXTERNAL-IP)
kubectl get svc backend-loadbalancer-service -w

# This may take 2-3 minutes...
```

**Output:**
```
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
backend-loadbalancer-service    LoadBalancer   10.96.100.52    <pending>       80:32456/TCP   10s
backend-loadbalancer-service    LoadBalancer   10.96.100.52    a1234567.elb... 80:32456/TCP   2m
```

---

**Step 3: Access via Load Balancer**

```bash
# Get external IP/DNS
kubectl get svc backend-loadbalancer-service

# Access via load balancer
curl http://<external-lb-dns>

# Example:
curl http://a1234567890.us-east-1.elb.amazonaws.com
```

---

**Step 4: Check in AWS Console**

```bash
# In AWS Console:
# 1. Go to EC2 → Load Balancers
# 2. You should see a new load balancer
# 3. Check:
#    - Target groups
#    - Health checks
#    - Listeners
```

---

#### AWS LoadBalancer Behavior

**What Kubernetes Does:**
1. Creates Service with `type: LoadBalancer`
2. Cloud Controller Manager detects this
3. Calls AWS API to provision ELB/NLB
4. Registers worker nodes as targets
5. Updates Service with external IP

**Important Notes:**

**💰 Cost:** AWS Load Balancers cost money (~$20-50/month per LB)!

**🕐 Creation Time:** Takes 2-5 minutes to provision

**🔧 LoadBalancer Types:**
- **Classic Load Balancer (ELB):** Default, older
- **Network Load Balancer (NLB):** Better performance, static IP
- **Application Load Balancer (ALB):** Requires AWS Load Balancer Controller (separate installation)

**📝 Annotations:**
```yaml
annotations:
  # Use NLB
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  
  # Internal LB (private subnet only)
  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
  
  # Specify subnets
  service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-12345,subnet-67890"
  
  # SSL certificate
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
```

---

**Step 5: Cleanup (To Avoid AWS Charges)**

```bash
# Delete LoadBalancer service
kubectl delete -f backend-loadbalancer-service.yaml

# Wait 2-3 minutes, verify in AWS Console that LB is deleted
```

**⚠️ Important:** Always delete LoadBalancer services when done to avoid charges!

---

### Service Comparison Table

| Feature           | ClusterIP         | NodePort          | LoadBalancer      |
|-------------------|-------------------|-------------------|-------------------|
| **Access**        | Internal only     | External (limited)| External (prod)   |
| **IP Assignment** | ClusterIP         | ClusterIP + NodePort | ClusterIP + NodePort + External IP |
| **Port Range**    | Any               | 30000-32767       | Any (on LB)       |
| **Use Case**      | Microservices     | Dev/Test          | Production        |
| **Cloud Cost**    | Free              | Free              | $$$ (LB charges)  |
| **Setup Time**    | Instant           | Instant           | 2-5 minutes       |
| **HA**            | Yes               | Manual            | Automatic         |
| **Security**      | Best (internal)   | Moderate          | Good (LB features)|

---

### Service Type Decision Tree

```
Need external access?
│
├─ No → Use ClusterIP
│      (Internal microservices)
│
└─ Yes → In production?
         │
         ├─ No → Use NodePort
         │      (Dev, testing, demos)
         │
         └─ Yes → Use LoadBalancer
                 (Production web apps)
```

---

## Namespaces for Resource Isolation

### What are Namespaces?

**Namespaces** provide a mechanism for isolating groups of resources within a single cluster.

**Key Concepts:**
- Virtual clusters within physical cluster
- Isolate teams, projects, or environments
- Apply resource quotas and limits
- Control access via RBAC

**Real-World Analogy: Office Building**

- **Cluster** = Office building
- **Namespaces** = Different floors/departments
- Each department (namespace) has its own:
  - Space allocation (resource quotas)
  - Access control (RBAC)
  - Resources (pods, services)
- But they share building infrastructure (nodes, storage)

---

### Namespace Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                        │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Namespace: default                                │   │
│  │  - Pods                                            │   │
│  │  - Services                                        │   │
│  │  - Deployments                                     │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Namespace: development                            │   │
│  │  - app-v1 (Deployment)                             │   │
│  │  - app-service (Service)                           │   │
│  │  - Resource Quota: CPU=2, Memory=4Gi               │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Namespace: staging                                │   │
│  │  - app-v2 (Deployment)                             │   │
│  │  - app-service (Service) - Same name, different NS!│   │
│  │  - Resource Quota: CPU=4, Memory=8Gi               │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Namespace: production                             │   │
│  │  - app-v2 (Deployment)                             │   │
│  │  - app-service (Service)                           │   │
│  │  - Resource Quota: CPU=8, Memory=16Gi              │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  Namespace: kube-system (Built-in)                 │   │
│  │  - kube-apiserver, kube-scheduler, etc.            │   │
│  └────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### Default Namespaces

Kubernetes comes with these namespaces:

| Namespace       | Purpose                                      |
|-----------------|----------------------------------------------|
| **default**     | Default namespace for resources              |
| **kube-system** | Kubernetes system components                 |
| **kube-public** | Public resources (readable by all)           |
| **kube-node-lease** | Node heartbeat leases                    |

---

### Why Use Namespaces?

**Problems Without Namespaces:**
- ❌ All resources in one namespace (messy)
- ❌ Name collisions (can't have 2 services named "api")
- ❌ No resource isolation (one team can consume all resources)
- ❌ Hard to manage permissions

**Benefits With Namespaces:**
- ✅ Logical separation (dev, staging, prod)
- ✅ Resource quotas per namespace
- ✅ Different teams/projects isolated
- ✅ RBAC per namespace
- ✅ Easy cleanup (delete namespace = delete everything in it)

---

### Demo 4: Working with Namespaces

**Step 1: View Existing Namespaces**

```bash
# List all namespaces
kubectl get namespaces
# OR short form:
kubectl get ns

# Get detailed info
kubectl describe namespace default
```

**Output:**
```
NAME              STATUS   AGE
default           Active   5d
kube-node-lease   Active   5d
kube-public       Active   5d
kube-system       Active   5d
```

---

**Step 2: Create Namespaces**

**Method 1: Imperative**

```bash
# Create development namespace
kubectl create namespace development

# Create staging namespace
kubectl create namespace staging

# Create production namespace
kubectl create namespace production

# Verify
kubectl get ns
```

**Method 2: Declarative**

Create `namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: prod
```

```bash
# Apply
kubectl apply -f namespaces.yaml

# Verify
kubectl get ns --show-labels
```

---

**Step 3: Deploy App in Multiple Namespaces**

Create `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 2
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
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

**Deploy to each namespace:**

```bash
# Deploy to development
kubectl apply -f app-deployment.yaml -n development

# Deploy to staging
kubectl apply -f app-deployment.yaml -n staging

# Deploy to production
kubectl apply -f app-deployment.yaml -n production

# Verify deployments in all namespaces
kubectl get deployments --all-namespaces | grep web-app
```

---

**Step 4: View Resources by Namespace**

```bash
# Get pods in default namespace
kubectl get pods

# Get pods in development namespace
kubectl get pods -n development

# Get pods in all namespaces
kubectl get pods --all-namespaces
# OR short form:
kubectl get pods -A

# Get specific resources in namespace
kubectl get deployment,svc,pods -n staging

# Describe resource in namespace
kubectl describe deployment web-app -n production
```

---

**Step 5: Switch Default Namespace Context**

Instead of typing `-n namespace` every time, switch context:

```bash
# View current context
kubectl config current-context

# Set namespace for current context
kubectl config set-context --current --namespace=development

# Verify
kubectl config view --minify | grep namespace

# Now kubectl commands default to 'development' namespace
kubectl get pods
# Shows pods in development namespace

# Switch to staging
kubectl config set-context --current --namespace=staging

# Switch back to default
kubectl config set-context --current --namespace=default
```

---

**Step 6: Cross-Namespace Communication**

**Test accessing service in different namespace:**

```bash
# Create test pod in development namespace
kubectl run test-pod -n development --image=busybox:1.35 --rm -it --restart=Never -- sh

# Inside test pod:

# Access service in same namespace (works)
wget -qO- http://web-service

# Access service in different namespace
wget -qO- http://web-service.staging.svc.cluster.local

# Access service in production namespace
wget -qO- http://web-service.production.svc.cluster.local

exit
```

**DNS Format for Cross-Namespace:**
```
<service-name>.<namespace>.svc.cluster.local
```

---

### Demo 5: Resource Quotas

**Purpose:** Limit resources a namespace can consume.

Create `dev-resource-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"        # Max 2 CPU cores requested
    requests.memory: 4Gi     # Max 4GB memory requested
    limits.cpu: "4"          # Max 4 CPU cores limit
    limits.memory: 8Gi       # Max 8GB memory limit
    pods: "10"               # Max 10 pods
    services: "5"            # Max 5 services
    persistentvolumeclaims: "3"  # Max 3 PVCs
```

```bash
# Apply resource quota
kubectl apply -f dev-resource-quota.yaml

# View quota
kubectl get resourcequota -n development

# Describe quota (shows usage)
kubectl describe resourcequota dev-quota -n development
```

**Output:**
```
NAME        AGE   REQUEST                                      LIMIT
dev-quota   10s   pods: 2/10, requests.cpu: 200m/2, requests.memory: 128Mi/4Gi, ...
```

**Test Quota Enforcement:**

```bash
# Try to create deployment exceeding quota
kubectl run quota-test -n development --image=nginx --replicas=20

# Will fail when quota is exceeded!
# Check events:
kubectl get events -n development --sort-by='.lastTimestamp'
```

---

### Demo 6: Limit Ranges

**Purpose:** Set default and maximum resource requests/limits for pods.

Create `dev-limit-range.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - max:                # Maximum per container
      cpu: "1"
      memory: "1Gi"
    min:                # Minimum per container
      cpu: "100m"
      memory: "64Mi"
    default:            # Default limits (if not specified)
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:     # Default requests (if not specified)
      cpu: "250m"
      memory: "256Mi"
    type: Container
  - max:                # Maximum per pod
      cpu: "2"
      memory: "2Gi"
    type: Pod
```

```bash
# Apply limit range
kubectl apply -f dev-limit-range.yaml

# View limit ranges
kubectl get limitranges -n development

# Describe
kubectl describe limitrange dev-limits -n development
```

**Test:**

```bash
# Create pod without resource specifications
# It will get default values from LimitRange
kubectl run test-limits -n development --image=nginx

# Check its resources
kubectl get pod test-limits -n development -o yaml | grep -A 10 resources
```

---

### Namespace Best Practices

**✅ DO:**
1. **Use namespaces for environments:** dev, staging, prod
2. **Use namespaces for teams:** team-a, team-b
3. **Apply resource quotas** to prevent resource hogging
4. **Use RBAC** for namespace-level access control
5. **Label namespaces** for easy identification
6. **Use convention:** consistent naming (kebab-case)

**❌ DON'T:**
1. **Don't over-namespace:** Too many namespaces = complexity
2. **Don't ignore kube-system:** Don't deploy user apps there
3. **Don't forget cleanup:** Delete unused namespaces
4. **Don't rely solely on namespaces for security:** Use NetworkPolicies too

---

### Delete Namespaces

```bash
# Delete namespace (deletes ALL resources in it!)
kubectl delete namespace development

# This deletes:
# - All pods
# - All services
# - All deployments
# - All configmaps/secrets
# - Everything in that namespace

# To be safe, first check what's in namespace
kubectl get all -n development

# Then delete
kubectl delete namespace development
```

**⚠️ Warning:** Deleting a namespace deletes everything in it! No confirmation prompt!

---

## Mini Real-World Project

### Project: E-Commerce Microservices

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    PRODUCTION NAMESPACE                     │
│                                                             │
│  ┌──────────────────────────────────────────────────┐     │
│  │           Internet Users                          │     │
│  └────────────────────┬─────────────────────────────┘     │
│                       │                                     │
│                       ▼                                     │
│            ┌─────────────────────┐                         │
│            │  Frontend Service   │                         │
│            │  (NodePort 30080)   │                         │
│            └──────────┬──────────┘                         │
│                       │                                     │
│                       ▼                                     │
│         ┌──────────────────────────┐                       │
│         │  Frontend Deployment     │                       │
│         │  - Nginx                 │                       │
│         │  - 3 replicas            │                       │
│         │  - Reverse proxy to API  │                       │
│         └──────────┬───────────────┘                       │
│                    │                                        │
│                    │ HTTP: http://backend-api-service:8080 │
│                    │                                        │
│                    ▼                                        │
│         ┌──────────────────────┐                           │
│         │ Backend API Service  │                           │
│         │ (ClusterIP)          │                           │
│         └──────────┬───────────┘                           │
│                    │                                        │
│                    ▼                                        │
│       ┌─────────────────────────────┐                      │
│       │  Backend API Deployment     │                      │
│       │  - Node.js API              │                      │
│       │  - 5 replicas (scaled)      │                      │
│       │  - Returns product data     │                      │
│       └─────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 1: Create Project Namespace

Create `project-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    project: ecommerce
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
    services: "20"
```

```bash
# Apply
kubectl apply -f project-namespace.yaml

# Set context to production namespace
kubectl config set-context --current --namespace=production
```

---

### Step 2: Deploy Backend API

Create `backend-api.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: production
  labels:
    app: backend-api
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
        tier: backend
    spec:
      containers:
      - name: api
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          cat > api.js << 'EOF'
          const http = require('http');
          const os = require('os');
          
          // Sample product data
          const products = [
            { id: 1, name: 'Laptop', price: 999.99, stock: 10 },
            { id: 2, name: 'Mouse', price: 29.99, stock: 50 },
            { id: 3, name: 'Keyboard', price: 79.99, stock: 30 },
            { id: 4, name: 'Monitor', price: 299.99, stock: 15 }
          ];
          
          let requestCount = 0;
          
          const server = http.createServer((req, res) => {
            requestCount++;
            
            // Enable CORS
            res.setHeader('Access-Control-Allow-Origin', '*');
            res.setHeader('Content-Type', 'application/json');
            
            if (req.url === '/api/products') {
              res.writeHead(200);
              res.end(JSON.stringify({
                products: products,
                meta: {
                  pod: os.hostname(),
                  timestamp: new Date().toISOString(),
                  requests: requestCount
                }
              }));
            } else if (req.url === '/api/health') {
              res.writeHead(200);
              res.end(JSON.stringify({ status: 'healthy' }));
            } else if (req.url === '/api/metrics') {
              res.writeHead(200);
              res.end(JSON.stringify({
                pod: os.hostname(),
                uptime: process.uptime(),
                memory: process.memoryUsage(),
                requests: requestCount
              }));
            } else {
              res.writeHead(404);
              res.end(JSON.stringify({ error: 'Not Found' }));
            }
          });
          
          server.listen(8080);
          console.log('Backend API server running on port 8080');
          console.log('Pod:', os.hostname());
          EOF
          node api.js
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-api-service
  namespace: production
  labels:
    app: backend-api
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: http
```

```bash
# Deploy backend
kubectl apply -f backend-api.yaml

# Verify
kubectl get deployment backend-api
kubectl get pods -l app=backend-api
kubectl get svc backend-api-service

# Test backend internally
kubectl run test --image=busybox:1.35 --rm -it --restart=Never -- wget -qO- http://backend-api-service:8080/api/products
```

---

### Step 3: Deploy Frontend

Create `frontend.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: production
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    http {
      server {
        listen 80;
        
        location / {
          root /usr/share/nginx/html;
          index index.html;
          try_files $uri $uri/ /index.html;
        }
        
        # Proxy API requests to backend
        location /api/ {
          proxy_pass http://backend-api-service:8080;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
      }
    }
  
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>E-Commerce Demo - Kubernetes</title>
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body {
                font-family: 'Segoe UI', Arial, sans-serif;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                min-height: 100vh;
                padding: 20px;
            }
            .container {
                max-width: 1200px;
                margin: 0 auto;
            }
            header {
                background: white;
                padding: 30px;
                border-radius: 10px;
                box-shadow: 0 10px 30px rgba(0,0,0,0.2);
                margin-bottom: 30px;
                text-align: center;
            }
            h1 {
                color: #667eea;
                margin-bottom: 10px;
            }
            .subtitle {
                color: #666;
                font-size: 14px;
            }
            .info-box {
                background: #fff;
                padding: 20px;
                border-radius: 10px;
                margin-bottom: 20px;
                box-shadow: 0 5px 15px rgba(0,0,0,0.1);
            }
            .products {
                display: grid;
                grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
                gap: 20px;
                margin-top: 20px;
            }
            .product-card {
                background: white;
                border-radius: 10px;
                padding: 20px;
                box-shadow: 0 5px 15px rgba(0,0,0,0.1);
                transition: transform 0.3s;
            }
            .product-card:hover {
                transform: translateY(-5px);
                box-shadow: 0 10px 25px rgba(0,0,0,0.2);
            }
            .product-name {
                font-size: 20px;
                font-weight: bold;
                color: #333;
                margin-bottom: 10px;
            }
            .product-price {
                font-size: 24px;
                color: #667eea;
                font-weight: bold;
                margin: 10px 0;
            }
            .product-stock {
                color: #666;
                font-size: 14px;
            }
            .meta {
                background: #f8f9fa;
                padding: 15px;
                border-radius: 5px;
                font-family: monospace;
                font-size: 12px;
                margin-top: 10px;
            }
            .loading {
                text-align: center;
                padding: 40px;
                color: white;
                font-size: 18px;
            }
            button {
                background: #667eea;
                color: white;
                border: none;
                padding: 10px 20px;
                border-radius: 5px;
                cursor: pointer;
                font-size: 14px;
                margin-top: 10px;
            }
            button:hover {
                background: #5568d3;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <header>
                <h1>🛒 E-Commerce Demo</h1>
                <div class="subtitle">Kubernetes Microservices Architecture</div>
            </header>
            
            <div class="info-box">
                <h2>Architecture Info</h2>
                <div class="meta" id="architecture">
                    <div>🎯 Frontend: Nginx (This page)</div>
                    <div>⚙️ Backend: Node.js API</div>
                    <div>🔗 Communication: ClusterIP Service</div>
                    <div>📦 Namespace: production</div>
                </div>
            </div>
            
            <div class="info-box">
                <h2>Products from Backend API</h2>
                <button onclick="loadProducts()">🔄 Reload Products</button>
                <div id="products" class="loading">Loading products...</div>
                <div class="meta" id="meta" style="display:none;"></div>
            </div>
        </div>
        
        <script>
            async function loadProducts() {
                const productsDiv = document.getElementById('products');
                const metaDiv = document.getElementById('meta');
                
                productsDiv.innerHTML = '<div class="loading">Loading products...</div>';
                
                try {
                    const response = await fetch('/api/products');
                    const data = await response.json();
                    
                    // Display products
                    let html = '<div class="products">';
                    data.products.forEach(product => {
                        html += `
                            <div class="product-card">
                                <div class="product-name">${product.name}</div>
                                <div class="product-price">$${product.price}</div>
                                <div class="product-stock">Stock: ${product.stock} units</div>
                            </div>
                        `;
                    });
                    html += '</div>';
                    productsDiv.innerHTML = html;
                    
                    // Display metadata
                    metaDiv.style.display = 'block';
                    metaDiv.innerHTML = `
                        <strong>Backend Response Metadata:</strong><br>
                        Pod: ${data.meta.pod}<br>
                        Timestamp: ${data.meta.timestamp}<br>
                        Requests Handled: ${data.meta.requests}
                    `;
                } catch (error) {
                    productsDiv.innerHTML = `<div class="loading" style="color: red;">Error: ${error.message}</div>`;
                }
            }
            
            // Load products on page load
            loadProducts();
            
            // Auto-refresh every 10 seconds to see load balancing
            setInterval(loadProducts, 10000);
        </script>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
  labels:
    app: frontend
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: frontend-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
  labels:
    app: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
    name: http
```

```bash
# Deploy frontend
kubectl apply -f frontend.yaml

# Verify
kubectl get deployment frontend
kubectl get pods -l app=frontend
kubectl get svc frontend-service
kubectl get configmap frontend-config
```

---

### Step 4: Access Application

```bash
# Get node IP
kubectl get nodes -o wide

# Access application in browser
http://<node-public-ip>:30080

# Or use curl
curl http://<node-public-ip>:30080
```

**You should see:**
- E-Commerce web page
- Products loaded from backend API
- Backend pod name in metadata (showing which pod responded)

---

### Step 5: Demonstrate Scaling

```bash
# Check current backend pods
kubectl get pods -l app=backend-api

# Scale backend to 5 replicas
kubectl scale deployment backend-api --replicas=5

# Watch scaling
kubectl get pods -l app=backend-api -w

# Verify 5 pods running
kubectl get pods -l app=backend-api

# Refresh browser multiple times - see different pod names in metadata!
# This demonstrates load balancing across pods
```

---

### Step 6: Test Internal Service Communication

```bash
# Exec into frontend pod
kubectl exec -it deployment/frontend -- sh

# Inside frontend pod:

# Test backend API
wget -qO- http://backend-api-service:8080/api/products

# Check service resolution
nslookup backend-api-service

# Full DNS name
wget -qO- http://backend-api-service.production.svc.cluster.local:8080/api/products

exit
```

---

### Step 7: View All Resources

```bash
# Get all resources in production namespace
kubectl get all -n production

# Get resources with labels
kubectl get all -n production --show-labels

# Get resource usage (if metrics-server installed)
kubectl top pods -n production
kubectl top nodes
```

---

### Step 8: Test High Availability

**Simulate pod failure:**

```bash
# Delete a backend pod
kubectl delete pod -l app=backend-api --force --grace-period=0

# Immediately refresh browser - app still works!
# Kubernetes automatically recreates the pod

# Check pods
kubectl get pods -l app=backend-api -w
```

**Simulate node failure:**

```bash
# Drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Pods are automatically rescheduled to other nodes
kubectl get pods -o wide

# App continues working!

# Uncordon node
kubectl uncordon <node-name>
```

---

### Step 9: Monitoring and Logs

```bash
# Get logs from all backend pods
kubectl logs -l app=backend-api --tail=20

# Follow logs in real-time
kubectl logs -f deployment/backend-api

# Get logs from specific container
kubectl logs -l app=backend-api -c api

# Get events
kubectl get events -n production --sort-by='.lastTimestamp'

# Describe deployment
kubectl describe deployment backend-api
kubectl describe deployment frontend
```

---

### Step 10: Cleanup Project

```bash
# Delete all resources in production namespace
kubectl delete namespace production

# This deletes:
# - backend-api deployment
# - frontend deployment
# - backend-api-service
# - frontend-service
# - frontend-config ConfigMap
# - All pods

# Verify cleanup
kubectl get all -n production
# Should show: No resources found in production namespace.
```

---

## Production Best Practices

### 1. Service Design

**✅ Best Practices:**

```yaml
# Always add labels
metadata:
  labels:
    app: my-app
    version: v1.0.0
    environment: production
    tier: backend

# Always add resource requests/limits
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"
  limits:
    memory: "256Mi"
    cpu: "500m"

# Always add health checks
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

### 2. Service Types by Environment

| Environment   | Service Type      | Rationale                            |
|---------------|-------------------|--------------------------------------|
| Development   | NodePort          | Easy access, no LB costs             |
| Staging       | LoadBalancer      | Test prod-like setup                 |
| Production    | LoadBalancer      | HA, TLS, proper external access      |
| Internal APIs | ClusterIP         | No external exposure needed          |

---

### 3. Namespace Strategy

**Recommended Structure:**

```
development/
  - team-a/
  - team-b/
staging/
production/
  - east-region/
  - west-region/
monitoring/
  - prometheus
  - grafana
logging/
  - elasticsearch
  - kibana
```

**Apply Resource Quotas:**

```yaml
# production-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.nodeports: "5"
    services.loadbalancers: "3"
    persistentvolumeclaims: "20"
```

---

### 4. Security Best Practices

**Service Security:**

```yaml
# Use NetworkPolicy to restrict traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Namespace Security:**

```yaml
# Apply RBAC for namespace access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### 5. Monitoring and Observability

```yaml
# Add Prometheus annotations
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"

# Add readiness/liveness endpoints
# /health - for health checks
# /ready - for readiness checks
# /metrics - for Prometheus
```

---

### 6. High Availability

**Multi-Replica Strategy:**

| Component Type | Minimum Replicas | Recommended     |
|----------------|------------------|-----------------|
| Stateless API  | 2                | 3-5             |
| Frontend       | 2                | 3               |
| Background Jobs| 1                | 2               |
| Database       | 1 (StatefulSet)  | 3 (with replication) |

**Pod Disruption Budget:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: backend-api
```

---

### 7. Resource Management

**Set Appropriate Limits:**

```yaml
# Under-provisioned (too small)
requests:
  memory: "64Mi"   # ❌ App needs 256Mi
  cpu: "50m"       # ❌ App needs 250m

# Over-provisioned (wasteful)
requests:
  memory: "4Gi"    # ❌ App only uses 256Mi
  cpu: "2"         # ❌ App only uses 250m

# Right-sized (optimal)
requests:
  memory: "256Mi"  # ✅ Matches app needs
  cpu: "250m"      # ✅ Matches app needs
limits:
  memory: "512Mi"  # ✅ 2x buffer
  cpu: "500m"      # ✅ 2x buffer
```

---

### 8. Service Naming Conventions

**Recommended Format:**
```
<app-name>-<service-type>-service

Examples:
- user-api-service
- payment-api-service
- frontend-service
- auth-service
```

**Labels:**
```yaml
labels:
  app: user-api
  tier: backend
  environment: production
  version: v2.3.1
  team: platform
```

---

### 9. Documentation

**Add Annotations:**

```yaml
metadata:
  annotations:
    description: "User authentication and authorization API"
    contact: "platform-team@company.com"
    runbook: "https://wiki.company.com/user-api-runbook"
    slack-channel: "#platform-alerts"
    pagerduty: "https://company.pagerduty.com/services/user-api"
```

---

### 10. Deployment Checklist

Before deploying to production:

- [ ] Resource requests/limits defined
- [ ] Health checks configured (liveness + readiness)
- [ ] Labels applied consistently
- [ ] Service type appropriate for environment
- [ ] Replica count >= 2 for HA
- [ ] Resource quotas applied to namespace
- [ ] NetworkPolicy configured (if needed)
- [ ] RBAC rules defined
- [ ] Monitoring/logging configured
- [ ] Documentation updated
- [ ] Tested in staging environment
- [ ] Rollback plan documented
- [ ] On-call team notified

---

## Common Interview Questions

### Beginner Level

**Q1: What is a Kubernetes Service and why do we need it?**

**A:** A Service is an abstraction that provides a stable network endpoint to access a set of Pods. We need it because:
- Pod IPs change when pods are recreated
- Services provide load balancing across multiple pods
- Services enable service discovery (DNS names)
- Services provide stable IP addresses

---

**Q2: What are the different types of Services?**

**A:**
- **ClusterIP** - Internal access only (default)
- **NodePort** - External access via Node IP:Port
- **LoadBalancer** - External access via cloud load balancer
- **ExternalName** - Maps service to external DNS name

---

**Q3: How does a Service find its Pods?**

**A:** Services use label selectors. The Service selector must match Pod labels. For example:

```yaml
# Service selector
selector:
  app: backend

# Pod labels
labels:
  app: backend
```

Kubernetes maintains an Endpoints object that lists all Pod IPs matching the selector.

---

**Q4: What is the difference between ClusterIP and NodePort?**

**A:**
- **ClusterIP:** Only accessible within the cluster, no external access
- **NodePort:** Accessible from outside via any Node's IP + NodePort (30000-32767)
- NodePort also creates a ClusterIP automatically

---

**Q5: What are Namespaces used for?**

**A:** Namespaces provide:
- Logical isolation of resources
- Resource quotas per namespace
- RBAC per namespace
- Separate environments (dev/staging/prod)
- Multi-tenancy (different teams/projects)

---

### Intermediate Level

**Q6: Explain the DNS name format for Services.**

**A:**
```
<service-name>.<namespace>.svc.<cluster-domain>

Examples:
- backend-service (same namespace)
- backend-service.production (cross-namespace)
- backend-service.production.svc.cluster.local (FQDN)
```

---

**Q7: How does load balancing work with Services?**

**A:** 
- kube-proxy on each node watches Services
- Creates iptables/IPVS rules
- Distributes traffic to backend pods using round-robin (default)
- Automatically excludes unhealthy pods (failed readiness probes)
- Works at Layer 4 (TCP/UDP)

---

**Q8: What happens when you scale a Deployment that has a Service?**

**A:**
1. Deployment creates new pods
2. New pods get labels matching Service selector
3. Service automatically adds new pod IPs to Endpoints
4. Traffic is load-balanced to new pods (no Service reconfiguration needed)

---

**Q9: What is the purpose of targetPort vs port in a Service?**

**A:**
- **port:** Port on the Service (what clients connect to)
- **targetPort:** Port on the Pod/Container (where app is listening)

Example:
```yaml
ports:
- port: 80           # Client connects to Service on port 80
  targetPort: 8080   # Service forwards to Pod on port 8080
```

---

**Q10: How do you restrict resources in a Namespace?**

**A:** Use ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    pods: "20"
```

And LimitRange for default values:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: my-limits
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    type: Container
```

---

### Advanced Level

**Q11: Explain how a LoadBalancer Service works in AWS.**

**A:**
1. You create Service with `type: LoadBalancer`
2. Cloud Controller Manager detects it
3. Calls AWS API to create ELB/NLB/ALB
4. Registers worker nodes as targets
5. Configures health checks pointing to NodePort
6. Updates Service with external IP/DNS
7. Traffic flow: Internet → Load Balancer → NodePort → Service → Pods

Cost: AWS charges for load balancer (~$20-50/month)

---

**Q12: What is the difference between Service and Ingress?**

**A:**
- **Service:** Layer 4 (TCP/UDP) load balancing
  - Creates separate load balancer per service
  - No host/path-based routing
  - No TLS termination

- **Ingress:** Layer 7 (HTTP/HTTPS) routing
  - Single load balancer for multiple services
  - Host/path-based routing
  - TLS termination
  - More cost-effective for multiple services

---

**Q13: How would you debug a Service that's not routing traffic?**

**A:**
1. Check Service selector matches Pod labels:
   ```bash
   kubectl describe svc my-service
   kubectl get pods --show-labels
   ```

2. Check Endpoints exist:
   ```bash
   kubectl get endpoints my-service
   ```
   If empty, selector doesn't match any pods

3. Check Pod readiness:
   ```bash
   kubectl get pods
   ```
   Only Ready pods are added to Endpoints

4. Test from within cluster:
   ```bash
   kubectl run test --image=busybox --rm -it -- wget -qO- http://my-service
   ```

5. Check network policies aren't blocking traffic

6. Check kube-proxy logs:
   ```bash
   kubectl logs -n kube-system -l k8s-app=kube-proxy
   ```

---

**Q14: What is headless Service and when would you use it?**

**A:** A headless Service (ClusterIP: None) doesn't allocate a cluster IP. It returns individual Pod IPs instead.

```yaml
spec:
  clusterIP: None
  selector:
    app: database
```

**Use cases:**
- StatefulSets (stable network identity)
- Client-side load balancing
- Direct pod-to-pod communication
- Service discovery without load balancing

DNS returns all Pod IPs instead of single Service IP.

---

**Q15: How do you implement zero-downtime deployments with Services?**

**A:**
1. **Use Deployment with RollingUpdate:**
   ```yaml
   strategy:
     type: RollingUpdate
     rollingUpdate:
       maxSurge: 1
       maxUnavailable: 0  # Keep all pods available
   ```

2. **Configure readiness probes:**
   - Service only routes to Ready pods
   - New pods won't receive traffic until ready

3. **Use Pod Disruption Budget:**
   ```yaml
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   spec:
     minAvailable: 2
   ```

4. **Graceful shutdown:**
   - Handle SIGTERM signal
   - Finish in-flight requests
   - Set `terminationGracePeriodSeconds: 30`

---

**Q16: Explain kube-proxy modes and their differences.**

**A:**

| Mode      | How it works                          | Performance | Use Case      |
|-----------|---------------------------------------|-------------|---------------|
| **iptables** | Creates iptables rules for routing | Good        | Default       |
| **IPVS**  | Uses IPVS for load balancing          | Better      | Large clusters|
| **userspace** | Proxies traffic in userspace       | Poor        | Legacy        |

**iptables mode:**
- Default mode
- Uses iptables NAT rules
- Random selection of backend pods

**IPVS mode:**
- More efficient for large clusters (>1000 services)
- Multiple load-balancing algorithms
- Better performance

Enable IPVS:
```yaml
# kube-proxy ConfigMap
mode: "ipvs"
```

---

**Q17: How would you set up cross-namespace service communication securely?**

**A:**

1. **Use FQDN for service:**
   ```yaml
   http://backend-api.production.svc.cluster.local:8080
   ```

2. **Apply NetworkPolicy:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-from-staging
     namespace: production
   spec:
     podSelector:
       matchLabels:
         app: backend-api
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             environment: staging
       ports:
       - protocol: TCP
         port: 8080
   ```

3. **Use service mesh (Istio/Linkerd) for:**
   - mTLS between services
   - Fine-grained authorization
   - Traffic encryption

---

**Q18: What are the implications of deleting a Service vs deleting a Deployment?**

**A:**

**Deleting Service:**
- Removes networking endpoint
- Pods still running
- Can't access pods via Service name/ClusterIP
- Can recreate Service (same selector will find existing pods)
- No downtime if pods are accessed directly

**Deleting Deployment:**
- Deletes ReplicaSet
- Deletes all Pods
- Service remains (but no endpoints)
- Causes downtime
- Service shows 0 endpoints

**Best practice:** Delete Deployment before Service during cleanup.

---

**Q19: How do you implement canary deployments with Services?**

**A:**

**Option 1: Multiple Services + Ingress**
```yaml
# Stable service (90% traffic)
apiVersion: v1
kind: Service
metadata:
  name: app-stable
spec:
  selector:
    app: myapp
    version: stable
---
# Canary service (10% traffic)
apiVersion: v1
kind: Service
metadata:
  name: app-canary
spec:
  selector:
    app: myapp
    version: canary
---
# Ingress with weighted routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

**Option 2: Service Mesh (Istio)**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
```

---

**Q20: Describe Session Affinity in Kubernetes Services.**

**A:** Session affinity (sticky sessions) ensures requests from same client go to same pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

**How it works:**
- Hash of client IP determines backend pod
- Useful for stateful applications
- **Limitation:** Only works with ClientIP (not cookie-based)

**Alternatives:**
- Use StatefulSets for truly stateful apps
- Use Service Mesh for advanced session affinity
- Implement application-level session management (Redis/Memcached)

---

## Quick Reference Card

### Service Commands

```bash
# Get services
kubectl get svc
kubectl get services --all-namespaces

# Describe service
kubectl describe svc <service-name>

# Get endpoints
kubectl get endpoints <service-name>

# Expose deployment as service
kubectl expose deployment <name> --port=80 --type=NodePort

# Delete service
kubectl delete svc <service-name>

# Test service from pod
kubectl run test --image=busybox --rm -it -- wget -qO- http://<service>
```

### Namespace Commands

```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace dev

# Set default namespace
kubectl config set-context --current --namespace=dev

# Delete namespace (and all resources in it!)
kubectl delete namespace dev

# Get resources in namespace
kubectl get all -n dev

# Get resources in all namespaces
kubectl get pods --all-namespaces
```

---

## Summary

**Services:**
- ✅ Provide stable networking for pods
- ✅ Enable load balancing
- ✅ Support service discovery
- ✅ Three main types: ClusterIP, NodePort, LoadBalancer
- ✅ Use ClusterIP for internal, LoadBalancer for production external access

**Namespaces:**
- ✅ Logical isolation of resources
- ✅ Enable multi-tenancy
- ✅ Apply resource quotas
- ✅ RBAC per namespace
- ✅ Use for environments: dev, staging, prod

**Production Tips:**
- Use LoadBalancer in production
- Apply resource quotas to all namespaces
- Implement NetworkPolicies
- Monitor service endpoints
- Document service dependencies
- Plan for high availability (multiple replicas)

---

**Congratulations! 🎉**

You now understand Kubernetes Services and Namespaces with hands-on experience. You've built a complete microservices application with proper isolation and networking!

**Next Topics to Explore:**
1. Ingress Controllers
2. NetworkPolicies
3. StatefulSets
4. ConfigMaps & Secrets
5. Persistent Volumes
6. Helm Charts
7. Service Mesh (Istio/Linkerd)

Keep practicing and building! 🚀

---

## 🧑‍💻 Author

*Md. Sarowar Alam*  
Lead DevOps Engineer, Hogarth Worldwide  
📧 Email: sarowar@hotmail.com  
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/

---
