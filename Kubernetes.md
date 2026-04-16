# Kubernetes Advanced Orchestration: Scaling, Updates, and Configuration Management

This repository serves as a professional engineering guide for managing production-grade Kubernetes workloads. It covers the core pillars of cloud-native infrastructure: self-healing via **ReplicaSets**, dynamic scaling through **HPA/VPA/CA**, zero-downtime **Rolling Updates**, and secure **Configuration Management**.

---

## 🏗️ 1. Workload Redundancy: ReplicaSets

A **ReplicaSet (RS)** ensures that a specified number of pod replicas are running at any given time. It acts as a self-healing mechanism, reconciling the **Actual State** with the **Desired State**.

> [!IMPORTANT]
> **Engineering Best Practice:** Never manage ReplicaSets directly. Always use a **Deployment** object, which manages ReplicaSets under the hood to support declarative updates and rollbacks.

### Key Capabilities:
- **High Availability:** Eliminates single points of failure by maintaining redundant pods.
- **Label Selectors:** RS uses pod labels to acquire and monitor pods, ensuring loose coupling between the controller and the workloads.
- **Fault Tolerance:** Automatically replaces pods that fail or are evicted.

```bash
# Scaling a deployment manually (imperative approach)
kubectl scale deployment/api-gateway --replicas=5

# Checking the status of the ReplicaSet
kubectl get rs

```

----------

## 📈 2. Automated Scaling Strategies

To optimize resource utilization and costs, Kubernetes provides three distinct autoscaling layers.

**Type**

**Level**

**Function**

**HPA** (Horizontal Pod Autoscaler)

**Pod**

Adjusts the _number_ of pod replicas based on CPU/Memory or custom metrics.

**VPA** (Vertical Pod Autoscaler)

**Pod**

Adjusts the _resource limits/requests_ (CPU/RAM) of existing pods.

**CA** (Cluster Autoscaler)

**Node**

Adds or removes _Nodes_ in the cluster when pods cannot be scheduled.

### Implementation (HPA)

Triggering a scale-out when CPU usage exceeds a specific threshold:

Bash

```
kubectl autoscale deployment/web-server --min=2 --max=10 --cpu-percent=70

```

----------

## 🔄 3. Zero-Downtime Rolling Updates

Rolling updates allow for seamless application upgrades by incrementally replacing old pod versions with new ones.

### Pre-requisites for Production Stability:

1.  **Readiness Probes:** Ensure the new pod is actually ready to handle traffic before the old one is terminated.
    
2.  **Liveness Probes:** Ensure the application remains healthy during its lifecycle.
    
3.  **Strategy Configuration:**
    
    -   `maxUnavailable`: Maximum number of pods that can be offline during the update (Set to `0` for strict zero-downtime).
        
    -   `maxSurge`: Maximum number of pods that can be created above the desired replica count.
        

### Deployment Strategy Example (YAML snippet):

YAML

```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0

```

### Rollback Operations:

If a deployment contains a bug, use the undo command to revert to the previous stable revision:

Bash

```
kubectl rollout undo deployment/web-server
kubectl rollout status deployment/web-server

```

----------

## 🔐 4. Externalized Configuration: ConfigMaps & Secrets

Following the **12-Factor App** methodology, configuration must be decoupled from code.

### ConfigMaps (Non-sensitive data)

Used for environment variables, config files, or command-line arguments.

-   **Limit:** 1MB per object.
    
-   **Consumption:** Mounted as Volumes or injected as Environment Variables.
    

### Secrets (Sensitive data)

Used for API keys, passwords, and certificates. Data is **Base64 encoded**.

Bash

```
# Creating a secret for database credentials
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=P@ssw0rd123

```

----------

## 🔗 5. Cloud Service Binding

Service Binding is the architectural pattern used to connect Kubernetes workloads to external backing services (e.g., Managed DBs, AI APIs, Event Buses).

### Workflow:

1.  **Provision:** Create an instance of the external service (e.g., IBM Watson, AWS RDS).
    
2.  **Bind:** Link the service to the cluster, which automatically generates a **Secret**.
    
3.  **Inject:** The secret is mounted into the Pod, allowing the application code to consume credentials via a JSON file or environment variables.
    

JavaScript

```
// Example: Consuming bound credentials in Node.js
const apiKey = process.env.BINDING_APIKEY;
const dbUrl = process.env.BINDING_DATABASE_URL;
```

### 🛠️ Essential Engineering Commands

| Category | Command |
| :--- | :--- |
| **Inspection** | `kubectl get all -o wide` |
| **Deep Dive** | `kubectl describe pod <pod-name>` |
| **Logs** | `kubectl logs -f deployment/<deploy-name>` |
| **Rollout History** | `kubectl rollout history deployment/<deploy-name>` |
| **Config Audit** | `kubectl get cm,secrets` |

---



# Lab: Kubernetes Workload Orchestration – Scaling, Updates, and Config Management


## 📋 Objectives
- Build and push OCI-compliant images to IBM Cloud Container Registry.
- Implement manual scaling via **ReplicaSet** controllers.
- Execute **Rolling Updates** and **Rollbacks** for zero-downtime deployments.
- Decouple application logic from environment configuration using **ConfigMaps**.
- Configure **Horizontal Pod Autoscaler (HPA)** for dynamic traffic handling.

---

## 🛠 Phase 1: Environment Initialization & Image Lifecycle

Before deploying to the cluster, we must prepare the artifacts and the container registry environment.

### 1.1 Project Setup
```bash
# Navigate to project root
cd /home/project

# Clone the lab artifacts if not present
[ ! -d 'CC201' ] && git clone [https://github.com/ibm-developer-skills-network/CC201.git](https://github.com/ibm-developer-skills-network/CC201.git)

# Enter lab directory
cd CC201/labs/3_K8sScaleAndUpdate/

```

### 1.2 Image Engineering (v1)

We define the namespace environment variable and build the initial version of our microservice.

Bash

```
# Export namespace for the session
export MY_NAMESPACE=sn-labs-$USERNAME

# Build and Push v1 to IBM Cloud Container Registry
docker build -t us.icr.io/$MY_NAMESPACE/hello-world:1 . && \
docker push us.icr.io/$MY_NAMESPACE/hello-world:1

```

----------

## 🚀 Phase 2: Deployment and Manual Scaling

We will now manifest the application into the cluster and manage its capacity manually.

### 2.1 Initial Manifest Application

Edit `deployment.yaml` to replace `<my_namespace>` with your actual namespace, then apply:

Bash

```
# Apply the deployment manifest
kubectl apply -f deployment.yaml

# Expose the deployment via a ClusterIP service
kubectl expose deployment/hello-world

# Verify pod status
kubectl get pods

```

### 2.2 Traffic Proxying & Manual Scaling

In a separate terminal, start the proxy to access internal services:

Bash

```
kubectl proxy

```

Back in the primary terminal, scale the workload to 3 replicas:

Bash

```
# Scale-out operation
kubectl scale deployment hello-world --replicas=3

# Verify load balancing across replicas
for i in `seq 10`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy; done

# Scale-in back to 1 replica
kubectl scale deployment hello-world --replicas=1

```

----------

## 🔄 Phase 3: Lifecycle Management (Updates & Rollbacks)

Simulating a production update with zero downtime.

### 3.1 Rolling Update to v2

1.  Modify `app.js` (Update the welcome message).
    
2.  Build and push the new image:
    

Bash

```
docker build -t us.icr.io/$MY_NAMESPACE/hello-world:2 . && \
docker push us.icr.io/$MY_NAMESPACE/hello-world:2

# Update the deployment image (Triggers Rolling Update)
kubectl set image deployment/hello-world hello-world=us.icr.io/$MY_NAMESPACE/hello-world:2

# Monitor rollout status
kubectl rollout status deployment/hello-world

```

### 3.2 Rollback (Undo)

If the v2 deployment is unstable, revert to the previous state:

Bash

```
kubectl rollout undo deployment/hello-world
kubectl get deployments -o wide

```

----------

## 📂 Phase 4: Decoupling Configuration with ConfigMaps

Hardcoding configuration is an anti-pattern. We move environmental variables to a `ConfigMap`.

### 4.1 ConfigMap Integration

Bash

```
# Create the ConfigMap object
kubectl create configmap app-config --from-literal=MESSAGE="This message came from a ConfigMap!"

# Inject environment variables from ConfigMap
# Update deployment-configmap-env-var.yaml with your namespace
kubectl apply -f deployment-configmap-env-var.yaml

```

### 4.2 Runtime Configuration Update

To update a message without rebuilding the image:

Bash

```
# Delete and recreate the ConfigMap with new data
kubectl delete configmap app-config && \
kubectl create configmap app-config --from-literal=MESSAGE="Dynamic update without rebuild!"

# Restart deployment to pick up new environment variables
kubectl rollout restart deployment hello-world

```

----------

## 📈 Phase 5: Dynamic Orchestration (HPA)

Automating capacity management based on real-time CPU utilization.

### 5.1 Define Resource Constraints

Ensure `deployment.yaml` includes CPU requests/limits:

YAML

```
resources:
  limits:
    cpu: 50m
  requests:
    cpu: 20m

```

### 5.2 Implement HPA

Bash

```
# Apply HPA: Trigger at 5% CPU usage, Scale between 1-10 replicas
kubectl autoscale deployment hello-world --cpu-percent=5 --min=1 --max=10

# Monitor HPA status
kubectl get hpa hello-world --watch

```

### 5.3 Load Injection (Stress Testing)

In a new terminal, execute a high-volume request loop:

Bash

```
for i in `seq 100000`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/hello-world/proxy; done

```

_Observe the `REPLICAS` column in the HPA watch terminal as it scales out to meet demand._

----------

## 🧹 Phase 6: Resource Cleanup

Decommissioning lab resources to prevent environment clutter.

Bash

```
kubectl delete deployment hello-world
kubectl delete service hello-world
kubectl delete hpa hello-world
kubectl delete configmap app-config
```


---


# Kubernetes Workload Orchestration: Scaling & Secrets Management


## 🏗 1. Application Infrastructure Lifecycle
In this phase, we establish the OCI-compliant image lifecycle and initialize the primary deployment manifest.

### 1.1 Image Engineering & Registry Management
We use the IBM Cloud Container Registry to host our artifacts.

```bash
# Workspace initialization
cd k8-scaling-and-secrets-mgmt

# Exporting namespace for environment isolation
export MY_NAMESPACE=sn-labs-$USERNAME 

# Building the production-ready image
docker build . -t us.icr.io/$MY_NAMESPACE/myapp:v1 

# Pushing artifact to the centralized registry
docker push us.icr.io/$MY_NAMESPACE/myapp:v1 

```

### 1.2 Resource-Defined Deployment

The deployment utilizes a `RollingUpdate` strategy to ensure zero-downtime during transitions.

**Parameter**

**Configuration**

**Replicas**

1 (Initial)

**Strategy**

RollingUpdate (25% Surge/Unavailable)

**Resources**

Request: 20m CPU / Limit: 50m CPU

Bash

```
# Manifest Application
kubectl apply -f deployment.yaml

# Service Exposure (ClusterIP)
kubectl expose deployment/myapp

```

----------

## 📈 2. Resource Optimization Layer: VPA

The **Vertical Pod Autoscaler (VPA)** is implemented to automatically manage the resource constraints (Requests/Limits) of the microservice based on real-world telemetry.

> [!NOTE]
> 
> VPA is essential for right-sizing workloads to prevent "OOMKilled" errors or resource over-provisioning.

### Configuration Analysis (`vpa.yaml`)

-   **UpdateMode:** `Auto` (Enables automatic pod restarts with optimized resources).
    
-   **Target:** Deployment `myapp`.
    

Bash

```
kubectl apply -f vpa.yaml
kubectl describe vpa myvpa

```

----------

## ↔️ 3. Elasticity Layer: Horizontal Pod Autoscaler (HPA)

To handle traffic volatility, we implement **Horizontal Pod Autoscaling (HPA)**. This ensures high availability during peak loads by adjusting the pod count rather than size.

### HPA Specifications

**Metric**

**Threshold**

**Range**

**Target CPU Utilization**

5%

1 - 10 Replicas

### Load Testing & Verification

We simulate a high-traffic event to trigger the reconciliation loop:

Bash

```
# Terminal 1: Proxy activation
kubectl proxy

# Terminal 2: Synthetic load injection (Stress Test)
for i in `seq 100000`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/myapp/proxy; done

# Terminal 3: Real-time monitoring
kubectl get hpa myhpa --watch

```

----------

## 🔐 4. Security Layer: Secrets Injection

To adhere to the **Twelve-Factor App** methodology, sensitive credentials are decoupled from the image and injected at runtime via Kubernetes Secrets.

### Secret Architecture

-   **Type:** `Opaque` (General purpose)
    
-   **Encoding:** `Base64`
    
-   **Injection Method:** Environment Variables (`envFrom` or `valueFrom`).
    

Bash

```
# Manifest Update
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml

# Integrity Check
kubectl get secret myapp-secret -o yaml

```

----------

## 🏁 Conclusion

By integrating **HPA**, **VPA**, and **Secrets**, this workload satisfies the core requirements of an enterprise-grade service:

1.  **Vertical Scalability:** Efficient resource usage per pod.
    
2.  **Horizontal Scalability:** Metric-based elasticity for high availability.
    
3.  **Security:** Zero hard-coded credentials in the application layer.


---


# Case Study: Transforming Retail with Kubernetes & Containerization

This repository summarizes the strategic transformation of retail digital infrastructure using Kubernetes. It highlights how container orchestration directly addresses critical business hurdles like seasonal traffic spikes and deployment bottlenecks.

---

## 🏗️ 1. Identifying Critical Retail Hurdles
Modern retail requires 24/7 availability and rapid updates. Traditional infrastructure often fails in four key areas:

| Challenge | Impact on Business | Kubernetes Strategy |
| :--- | :--- | :--- |
| **Scalability** | Performance degradation during Black Friday/Holidays. | **HPA/CA** for dynamic resource expansion. |
| **Deployment** | New offers take weeks to go live. | **CI/CD & Rolling Updates** for minutes-to-market. |
| **Resource Waste** | High costs due to static over-provisioning. | **VPA & Bin Packing** for cost optimization. |
| **Disaster Recovery** | System failures lead to massive revenue loss. | **Multi-region clusters & Velero backups.** |

---

## 🚀 2. The Transformative Solution Stack

### A. Microservices & Docker
- **Decoupling:** Breaking monoliths into independent services for focused scaling.
- **Consistency:** Docker ensures the code works exactly the same on a dev's laptop as it does in the production cluster.

### B. Intelligent Orchestration
- **Auto-scaling:** Dynamic adaptation to traffic (Scaling up for peak hours, scaling down to save costs at night).
- **Load Balancing:** Automated traffic distribution across healthy pods via Services.

### C. Advanced Deployment Patterns
- **Blue-Green Deployments:** Running two identical production environments to ensure seamless cutovers.
- **Canary Releases:** Gradually rolling out changes to a small subset of users to mitigate risk.

---

## 🛠️ 3. Essential Engineering Cheat Sheet

### Workload Management
| Command | Purpose |
| :--- | :--- |
| `kubectl scale deployment --replicas=X` | Manual capacity adjustment. |
| `kubectl autoscale deployment --min=X --max=Y` | Implementing HPA (Metric-based scaling). |
| `kubectl set image deployment/myapp v2` | Triggering a Rolling Update. |
| `kubectl rollout undo deployment/myapp` | Immediate rollback to the last stable state. |

### Configuration & Security
| Command | Purpose |
| :--- | :--- |
| `kubectl create configmap <name> --from-literal` | Externalizing non-sensitive environment variables. |
| `kubectl get secret <name> -o yaml` | Inspecting (Base64 encoded) sensitive credentials. |
| `kubectl rollout restart deployment` | Refreshing pods to pick up new ConfigMap/Secret data. |

---

## 📖 4. Glossary for the Modern DevOps Engineer

- **HPA (Horizontal Pod Autoscaler):** Adds more Pods (Scale-out).
- **VPA (Vertical Pod Autoscaler):** Adds more CPU/RAM to existing Pods (Scale-up).
- **CA (Cluster Autoscaler):** Adds more physical/virtual Nodes to the cluster.
- **Service Binding:** The secure process of connecting a K8s workload to external cloud services (e.g., IBM Watson Tone Analyzer) using managed credentials.
- **Persistent Volume (PV):** Storage that survives pod restarts, essential for stateful applications like databases.
