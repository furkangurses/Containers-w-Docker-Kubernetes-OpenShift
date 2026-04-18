# 🚀 Enterprise Kubernetes Architecture: Data API

> **Infrastructure-as-Code (IaC) & Deployment Runbook**
> This repository contains the containerization and Kubernetes orchestration configurations for the `fiscal-data-api` microservice. It utilizes a resilient, multi-tiered deployment strategy prioritizing high availability, secure secret management, and zero-downtime rolling updates.

---

## 🏛 Architecture Overview

| Component | Kubernetes Resource | Engineering Justification |
| :--- | :--- | :--- |
| **App Workload** | `Deployment` | Manages stateless REST API replicas with a `RollingUpdate` strategy (`maxSurge: 25%`) to prevent deployment downtime. |
| **Node Agents** | `DaemonSet` | Ensures critical background processes (e.g., log forwarders) run on every cluster node for full observability. |
| **Config Management**| `ConfigMap` | Decouples environment variables (Gateway URLs, Timeouts) from the container image. |
| **Credentials** | `Secret` | Securely injects base64-encoded database authentication credentials. |
| **Persistence** | `PV` & `PVC` | Provisions a 1Gi block storage volume for localized, persistent data caching. |
| **Networking** | `Service` | Exposes the microservice securely across the cluster via `NodePort`. |

---

## 🛠 Prerequisites
Ensure your local environment is authenticated with your cloud provider and the Kubernetes cluster context is correctly set.

```bash
# Export your designated namespace variable
export MY_NAMESPACE=sn-labs-$USERNAME 

```

----------

## 1. Source Code & Containerization

Begin by cloning the base repository. We utilize a multi-layered Dockerfile to optimize cache invalidation during the build process.

Bash

```
git clone [https://github.com/ibm-developer-skills-network/containers-project.git](https://github.com/ibm-developer-skills-network/containers-project.git)
cd containers-project/

```

### `Dockerfile`

Dockerfile

```
# Use an official, stable Node.js runtime as the base image
FROM node:14-alpine 

# Set the working directory to isolate application files
WORKDIR /app

# Copy application assets sequentially to leverage Docker layer caching
COPY main.js .
COPY public/index.html public/index.html
COPY public/style.css public/style.css

# Expose the standard microservice port
EXPOSE 3000

# Define the default execution command
CMD ["node", "main.js"]

```

----------

## 2. CI/CD: Build & Push Registry

Build the immutable container image and push it to the IBM Cloud Container Registry (ICR) for secure distribution.

Bash

```
# 1. Build the image and tag it with the v1 release
docker build . -t us.icr.io/$MY_NAMESPACE/fiscal-data-api:v1 

# 2. Push the tagged image to the remote registry
docker push us.icr.io/$MY_NAMESPACE/fiscal-data-api:v1 

# 3. Verify the image exists in the registry
ibmcloud cr images

```

----------

## 3. Configuration & Security

Hardcoding endpoints and passwords in code is an anti-pattern. We inject these at runtime using K8s native objects.

### ConfigMap (Environment Variables)

Creates environmental parameters for the external API dependencies.

Bash

```
kubectl create configmap fiscal-api-config \
  --from-literal=API_GATEWAY_URL=[https://api.fiscal-autopsy.internal/v1](https://api.fiscal-autopsy.internal/v1) \
  --from-literal=REQ_TIMEOUT_MS=5000

# Verify creation
kubectl get configmap fiscal-api-config

```

### Secrets (Sensitive Data)

Generates an Opaque secret for database authentication. _(Note: In production, use a declarative YAML with Base64 encoding or integrate HashiCorp Vault)._

Bash

```
kubectl create secret generic fiscal-db-credentials \
  --from-literal=DB_ADMIN_USER=fiscal_sysadmin \
  --from-literal=DB_ADMIN_PASS=SecureComplexHash99!

# Verify creation
kubectl get secret fiscal-db-credentials

```

----------

## 4. Storage & Persistence

For stateful requirements (like temporary data processing), we provision persistent storage independent of the Pod lifecycle.

### `volume-and-pvc.yaml`

YAML

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fiscal-data-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /var/data/fiscal-cache
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fiscal-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

----------

## 5. Workload Deployment

### The Main Application (`deployment.yml`)

This deployment manages the main lifecycle, including resource guardrails to prevent CPU starvation on the nodes.

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fiscal-data-api
  labels:
    app: fiscal-data-api
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fiscal-data-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        app: fiscal-data-api
    spec:
      containers:
      - name: node-app
        image: us.icr.io/<your-namespace>/fiscal-data-api:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        resources:
          limits:
            cpu: 50m
            memory: 128Mi
          requests:
            cpu: 20m
            memory: 64Mi

```

### Node-Level Agents (`daemonset.yaml`)

Ensures high-priority systemic pods run on every node (including the master node, via tolerations).

YAML

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fiscal-node-agent
  labels:
    app: fiscal-node-agent
spec:
  selector:
    matchLabels:
      app: fiscal-node-agent
  template:
    metadata:
      labels:
        app: fiscal-node-agent
    spec:
      containers:
      - name: agent-container
        image: us.icr.io/<your-namespace>/fiscal-data-api:v1
        ports:
        - containerPort: 3000
          name: http
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule

```

**Execution:**

Bash

```
kubectl apply -f deployment.yml
kubectl apply -f daemonset.yaml
kubectl get pods -o wide
kubectl get daemonsets

```

----------

## 6. Network Routing

To expose the application to other cluster services or external ingress controllers, we define a Service.

### `service.yaml`

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: fiscal-api-service
  labels:
    app: fiscal-data-api
spec:
  type: NodePort
  selector:
    app: fiscal-data-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000

```

**Execution & Testing:**

Bash

```
kubectl apply -f service.yaml
kubectl get services

# Forward the port locally for testing (Press CTRL+C to terminate)
kubectl port-forward deployment.apps/fiscal-data-api 3000:3000 

```

----------

## ✅ Pre-Flight Checklist

Before signing off on the environment deployment, ensure the following constraints are met:

-   [ ] Docker image built and verified in IBM Cloud Registry.
    
-   [ ] `ConfigMap` and `Secret` injected correctly without exposing plaintext credentials in logs.
    
-   [ ] PV and PVC bound successfully (`Status: Bound`).
    
-   [ ] Application pods are running with `1/1` readiness.
    
-   [ ] `DaemonSet` pods are deployed equal to the number of active cluster nodes.
    
-   [ ] API responds `HTTP 200 OK` via local port-forwarding.

---


# 🚀 Enterprise Container Orchestration: Scalable Guestbook Microservice

> **Infrastructure-as-Code (IaC) & Release Management Runbook**
> This repository demonstrates a production-grade container orchestration strategy for a Node.js-based Customer Interaction (Guestbook) microservice. It highlights end-to-end container lifecycle management, including image registry distribution, dynamic scaling under load, zero-downtime deployments, and automated rollback mechanisms.

---

## 🏛 Architecture & Tech Stack

This project implements modern DevOps principles to ensure high availability and efficient resource utilization.

| Component | Technology / Resource | Engineering Justification |
| :--- | :--- | :--- |
| **Application Runtime** | `Node.js` / `Docker` | Lightweight, isolated application packaging using multi-stage builds. |
| **Artifact Registry** | `IBM Cloud Container Registry` | Secure, remote storage for versioned immutable image artifacts. |
| **Workload Management** | `Deployment` | Manages replica sets with a `RollingUpdate` strategy to ensure zero-downtime upgrades. |
| **Auto-Scaling** | `HorizontalPodAutoscaler (HPA)` | Dynamically scales pod replicas (1 to 10) based on CPU utilization thresholds. |
| **Network Exposure** | `Port-Forwarding` / `Service` | Securely tunnels traffic for local environment validation. |

---

## 🛠 Prerequisites

Ensure your cluster context is correctly configured and you are authenticated with your cloud provider's image registry.

```bash
# Define target namespace for deployment isolation
export MY_NAMESPACE=sn-labs-$USERNAME

# Clone the base infrastructure repository
git clone [https://github.com/ibm-developer-skills-network/guestbook.git](https://github.com/ibm-developer-skills-network/guestbook.git)
cd guestbook/v1/guestbook

```

----------

## Phase 1: Container Build & Distribution

We treat container images as immutable artifacts. The application is packaged and pushed to the remote registry for cluster-wide accessibility.

### 1. Build the Release Candidate

Bash

```
# Build the Docker image utilizing the local Dockerfile
docker build . -t us.icr.io/$MY_NAMESPACE/guestbook:v1

```

### 2. Push to Container Registry

Bash

```
# Push the versioned artifact to the secure registry
docker push us.icr.io/$MY_NAMESPACE/guestbook:v1

# Validate artifact presence in the repository
ibmcloud cr images

```

----------

## Phase 2: Kubernetes Orchestration

Deploy the application using declarative YAML manifests to establish the baseline infrastructure state.

### `deployment.yml` (Baseline Configuration)

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: us.icr.io/<your-namespace>/guestbook:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        resources:
          requests:
            cpu: 20m
          limits:
            cpu: 50m

```

### Apply and Verify

Bash

```
# Provision the deployment state
kubectl apply -f deployment.yml

# Establish a secure tunnel to verify the application payload
kubectl port-forward deployment.apps/guestbook 3000:3000

```

----------

## Phase 3: Elasticity & Load Testing (HPA)

To ensure the microservice can handle traffic spikes without degradation, we configure a Horizontal Pod Autoscaler (HPA) and simulate a distributed denial-of-service (DDoS) / high-traffic event.

### 1. Configure the Autoscaler

Bash

```
# Define scaling thresholds: target 50% CPU, scale between 1 and 10 pods
kubectl autoscale deployment guestbook --cpu-percent=50 --min=1 --max=10

# Verify HPA initialization
kubectl get hpa guestbook

```

### 2. Traffic Simulation

In a separate terminal, deploy a temporary `busybox` container to generate aggressive traffic against the application endpoint.

Bash

```
# Execute a continuous HTTP GET request loop
kubectl run -i --tty load-generator --rm --image=busybox:1.36.0 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- <your-app-URL>; done"

```

### 3. Monitor Scaling Telemetry

Watch the cluster react to the CPU pressure by dynamically provisioning new replicas.

Bash

```
# Observe the replicas scaling up (e.g., from 1 to 5+)
kubectl get hpa guestbook --watch

```

----------

## Phase 4: Release Management & Rollbacks

Modern CI/CD requires the ability to push updates seamlessly and revert immediately if anomalous behavior is detected.

### 1. Deploying Version 2 (Rolling Update)

After updating the frontend code (`index.html` to v2) and optimizing CPU resource allocations in the manifest (Requests: `2m`, Limits: `5m`), we push the new release.

Bash

```
# Build, tag, and push the v2 artifact
docker build . -t us.icr.io/$MY_NAMESPACE/guestbook:v1 && \
docker push us.icr.io/$MY_NAMESPACE/guestbook:v1

# Apply the mutated state to trigger a zero-downtime rolling update
kubectl apply -f deployment.yml

```

### 2. Audit & Rollout History

Verify the deployment history to ensure proper version tracking.

Bash

```
# Review the cluster's deployment ledger
kubectl rollout history deployment/guestbook

# Inspect the granular details of Revision 2
kubectl rollout history deployments guestbook --revision=2

```

### 3. Incident Response: Automated Rollback

If `v2` introduces a critical bug or fails health checks, we execute an immediate rollback to the last known stable state (Revision 1).

Bash

```
# Check current ReplicaSets before rollback
kubectl get rs

# Revert cluster state to Revision 1
kubectl rollout undo deployment/guestbook --to-revision=1

# Validate the graceful termination of v2 pods and restoration of v1
kubectl get rs

```

----------

## ✅ Deployment Sign-Off Checklist

-   [ ] Base Docker image successfully built and pushed to the internal registry.
    
-   [ ] Kubernetes `Deployment` applied and responding on Port 3000.
    
-   [ ] `HorizontalPodAutoscaler` validated under simulated load (Replicas > 1).
    
-   [ ] Application successfully updated to `v2` via Rolling Update.
    
-   [ ] Rollback procedure executed and validated (`Revision 1` restored).

---


# 🛠️ Containerization & Orchestration

> **Reference guide for Docker, Kubernetes (K8s), and OpenShift CLI operations.**
> This document serves as a high-performance quick-reference for managing container lifecycles, cluster resources, and automated scaling.

---

## 🐳 Docker CLI & Cloud Registry
*Essential commands for building, tagging, and distributing container images.*

| Command | Description |
| :--- | :--- |
| `docker build . -t <name>:<tag>` | Builds an image from the local Dockerfile and assigns a name/tag. |
| `docker images` | Lists all locally available Docker images. |
| `docker run -p <host>:<cont>` | Executes a container with specific port mapping. |
| `docker ps` | Lists active containers (`-a` for all, including stopped). |
| `docker stop $(docker ps -q)` | Force-stops all currently running containers. |
| `docker tag <src> <target>` | References a source image with a new target tag for registry pushes. |
| `docker push <image>` | Uploads a versioned image to a remote registry. |
| `ibmcloud cr images` | Audits available images within the IBM Cloud Container Registry. |
| `ibmcloud cr login` | Authenticates the local Docker daemon to the cloud registry. |
| `git clone <url>` | Retrieves infrastructure-as-code (IaC) artifacts from remote repos. |

---

## ☸️ Kubernetes: Architecture & Resource Management
*Core commands for deploying workloads and inspecting cluster state.*

| Command | Description |
| :--- | :--- |
| `kubectl apply -f <file>.yml` | Declaratively creates/updates resources defined in a YAML manifest. |
| `kubectl get pods -o wide` | Lists pods with extended information (IPs, Node assignment). |
| `kubectl describe <type> <name>` | Provides deep-dive metadata and event logs for a specific resource. |
| `kubectl get deployments` | Audits all managed application workloads in the current namespace. |
| `kubectl get services` | Lists internal and external networking endpoints (ClusterIP, NodePort). |
| `kubectl expose <res> --type=...` | Dynamically creates a service to route traffic to a resource. |
| `kubectl config get-contexts` | Identifies the active cluster and user credentials being used. |
| `kubectl proxy` | Bridges the local machine to the K8s API server for secure access. |

---

## 📈 Kubernetes: Scaling & Release Operations
*Advanced operations for managing application elasticity and lifecycle updates.*

| Command | Description |
| :--- | :--- |
| `kubectl autoscale deployment ...` | Attaches a Horizontal Pod Autoscaler (HPA) to a deployment. |
| `kubectl get hpa` | Monitors the current vs. target CPU utilization and replica count. |
| `kubectl scale deployment --replicas=N` | Manually overrides the number of running pod instances. |
| `kubectl create configmap <name>` | Injects non-sensitive configuration data into containers at runtime. |
| `kubectl create secret generic <name>` | Securely injects sensitive credentials (passwords, keys) into pods. |
| `kubectl rollout history <res>` | Audits the revision history for deployments (used for version tracking). |
| `kubectl rollout undo <res>` | Executes an immediate rollback to the previous stable revision. |
| `kubectl rollout restart <res>` | Triggers a fresh pull and restart of all pods in a deployment. |

---

## 🔴 OpenShift CLI (oc)
*Platform-specific commands for the Red Hat OpenShift Container Platform.*

| Command | Description |
| :--- | :--- |
| `oc get <resource>` | Standard retrieval command (similar to `kubectl get`). |
| `oc project <name>` | Context-switch between different isolated namespaces/projects. |
| `oc version` | Displays client and server-side OpenShift environment details. |
| `oc tag <src> <dest>` | Manages internal Image Streams for automated deployment triggers. |

---

> **Tip:** Use `kubectl <command> --help` for detailed flag documentation on any specific operation.

---
