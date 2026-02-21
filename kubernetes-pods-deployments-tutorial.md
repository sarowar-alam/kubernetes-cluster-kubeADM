# Kubernetes Core Objects: Pods, ReplicaSets & Deployments
## Hands-On Tutorial with Real-World Examples

---

## 📋 Table of Contents

1. [Understanding Kubernetes Objects and Resources](#understanding-kubernetes-objects-and-resources)
2. [Pods – The Smallest Deployable Unit](#pods--the-smallest-deployable-unit)
3. [Creating and Managing Pods](#creating-and-managing-pods)
4. [ReplicaSets – High Availability](#replicasets--high-availability)
5. [Deployments – Managing Updates and Rollbacks](#deployments--managing-updates-and-rollbacks)
6. [Imperative vs Declarative Approach](#imperative-vs-declarative-approach)
7. [Complete Demo Workflow](#complete-demo-workflow)
8. [Mini Assignment](#mini-assignment)
9. [Troubleshooting Scenarios](#troubleshooting-scenarios)

---

## Understanding Kubernetes Objects and Resources

### What Are Kubernetes Objects?

**Kubernetes Objects** are persistent entities in the Kubernetes system. They represent the **desired state** of your cluster.

**Key Concept:** You describe what you want (desired state), and Kubernetes works continuously to make the actual state match it.

```
┌─────────────────────────────────────────────────────────┐
│              KUBERNETES CONTROL LOOP                    │
│                                                         │
│   Desired State          Kubernetes         Actual     │
│   (What you want)   ────►   Engine    ────► State     │
│                              │                          │
│                              │                          │
│                              └──────────────────────┐   │
│                                                     │   │
│                    Continuously monitors and        │   │
│                    makes corrections  ◄─────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Core Kubernetes Objects Hierarchy

```
┌────────────────────────────────────────────────────────┐
│                    DEPLOYMENT                          │
│  (Manages updates, rollbacks, versioning)              │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │           REPLICASET                          │    │
│  │  (Ensures N pods are running)                 │    │
│  │                                               │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │    │
│  │  │  Pod 1  │  │  Pod 2  │  │  Pod 3  │     │    │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │     │    │
│  │  │ │ App │ │  │ │ App │ │  │ │ App │ │     │    │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │     │    │
│  │  └─────────┘  └─────────┘  └─────────┘     │    │
│  └──────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

---

### Real-World Analogy: Restaurant Service

Think of a restaurant:

- **Pod** = Individual waiter serving customers
- **ReplicaSet** = Restaurant policy: "Always have 5 waiters on duty"
- **Deployment** = Restaurant manager who:
  - Hires/trains new waiters
  - Gradually replaces old staff with trained ones
  - Can roll back if new staff isn't working out
  - Maintains the "5 waiters" policy automatically

---

### Common Kubernetes Objects

| Object         | Purpose                                    | Analogy                    |
|----------------|--------------------------------------------|----------------------------|
| **Pod**        | Basic unit of deployment                   | Single container/app       |
| **ReplicaSet** | Maintains N identical pods                 | Backup generators          |
| **Deployment** | Manages ReplicaSets, updates, rollbacks    | Update manager             |
| **Service**    | Stable network endpoint for pods           | Load balancer              |
| **ConfigMap**  | Configuration data                         | Settings file              |
| **Secret**     | Sensitive data (passwords, tokens)         | Vault                      |
| **Volume**     | Storage for pods                           | Hard drive                 |
| **Namespace**  | Virtual cluster isolation                  | Project folders            |

---

## Pods – The Smallest Deployable Unit

### What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

**Key Points:**
- A pod can contain one or more containers
- Containers in a pod share:
  - Network namespace (same IP address)
  - Storage volumes
  - Lifecycle
- Pods are ephemeral (temporary) - they can die and be replaced

---

### Pod Architecture

```
┌────────────────────────────────────────────────┐
│                    POD                         │
│  IP: 192.168.1.10                              │
│                                                │
│  ┌──────────────┐         ┌─────────────┐    │
│  │  Container 1 │         │ Container 2 │    │
│  │              │         │             │    │
│  │   nginx      │◄───────►│  log-agent  │    │
│  │   :80        │  shared │   :9000     │    │
│  │              │  network│             │    │
│  └──────────────┘         └─────────────┘    │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │      Shared Volume                    │    │
│  │      /var/log/nginx                   │    │
│  └──────────────────────────────────────┘    │
└────────────────────────────────────────────────┘
```

---

### Problem Pods Solve

**Without Pods (Traditional Deployment):**
- Deploy app directly on servers
- Manual scaling (add more servers)
- Manual recovery if app crashes
- Hard to move between environments

**With Pods:**
- ✅ Abstraction layer over containers
- ✅ Self-healing (automatic restart)
- ✅ Portable (runs anywhere Kubernetes runs)
- ✅ Co-located containers can share resources
- ✅ Easy to scale and manage

---

### Real-World Analogy: Delivery Van

A **Pod** is like a delivery van:
- The van itself is the Pod
- Packages inside are Containers
- Multiple packages can share the same van (same IP, same journey)
- If the van breaks down, a new van (Pod) is dispatched with the same packages
- Each van has a unique license plate (IP address)

---

## Creating and Managing Pods

### Method 1: Imperative Approach (Quick Testing)

**Imperative** = Give direct commands (like ordering food verbally)

```bash
# Create a pod with nginx
kubectl run nginx-pod --image=nginx:1.21

# Verify pod creation
kubectl get pods

# Get detailed information
kubectl get pods -o wide

# Describe pod (shows events, status, IP)
kubectl describe pod nginx-pod

# Check pod logs
kubectl logs nginx-pod

# Execute command inside pod
kubectl exec -it nginx-pod -- /bin/bash
# Inside pod:
# curl localhost
# exit

# Delete pod
kubectl delete pod nginx-pod
```

**Output Example:**
```
NAME        READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-pod   1/1     Running   0          30s   192.168.1.5   k8s-worker-1
```

---

### Method 2: Declarative Approach (Production Standard)

**Declarative** = Describe what you want in a file (like written order)

Create a file: `nginx-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: demo
spec:
  containers:
  - name: nginx-container
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
```

**YAML Breakdown:**

| Field                  | Purpose                                      |
|------------------------|----------------------------------------------|
| `apiVersion: v1`       | API version for this object                  |
| `kind: Pod`            | Type of object                               |
| `metadata.name`        | Unique name for this pod                     |
| `metadata.labels`      | Key-value pairs for identification           |
| `spec.containers`      | List of containers in this pod               |
| `image`                | Docker image to use                          |
| `ports.containerPort`  | Port exposed by container                    |
| `resources.requests`   | Minimum resources needed                     |
| `resources.limits`     | Maximum resources allowed                    |

---

### Apply and Manage the Pod

```bash
# Create pod from YAML
kubectl apply -f nginx-pod.yaml

# Verify creation
kubectl get pods

# Get pod with labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=nginx

# Describe pod (detailed info)
kubectl describe pod nginx-pod

# View logs
kubectl logs nginx-pod

# Follow logs (real-time)
kubectl logs -f nginx-pod

# Execute command
kubectl exec nginx-pod -- nginx -v

# Interactive shell
kubectl exec -it nginx-pod -- /bin/bash

# Port forward to access locally
kubectl port-forward nginx-pod 8080:80
# Access: http://localhost:8080

# Delete pod
kubectl delete -f nginx-pod.yaml
# OR
kubectl delete pod nginx-pod
```

---

### Demo 1: Pod Lifecycle and Self-Healing

Let's see what happens when a pod crashes:

```bash
# Create pod
kubectl apply -f nginx-pod.yaml

# Watch pod status
kubectl get pods -w
# Press Ctrl+C to stop watching

# In another terminal, kill the container
kubectl exec nginx-pod -- kill 1

# Observe pod restart
kubectl get pods
# RESTARTS column will increment

# Check events
kubectl describe pod nginx-pod | grep -A 10 Events
```

**What Happened?**
1. Container died
2. kubelet detected the failure
3. kubelet restarted the container automatically
4. Pod got a new container but kept the same Pod identity

**Important:** Pod self-heals by restarting containers, but if the Pod itself is deleted, it's gone forever (unless managed by ReplicaSet/Deployment).

---

### Demo 2: Multi-Container Pod

Create `multi-container-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  - name: log-sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /logs/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /logs
  
  volumes:
  - name: shared-logs
    emptyDir: {}
```

**Use Case:** Sidecar pattern - log collector running alongside main app.

```bash
# Create pod
kubectl apply -f multi-container-pod.yaml

# Check pod (should show 2/2 containers)
kubectl get pods

# View logs from specific container
kubectl logs multi-container-pod -c nginx
kubectl logs multi-container-pod -c log-sidecar

# Exec into specific container
kubectl exec -it multi-container-pod -c nginx -- /bin/bash

# Delete pod
kubectl delete -f multi-container-pod.yaml
```

---

### Demo 3: Node.js Application Pod

Create `nodejs-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-app
  labels:
    app: nodejs
    tier: backend
spec:
  containers:
  - name: nodejs-container
    image: node:18-alpine
    command: ["sh", "-c"]
    args:
    - |
      cat > server.js << 'EOF'
      const http = require('http');
      const os = require('os');
      
      const server = http.createServer((req, res) => {
        res.writeHead(200, {'Content-Type': 'text/html'});
        res.end(`
          <h1>Hello from Node.js Pod!</h1>
          <p>Hostname: ${os.hostname()}</p>
          <p>Node Version: ${process.version}</p>
          <p>Request URL: ${req.url}</p>
        `);
      });
      
      server.listen(3000, () => {
        console.log('Server running on port 3000');
      });
      EOF
      node server.js
    ports:
    - containerPort: 3000
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

```bash
# Create pod
kubectl apply -f nodejs-pod.yaml

# Verify
kubectl get pods

# Check logs
kubectl logs nodejs-app

# Port forward to access
kubectl port-forward nodejs-app 3000:3000

# In browser or another terminal:
curl http://localhost:3000

# Delete pod
kubectl delete -f nodejs-pod.yaml
```

---

### Problems with Standalone Pods

**❌ Issues:**
1. **No high availability:** If pod dies, it's gone (unless kubelet restarts it on the same node)
2. **No scaling:** Can't easily create multiple copies
3. **Manual management:** You manage each pod individually
4. **Node failure:** If node dies, pod is lost forever

**💡 Solution:** Use **ReplicaSets** (or better, **Deployments**)

---

## ReplicaSets – High Availability

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of pod replicas are running at all times.

**Key Features:**
- Maintains N identical pods
- Replaces failed pods automatically
- Scales pods up or down
- Uses label selectors to identify pods it manages

---

### ReplicaSet Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      REPLICASET                         │
│              Desired Replicas: 3                        │
│                                                         │
│  Selector: app=nginx                                    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   Pod 1     │  │   Pod 2     │  │   Pod 3     │   │
│  │ app=nginx   │  │ app=nginx   │  │ app=nginx   │   │
│  │ nginx:1.21  │  │ nginx:1.21  │  │ nginx:1.21  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
│                                                         │
│  If Pod 2 dies → ReplicaSet creates new Pod 2          │
└─────────────────────────────────────────────────────────┘
```

---

### Problem ReplicaSet Solves

**Without ReplicaSet:**
- ❌ Manual pod creation for each replica
- ❌ Manual replacement if pod fails
- ❌ Manual scaling
- ❌ No guarantee of availability

**With ReplicaSet:**
- ✅ Automatic pod creation
- ✅ Self-healing (replaces failed pods)
- ✅ Easy scaling (change replica count)
- ✅ High availability guaranteed

---

### Real-World Analogy: Security Guards

Imagine a building that needs **3 security guards** on duty 24/7:

- **ReplicaSet** = Security agency with a contract: "Maintain 3 guards always"
- **Pods** = Individual guards
- If one guard gets sick (pod dies), the agency immediately sends a replacement
- If you need more security (scale up), tell the agency to send 5 guards instead
- The agency doesn't care which specific people, just that there are always the right number

---

### Create ReplicaSet: Declarative Approach

Create `nginx-replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
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
```

**ReplicaSet YAML Breakdown:**

| Field                      | Purpose                                           |
|----------------------------|---------------------------------------------------|
| `kind: ReplicaSet`         | Object type                                       |
| `spec.replicas`            | Desired number of pods                            |
| `spec.selector.matchLabels`| How ReplicaSet finds its pods                     |
| `spec.template`            | Pod template (exactly like Pod YAML)             |
| `template.metadata.labels` | Must match selector (critical!)                   |

**⚠️ CRITICAL:** `selector.matchLabels` MUST match `template.metadata.labels`

---

### Apply and Manage ReplicaSet

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# Verify ReplicaSet
kubectl get replicasets
# OR short form:
kubectl get rs

# Get pods created by ReplicaSet
kubectl get pods

# Detailed view
kubectl get pods -o wide

# Describe ReplicaSet
kubectl describe rs nginx-replicaset

# Get pods with labels
kubectl get pods --show-labels
```

**Output Example:**
```
NAME                      DESIRED   CURRENT   READY   AGE
nginx-replicaset          3         3         3       45s

NAME                           READY   STATUS    RESTARTS   AGE
nginx-replicaset-abc12         1/1     Running   0          45s
nginx-replicaset-def34         1/1     Running   0          45s
nginx-replicaset-ghi56         1/1     Running   0          45s
```

---

### Demo 4: ReplicaSet Self-Healing

**Test automatic pod replacement:**

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# Watch pods
kubectl get pods -w

# In another terminal, delete a pod
kubectl delete pod <pod-name>

# Watch as ReplicaSet immediately creates a new pod!
# The pods list will briefly show 2/3, then back to 3/3
```

**What Happens:**
1. You delete a pod
2. ReplicaSet controller notices: "I have 2 pods but need 3"
3. ReplicaSet immediately creates a new pod
4. Total is back to 3 pods

**Key Insight:** You can't permanently reduce pods by deleting them. The ReplicaSet will always maintain the desired count.

---

### Demo 5: Scaling ReplicaSet

**Method 1: Imperative Scaling**

```bash
# Scale up to 5 replicas
kubectl scale rs nginx-replicaset --replicas=5

# Verify
kubectl get rs
kubectl get pods

# Scale down to 2 replicas
kubectl scale rs nginx-replicaset --replicas=2

# Verify (some pods will be terminated)
kubectl get pods
```

**Method 2: Declarative Scaling**

```bash
# Edit YAML file, change replicas: 3 to replicas: 6
nano nginx-replicaset.yaml

# Apply changes
kubectl apply -f nginx-replicaset.yaml

# Verify
kubectl get rs
kubectl get pods
```

**Best Practice:** Always use declarative approach (YAML files) in production for version control.

---

### Demo 6: Node Failure Simulation

```bash
# Create ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# Check which nodes pods are on
kubectl get pods -o wide

# Simulate node failure (drain a node)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Watch pods migrate to other nodes
kubectl get pods -o wide -w

# Uncordon node (make it schedulable again)
kubectl uncordon <node-name>
```

**What Happens:**
- Pods on drained node are evicted
- ReplicaSet sees pod count is below desired
- New pods are scheduled on healthy nodes
- Application remains available!

---

### Delete ReplicaSet

```bash
# Delete ReplicaSet AND all its pods
kubectl delete -f nginx-replicaset.yaml

# OR
kubectl delete rs nginx-replicaset

# Verify all pods are gone
kubectl get pods
```

**Important:** Deleting a ReplicaSet deletes all pods it manages.

---

### Node.js ReplicaSet Example

Create `nodejs-replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nodejs-replicaset
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nodejs-api
  template:
    metadata:
      labels:
        app: nodejs-api
    spec:
      containers:
      - name: nodejs
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          cat > app.js << 'EOF'
          const http = require('http');
          const os = require('os');
          
          const server = http.createServer((req, res) => {
            res.writeHead(200, {'Content-Type': 'application/json'});
            res.end(JSON.stringify({
              message: 'Hello from Node.js ReplicaSet',
              pod: os.hostname(),
              timestamp: new Date().toISOString()
            }));
          });
          
          server.listen(3000);
          console.log('Server running on port 3000');
          EOF
          node app.js
        ports:
        - containerPort: 3000
```

```bash
# Apply
kubectl apply -f nodejs-replicaset.yaml

# Verify 4 pods
kubectl get pods -l app=nodejs-api

# Test from one pod to another
kubectl exec <pod-1-name> -- wget -qO- http://<pod-2-ip>:3000

# Scale to 6
kubectl scale rs nodejs-replicaset --replicas=6

# Verify
kubectl get pods -l app=nodejs-api

# Delete
kubectl delete -f nodejs-replicaset.yaml
```

---

### Limitations of ReplicaSet

**❌ Issues:**
1. **No rolling updates:** Can't gradually update to new image version
2. **No rollback:** Can't undo updates
3. **No revision history:** Can't track changes
4. **Manual image updates:** Have to delete/recreate pods for new image

**💡 Solution:** Use **Deployments** (which manage ReplicaSets for you)

---

## Deployments – Managing Updates and Rollbacks

### What is a Deployment?

A **Deployment** is a higher-level object that manages ReplicaSets and provides declarative updates, rollbacks, and revision history.

**Key Features:**
- Creates and manages ReplicaSets automatically
- Rolling updates (gradual pod replacement)
- Rollback to previous versions
- Revision history
- Pause/resume deployments
- Declarative updates

---

### Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     DEPLOYMENT                           │
│              (Update & Rollback Manager)                 │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │        ReplicaSet v2 (Current)                   │   │
│  │        replicas: 3                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │   │
│  │  │  Pod v2  │  │  Pod v2  │  │  Pod v2  │      │   │
│  │  │nginx:1.22│  │nginx:1.22│  │nginx:1.22│      │   │
│  │  └──────────┘  └──────────┘  └──────────┘      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │        ReplicaSet v1 (Old - scaled to 0)        │   │
│  │        replicas: 0                               │   │
│  │                                                  │   │
│  │  (Kept for rollback capability)                 │   │
│  └─────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

### Problem Deployment Solves

**Without Deployment (using ReplicaSet directly):**
- ❌ Update requires deleting all pods (downtime)
- ❌ No gradual rollout
- ❌ Can't rollback easily
- ❌ No version history
- ❌ Manual ReplicaSet management

**With Deployment:**
- ✅ Zero-downtime updates (rolling update)
- ✅ Gradual rollout (controlled update)
- ✅ Easy rollback (one command)
- ✅ Revision history (track changes)
- ✅ Automatic ReplicaSet management

---

### Real-World Analogy: Software Release Manager

Think of updating software on multiple servers:

- **Deployment** = Release manager with a plan
- **ReplicaSets** = Groups of servers on different versions
- **Rolling Update** = Update servers gradually (not all at once)
- **Rollback** = If new version has bugs, quickly revert to old version
- **Revision History** = Log of all updates made

**Example:**
- You have 10 servers running App v1.0
- Deployment creates new ReplicaSet for App v2.0
- Gradually moves traffic: 8→v1.0, 2→v2.0, then 6→v1.0, 4→v2.0, until 0→v1.0, 10→v2.0
- If v2.0 has bugs, rollback instantly to v1.0

---

### Create Deployment: Declarative Approach

Create `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired during update
      maxUnavailable: 1  # Max pods unavailable during update
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
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
```

**Deployment YAML Breakdown:**

| Field                        | Purpose                                    |
|------------------------------|--------------------------------------------|
| `kind: Deployment`           | Object type                                |
| `spec.replicas`              | Desired number of pods                     |
| `spec.strategy.type`         | Update strategy (RollingUpdate or Recreate)|
| `maxSurge`                   | Extra pods during update                   |
| `maxUnavailable`             | Pods that can be down during update        |
| `spec.selector`              | How Deployment finds its pods              |
| `spec.template`              | Pod template (same as ReplicaSet)          |

---

### Apply and Manage Deployment

```bash
# Create Deployment
kubectl apply -f nginx-deployment.yaml

# Get Deployments
kubectl get deployments
# OR short form:
kubectl get deploy

# Get ReplicaSets (Deployment creates it automatically)
kubectl get rs

# Get Pods
kubectl get pods

# Get all related resources
kubectl get deploy,rs,pods

# Describe Deployment
kubectl describe deployment nginx-deployment

# Check rollout status
kubectl rollout status deployment/nginx-deployment
```

**Output Example:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           1m

NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7d8f6c8d9    3         3         3       1m

NAME                                READY   STATUS    AGE
nginx-deployment-7d8f6c8d9-abc12    1/1     Running   1m
nginx-deployment-7d8f6c8d9-def34    1/1     Running   1m
nginx-deployment-7d8f6c8d9-ghi56    1/1     Running   1m
```

**Notice:**
- Deployment created a ReplicaSet
- ReplicaSet created 3 pods
- Pod names include ReplicaSet hash

---

### Demo 7: Deployment Self-Healing

```bash
# Create Deployment
kubectl apply -f nginx-deployment.yaml

# Delete a pod
kubectl delete pod <pod-name>

# Watch automatic replacement
kubectl get pods -w

# Delete multiple pods
kubectl delete pod --all

# All pods recreated automatically!
kubectl get pods
```

**What Happens:**
1. Deployment owns ReplicaSet
2. ReplicaSet owns Pods
3. If pod deleted → ReplicaSet recreates it
4. If ReplicaSet deleted → Deployment recreates it

**Try deleting ReplicaSet:**
```bash
kubectl delete rs <replicaset-name>

# Deployment immediately creates a new ReplicaSet!
kubectl get rs
kubectl get pods
```

---

### Demo 8: Scaling Deployment

**Method 1: Imperative**

```bash
# Scale to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Verify
kubectl get deploy
kubectl get pods

# Scale to 10
kubectl scale deployment nginx-deployment --replicas=10

# Verify
kubectl get pods -l app=nginx

# Scale back to 3
kubectl scale deployment nginx-deployment --replicas=3

# Verify
kubectl get pods
```

**Method 2: Declarative**

```bash
# Edit YAML: change replicas: 3 to replicas: 6
nano nginx-deployment.yaml

# Apply
kubectl apply -f nginx-deployment.yaml

# Verify
kubectl get deploy
```

---

### Demo 9: Rolling Update (Zero Downtime Update)

**Scenario:** Update nginx from 1.21 to 1.22

**Method 1: Update YAML file**

```bash
# Edit nginx-deployment.yaml
# Change: image: nginx:1.21
# To:     image: nginx:1.22

nano nginx-deployment.yaml

# Apply update
kubectl apply -f nginx-deployment.yaml

# Watch rollout in real-time
kubectl rollout status deployment/nginx-deployment

# Or watch pods change
kubectl get pods -w
```

**Method 2: Imperative update**

```bash
# Update image directly
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Watch rollout
kubectl rollout status deployment/nginx-deployment

# Verify new image
kubectl describe deployment nginx-deployment | grep Image
```

---

### What Happens During Rolling Update?

```
Step 1: Initial State (3 pods, nginx:1.21)
┌─────────────────────────────────────────┐
│ ReplicaSet v1 (nginx:1.21)              │
│ [Pod 1] [Pod 2] [Pod 3]                 │
└─────────────────────────────────────────┘

Step 2: Deployment creates new ReplicaSet
┌─────────────────────────────────────────┐
│ ReplicaSet v1 (nginx:1.21) - 3 pods     │
│ [Pod 1] [Pod 2] [Pod 3]                 │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│ ReplicaSet v2 (nginx:1.22) - 0 pods     │
└─────────────────────────────────────────┘

Step 3: Gradual rollout (maxSurge:1, maxUnavailable:1)
┌─────────────────────────────────────────┐
│ ReplicaSet v1 - 2 pods                  │
│ [Pod 1] [Pod 2]                         │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│ ReplicaSet v2 - 1 pod                   │
│ [Pod 4 NEW]                             │
└─────────────────────────────────────────┘

Step 4: Continue gradual rollout
┌─────────────────────────────────────────┐
│ ReplicaSet v1 - 0 pods                  │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│ ReplicaSet v2 - 3 pods                  │
│ [Pod 4] [Pod 5] [Pod 6]                 │
└─────────────────────────────────────────┘

Final: Old ReplicaSet kept (scaled to 0) for rollback
```

**Key Parameters:**
- **maxSurge: 1** = Can have max 4 pods during update (3 + 1)
- **maxUnavailable: 1** = Can have min 2 pods available (3 - 1)

---

### Demo 10: Monitor Rollout in Detail

```bash
# Start rollout
kubectl set image deployment/nginx-deployment nginx=nginx:1.23

# Watch rollout status
kubectl rollout status deployment/nginx-deployment

# Watch pods change (in another terminal)
kubectl get pods -w

# Check ReplicaSets
kubectl get rs
# You'll see 2 ReplicaSets:
# - Old one scaled to 0
# - New one scaled to 3

# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Check specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

**Output Example (rollout history):**
```
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.23
```

---

### Demo 11: Rollback to Previous Version

**Scenario:** New version (1.23) has bugs. Rollback to 1.22.

```bash
# Check current image
kubectl describe deployment nginx-deployment | grep Image

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Watch rollback
kubectl rollout status deployment/nginx-deployment

# Verify image version
kubectl describe deployment nginx-deployment | grep Image

# Check history
kubectl rollout history deployment/nginx-deployment
```

**What Happens:**
1. Deployment scales up old ReplicaSet
2. Deployment scales down new ReplicaSet
3. Zero downtime!

**Rollback to Specific Revision:**

```bash
# View history with revisions
kubectl rollout history deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verify
kubectl describe deployment nginx-deployment | grep Image
```

---

### Demo 12: Pause and Resume Deployment

**Use Case:** Make multiple changes before triggering rollout.

```bash
# Pause deployment
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes (no rollout triggered yet)
kubectl set image deployment/nginx-deployment nginx=nginx:1.24
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=256Mi

# Resume deployment (now rollout happens with all changes)
kubectl rollout resume deployment/nginx-deployment

# Watch rollout
kubectl rollout status deployment/nginx-deployment
```

---

### Node.js Deployment Example

Create `nodejs-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-api-deployment
  labels:
    app: nodejs-api
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nodejs-api
  template:
    metadata:
      labels:
        app: nodejs-api
        version: v1
    spec:
      containers:
      - name: nodejs
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          cat > server.js << 'EOF'
          const http = require('http');
          const os = require('os');
          
          const VERSION = 'v1.0.0';
          
          const server = http.createServer((req, res) => {
            if (req.url === '/health') {
              res.writeHead(200);
              res.end('OK');
            } else {
              res.writeHead(200, {'Content-Type': 'application/json'});
              res.end(JSON.stringify({
                message: 'Hello from Node.js Deployment',
                version: VERSION,
                pod: os.hostname(),
                timestamp: new Date().toISOString(),
                uptime: process.uptime()
              }, null, 2));
            }
          });
          
          server.listen(3000);
          console.log(`Server ${VERSION} running on port 3000`);
          EOF
          node server.js
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 3
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**Apply and test:**

```bash
# Create deployment
kubectl apply -f nodejs-deployment.yaml

# Verify
kubectl get deploy
kubectl get pods

# Port forward to test
kubectl port-forward deployment/nodejs-api-deployment 3000:3000

# In another terminal or browser:
curl http://localhost:3000

# Output:
# {
#   "message": "Hello from Node.js Deployment",
#   "version": "v1.0.0",
#   "pod": "nodejs-api-deployment-abc123",
#   "timestamp": "2026-02-21T10:30:00.000Z",
#   "uptime": 45.123
# }
```

---

### Demo 13: Update Node.js App Version

**Update to v2.0.0:**

```bash
# Edit nodejs-deployment.yaml
# Change: const VERSION = 'v1.0.0';
# To:     const VERSION = 'v2.0.0';

# Also update label:
# version: v2

# Apply update
kubectl apply -f nodejs-deployment.yaml

# Watch rollout
kubectl rollout status deployment/nodejs-api-deployment

# Test new version
kubectl port-forward deployment/nodejs-api-deployment 3000:3000
curl http://localhost:3000
# Should show "version": "v2.0.0"

# Check history
kubectl rollout history deployment/nodejs-api-deployment
```

---

### Demo 14: Rollback Node.js App

```bash
# Current version
curl http://localhost:3000 | grep version
# Shows v2.0.0

# Rollback
kubectl rollout undo deployment/nodejs-api-deployment

# Wait for rollback to complete
kubectl rollout status deployment/nodejs-api-deployment

# Test again
curl http://localhost:3000 | grep version
# Shows v1.0.0
```

---

### Delete Deployment

```bash
# Delete deployment (also deletes ReplicaSets and Pods)
kubectl delete -f nodejs-deployment.yaml

# OR
kubectl delete deployment nodejs-api-deployment

# Verify everything is gone
kubectl get deploy,rs,pods
```

---

## Imperative vs Declarative Approach

### Comparison Table

| Aspect              | Imperative                          | Declarative                        |
|---------------------|-------------------------------------|------------------------------------|
| **Style**           | Give commands                       | Describe desired state             |
| **Example**         | `kubectl run nginx --image=nginx`   | `kubectl apply -f nginx-pod.yaml`  |
| **Use Case**        | Quick testing, debugging            | Production, version control        |
| **Reproducibility** | Hard to reproduce                   | Easy (YAML in git)                 |
| **Collaboration**   | Difficult                           | Easy (team reviews YAML)           |
| **Automation**      | Harder to automate                  | GitOps friendly                    |
| **Tracking Changes**| No history                          | Git commit history                 |

---

### Imperative Commands (Quick Reference)

```bash
# Create pod
kubectl run nginx --image=nginx:1.21

# Create deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Expose deployment as service
kubectl expose deployment nginx --port=80 --type=NodePort

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# Autoscale
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80

# Delete resources
kubectl delete pod nginx
kubectl delete deployment nginx
```

---

### Declarative Approach (Production Standard)

```bash
# Apply configuration (create or update)
kubectl apply -f nginx-deployment.yaml

# Apply entire directory
kubectl apply -f ./kubernetes-manifests/

# Apply with recursive directory
kubectl apply -f ./kubernetes-manifests/ --recursive

# Diff before applying
kubectl diff -f nginx-deployment.yaml

# Dry run (validate without applying)
kubectl apply -f nginx-deployment.yaml --dry-run=client

# Server-side validation
kubectl apply -f nginx-deployment.yaml --dry-run=server

# Delete resources
kubectl delete -f nginx-deployment.yaml
```

---

### Best Practices

✅ **DO:**
- Use declarative (YAML) in production
- Store YAML files in Git
- Use `kubectl apply` (not `kubectl create`)
- Document changes in Git commit messages
- Use meaningful names and labels
- Add resource requests/limits
- Use namespaces for isolation

❌ **DON'T:**
- Mix imperative and declarative for same resources
- Manually edit resources in production
- Skip YAML validation
- Forget to version control YAML files
- Use latest image tag in production

---

## Complete Demo Workflow

### End-to-End Demo: Deploy Web Application

**Scenario:** Deploy a Node.js API with proper DevOps practices.

**Step 1: Create Directory Structure**

```bash
mkdir nodejs-k8s-demo
cd nodejs-k8s-demo
```

**Step 2: Create Deployment YAML**

`deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-web-api
  labels:
    app: web-api
    environment: production
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      containers:
      - name: api
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          cat > app.js << 'EOF'
          const http = require('http');
          const os = require('os');
          const VERSION = process.env.APP_VERSION || 'v1.0.0';
          
          let requestCount = 0;
          
          const server = http.createServer((req, res) => {
            requestCount++;
            
            if (req.url === '/health') {
              res.writeHead(200);
              res.end('OK');
            } else if (req.url === '/metrics') {
              res.writeHead(200, {'Content-Type': 'application/json'});
              res.end(JSON.stringify({
                requests: requestCount,
                uptime: process.uptime(),
                memory: process.memoryUsage()
              }));
            } else {
              res.writeHead(200, {'Content-Type': 'text/html'});
              res.end(`
                <html>
                <head><title>Node.js K8s Demo</title></head>
                <body style="font-family: Arial; margin: 50px;">
                  <h1>🚀 Kubernetes Demo Application</h1>
                  <h2>Version: ${VERSION}</h2>
                  <p><strong>Pod Name:</strong> ${os.hostname()}</p>
                  <p><strong>Node.js Version:</strong> ${process.version}</p>
                  <p><strong>Uptime:</strong> ${Math.floor(process.uptime())} seconds</p>
                  <p><strong>Requests Handled:</strong> ${requestCount}</p>
                  <p><strong>Timestamp:</strong> ${new Date().toISOString()}</p>
                  <hr>
                  <p><a href="/health">Health Check</a> | <a href="/metrics">Metrics</a></p>
                </body>
                </html>
              `);
            }
          });
          
          server.listen(8080);
          console.log(`Server ${VERSION} running on port 8080`);
          EOF
          node app.js
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: APP_VERSION
          value: "v1.0.0"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

**Step 3: Create Service YAML**

`service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-web-api-service
  labels:
    app: web-api
spec:
  type: NodePort
  selector:
    app: web-api
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
```

**Step 4: Deploy Application**

```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Apply service
kubectl apply -f service.yaml

# Watch rollout
kubectl rollout status deployment/nodejs-web-api

# Verify resources
kubectl get deploy,rs,pods,svc -l app=web-api

# Get service details
kubectl get svc nodejs-web-api-service

# Get NodePort
kubectl get svc nodejs-web-api-service -o jsonpath='{.spec.ports[0].nodePort}'
```

**Step 5: Access Application**

```bash
# Port forward
kubectl port-forward svc/nodejs-web-api-service 8080:80

# Access in browser: http://localhost:8080

# OR test with curl multiple times (see different pods respond)
for i in {1..10}; do
  curl http://localhost:8080 | grep "Pod Name"
  echo ""
done
```

**Step 6: Scale Application**

```bash
# Scale to 10 replicas
kubectl scale deployment nodejs-web-api --replicas=10

# Watch scaling
kubectl get pods -w -l app=web-api

# Verify
kubectl get deployment nodejs-web-api
```

**Step 7: Update Application Version**

```bash
# Edit deployment.yaml
# Change: APP_VERSION value: "v1.0.0"
# To:     APP_VERSION value: "v2.0.0"

# Apply update
kubectl apply -f deployment.yaml

# Watch rolling update
kubectl rollout status deployment/nodejs-web-api

# Access app and refresh - you'll see gradual change from v1.0.0 to v2.0.0
```

**Step 8: Test Self-Healing**

```bash
# Delete random pods
kubectl delete pod -l app=web-api --force --grace-period=0

# Watch immediate recreation
kubectl get pods -l app=web-api -w
```

**Step 9: Rollback Update**

```bash
# Simulate bad deployment - update to broken version
kubectl set env deployment/nodejs-web-api APP_VERSION=v3.0.0-broken

# Oh no! Rollback!
kubectl rollout undo deployment/nodejs-web-api

# Verify rollback
kubectl rollout status deployment/nodejs-web-api

# Check version in app
curl http://localhost:8080 | grep Version
```

**Step 10: View Logs**

```bash
# Get logs from all pods
kubectl logs -l app=web-api --tail=20

# Follow logs from specific pod
kubectl logs -f <pod-name>

# Get logs from previous crashed container
kubectl logs <pod-name> --previous
```

**Step 11: Cleanup**

```bash
# Delete all resources
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml

# Verify cleanup
kubectl get all -l app=web-api
```

---

## Mini Assignment

### Assignment: Deploy Your Own Microservice

**Objective:** Apply everything you learned to deploy a complete application.

**Requirements:**

1. **Create a Deployment** for a web application (your choice: nginx, nodejs, python, etc.)
   - Minimum 3 replicas
   - Resource requests and limits defined
   - Health checks (liveness and readiness probes)
   - Rolling update strategy configured

2. **Create a Service** to expose your deployment
   - Use NodePort type
   - Map appropriate ports

3. **Demonstrate:**
   - Deploy the application
   - Scale it to 6 replicas
   - Update to a new version (change image tag or environment variable)
   - Perform a rollback
   - Delete and redeploy from YAML

4. **Document:**
   - Create README.md explaining your application
   - List all kubectl commands used
   - Screenshot terminal output showing:
     - Deployment status
     - Pod distribution across nodes
     - Successful rollout
     - Successful rollback

**Bonus Points:**
- Use meaningful labels and annotations
- Implement resource quotas
- Add multiple containers in pod (sidecar pattern)
- Use ConfigMap for application configuration
- Show pod-to-pod communication

---

### Example Solution Template

`my-app-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
    environment: dev
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: main-app
        image: nginx:1.21  # Replace with your image
        ports:
        - containerPort: 80
        env:
        - name: APP_VERSION
          value: "1.0.0"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

`my-app-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

**Commands to run:**
```bash
# Deploy
kubectl apply -f my-app-deployment.yaml
kubectl apply -f my-app-service.yaml

# Scale
kubectl scale deployment my-app --replicas=6

# Update
kubectl set env deployment/my-app APP_VERSION=2.0.0

# Rollback
kubectl rollout undo deployment/my-app

# Cleanup
kubectl delete -f my-app-deployment.yaml
kubectl delete -f my-app-service.yaml
```

---

## Troubleshooting Scenarios

### Scenario 1: Pods Stuck in Pending State

**Problem:**
```bash
kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# my-app-xxx   0/1     Pending   0          5m
```

**Diagnosis:**
```bash
# Check pod details
kubectl describe pod my-app-xxx

# Look for events like:
# "0/3 nodes are available: 3 Insufficient cpu"
```

**Possible Causes & Solutions:**

**A) Insufficient resources:**
```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Solution 1: Reduce resource requests in deployment
# Edit YAML, reduce requests.cpu and requests.memory

# Solution 2: Add more worker nodes
```

**B) Node selector/affinity mismatch:**
```bash
# Check if pod has nodeSelector
kubectl get pod my-app-xxx -o yaml | grep -A 3 nodeSelector

# Solution: Remove nodeSelector or label appropriate nodes
kubectl label nodes <node-name> <key>=<value>
```

**C) PersistentVolumeClaim not bound:**
```bash
# Check PVCs
kubectl get pvc

# Solution: Create PersistentVolume or use dynamic provisioning
```

---

### Scenario 2: Pods Continuously Restarting (CrashLoopBackOff)

**Problem:**
```bash
kubectl get pods
# NAME         READY   STATUS             RESTARTS   AGE
# my-app-xxx   0/1     CrashLoopBackOff   5          3m
```

**Diagnosis:**
```bash
# Check pod logs
kubectl logs my-app-xxx

# Check previous container logs
kubectl logs my-app-xxx --previous

# Describe pod for events
kubectl describe pod my-app-xxx
```

**Possible Causes & Solutions:**

**A) Application error:**
```bash
# Logs show: "Error: Cannot find module 'express'"

# Solution: Fix application code or image
# Update image with dependencies installed
```

**B) Wrong command/args:**
```bash
# Check container command
kubectl get pod my-app-xxx -o yaml | grep -A 5 command

# Solution: Fix command in deployment YAML
```

**C) Missing environment variables:**
```bash
# Logs show: "Error: DATABASE_URL is not defined"

# Solution: Add environment variable in deployment
env:
- name: DATABASE_URL
  value: "postgresql://..."
```

**D) Failed health checks:**
```bash
# Events show: "Liveness probe failed"

# Solution 1: Fix health check endpoint
# Solution 2: Increase initialDelaySeconds
livenessProbe:
  initialDelaySeconds: 30  # Give app more startup time
```

---

### Scenario 3: Unable to Access Application

**Problem:**
```bash
# Pods are running but can't access via NodePort
kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# my-app-xxx   1/1     Running   0          5m

curl http://<node-ip>:<nodeport>
# Connection refused
```

**Diagnosis:**
```bash
# Check service
kubectl get svc my-app-service
kubectl describe svc my-app-service

# Check endpoints
kubectl get endpoints my-app-service
```

**Possible Causes & Solutions:**

**A) Service selector doesn't match pod labels:**
```bash
# Check service selector
kubectl get svc my-app-service -o yaml | grep -A 2 selector

# Check pod labels
kubectl get pods --show-labels

# Solution: Fix selector in service YAML to match pod labels
```

**B) Wrong targetPort:**
```bash
# Service targetPort: 8080
# But container listens on port: 80

# Solution: Fix targetPort in service YAML
ports:
- port: 80
  targetPort: 80  # Match container port
```

**C) Security group not allowing traffic:**
```bash
# AWS: Check security group allows NodePort range (30000-32767)

# Solution: Add inbound rule for NodePort in AWS security group
```

**D) Pod not ready:**
```bash
# Check readiness
kubectl get pods
# READY shows 0/1

# Check readiness probe
kubectl describe pod my-app-xxx | grep -A 10 Readiness

# Solution: Fix readiness probe or wait for app to be ready
```

---

### Scenario 4: Rolling Update Stuck

**Problem:**
```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas have been updated...
# (Stuck for 10 minutes)
```

**Diagnosis:**
```bash
# Check deployment status
kubectl get deployment my-app

# Check ReplicaSets
kubectl get rs

# Check pod status
kubectl get pods

# Describe deployment
kubectl describe deployment my-app
```

**Possible Causes & Solutions:**

**A) New pods failing readiness probe:**
```bash
# New pods show Running but not Ready
kubectl get pods
# NAME              READY   STATUS    RESTARTS   AGE
# my-app-new-xxx    0/1     Running   0          5m

# Check readiness probe
kubectl logs my-app-new-xxx

# Solution 1: Fix application issue
# Solution 2: Rollback
kubectl rollout undo deployment/my-app
```

**B) Image pull failure:**
```bash
# Pods show ImagePullBackOff
kubectl describe pod my-app-new-xxx
# Events: Failed to pull image "myapp:v2.0.0": image not found

# Solution: Fix image name/tag in deployment
```

**C) Insufficient resources:**
```bash
# New pods stuck in Pending
kubectl describe pod my-app-new-xxx
# Events: 0/3 nodes are available: 3 Insufficient memory

# Solution: Scale down replicas or add more nodes
kubectl scale deployment my-app --replicas=3
```

**D) Rollout paused:**
```bash
# Check if deployment is paused
kubectl get deployment my-app -o yaml | grep paused

# Solution: Resume deployment
kubectl rollout resume deployment/my-app
```

---

### Scenario 5: Deployment Deleted But Pods Still Running

**Problem:**
```bash
# Deleted deployment but pods still exist
kubectl delete deployment my-app
kubectl get pods
# Pods still showing...
```

**Diagnosis:**
```bash
# Check if pods belong to different controller
kubectl describe pod <pod-name> | grep "Controlled By"

# Check all resources
kubectl get all
```

**Possible Causes & Solutions:**

**A) Pods belong to ReplicaSet that wasn't deleted:**
```bash
# Check ReplicaSets
kubectl get rs

# Solution: Delete ReplicaSet
kubectl delete rs <replicaset-name>
```

**B) Pods belong to StatefulSet or DaemonSet:**
```bash
# Check other controllers
kubectl get statefulsets,daemonsets

# Solution: Delete appropriate controller
kubectl delete statefulset <name>
```

**C) Pods have finalizers:**
```bash
# Check for finalizers
kubectl get pod <pod-name> -o yaml | grep finalizers

# Solution: Remove finalizers (advanced)
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":null}}'
```

**D) Different namespace:**
```bash
# Check all namespaces
kubectl get pods --all-namespaces | grep my-app

# Solution: Delete from correct namespace
kubectl delete pod <pod-name> -n <namespace>
```

---

### Troubleshooting Command Reference

```bash
# Get resources with more details
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pods -o yaml

# Describe resources (shows events)
kubectl describe pod <pod-name>
kubectl describe deployment <deploy-name>
kubectl describe node <node-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Multi-container pod
kubectl logs <pod-name> --previous            # Previous crashed container
kubectl logs -f <pod-name>                    # Follow logs
kubectl logs -l app=myapp --tail=50           # Logs from all pods with label

# Execute commands in pod
kubectl exec <pod-name> -- <command>
kubectl exec -it <pod-name> -- /bin/bash     # Interactive shell

# Port forwarding for testing
kubectl port-forward <pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80

# Check resource usage
kubectl top nodes
kubectl top pods

# Check events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n kube-system
kubectl get events --field-selector involvedObject.name=<pod-name>

# Debug networking
kubectl run debug --image=nicolaka/netshoot -it --rm -- /bin/bash
# Inside: ping, curl, nslookup, dig, traceroute

# Check API resources
kubectl api-resources
kubectl explain pod
kubectl explain deployment.spec

# Force delete stuck pod
kubectl delete pod <pod-name> --force --grace-period=0

# Check cluster health
kubectl get componentstatuses
kubectl cluster-info
kubectl get nodes
```

---

## Summary

### Key Concepts Learned

**Pods:**
- ✅ Smallest deployable unit in Kubernetes
- ✅ Can contain one or more containers
- ✅ Ephemeral (temporary) - can be replaced
- ✅ Self-healing at container level
- ✅ Not recommended for direct use in production

**ReplicaSets:**
- ✅ Ensures N identical pods always running
- ✅ Self-healing at pod level
- ✅ Easy scaling (change replica count)
- ✅ Label-based pod selection
- ✅ No built-in update/rollback mechanism

**Deployments:**
- ✅ Manages ReplicaSets automatically
- ✅ Rolling updates (zero-downtime)
- ✅ Easy rollbacks
- ✅ Revision history
- ✅ Production standard for stateless applications

---

### Hierarchy Recap

```
You create → Deployment
             ↓
Deployment creates → ReplicaSet
                     ↓
ReplicaSet creates → Pods
                     ↓
Pods run → Containers
```

**Never create ReplicaSets directly. Always use Deployments!**

---

### When to Use What?

| Use Case                          | Resource to Use |
|-----------------------------------|-----------------|
| Quick testing                     | Pod (imperative)|
| Learning pod concepts             | Pod (YAML)      |
| Production stateless app          | Deployment      |
| Need rolling updates              | Deployment      |
| Need rollback capability          | Deployment      |
| Database (needs persistent identity)| StatefulSet   |
| Monitoring agent on every node    | DaemonSet       |
| Batch/one-time jobs               | Job/CronJob     |

---

### Production Best Practices

1. **Always use Deployments** (not bare Pods or ReplicaSets)
2. **Define resource requests and limits**
3. **Implement health checks** (liveness and readiness probes)
4. **Use declarative YAML** (version control in Git)
5. **Add meaningful labels** and annotations
6. **Set appropriate rolling update strategy**
7. **Test updates in dev/staging first**
8. **Monitor rollout status**
9. **Have rollback plan ready**
10. **Use namespaces** for isolation

---

### Next Steps

After mastering Pods, ReplicaSets, and Deployments, learn:

1. **Services** - Networking and load balancing
2. **ConfigMaps & Secrets** - Configuration management
3. **StatefulSets** - Stateful applications (databases)
4. **DaemonSets** - One pod per node
5. **Jobs & CronJobs** - Batch processing
6. **Ingress** - External HTTP(S) routing
7. **Persistent Volumes** - Storage
8. **RBAC** - Security and access control
9. **Network Policies** - Pod-to-pod security
10. **Helm** - Package manager for Kubernetes

---

## Quick Reference Card

### Essential kubectl Commands

```bash
# Get resources
kubectl get pods
kubectl get deployments
kubectl get rs
kubectl get all

# Detailed info
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash

# Create (imperative)
kubectl run nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3

# Create (declarative)
kubectl apply -f file.yaml

# Update
kubectl set image deployment/nginx nginx=nginx:1.22
kubectl scale deployment nginx --replicas=5
kubectl edit deployment nginx

# Rollout management
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx

# Delete
kubectl delete pod <name>
kubectl delete deployment <name>
kubectl delete -f file.yaml

# Troubleshooting
kubectl get events
kubectl top pods
kubectl port-forward pod/<name> 8080:80
```

---

**Congratulations! 🎉**

You now understand the core building blocks of Kubernetes applications. Practice these concepts with the mini assignment, and you'll be ready for production deployments!

**Remember:** Kubernetes is declarative. You describe what you want, and Kubernetes makes it happen. Trust the system! 🚀

---

## 🧑‍💻 Author

*Md. Sarowar Alam*  
Lead DevOps Engineer, Hogarth Worldwide  
📧 Email: sarowar@hotmail.com  
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/

---
