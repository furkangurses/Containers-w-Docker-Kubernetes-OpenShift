# 🚀 IBM: Containers, Docker, Kubernetes & OpenShift

This repository contains projects, hands-on lab exercises, and technical documentation developed during the **"Containers, Docker, Kubernetes & OpenShift"** course provided by IBM. This training covers the essential pillars of Cloud-Native application development and modern DevOps practices.


---

## 🛠️ Tech Stack & Tools
* **Containerization:** Docker, Dockerfile, Docker Hub, Podman
* **Orchestration:** Kubernetes (K8s), Red Hat OpenShift
* **Cloud Architecture:** IBM Cloud, Service Mesh (Istio)
* **Automation & Configuration:** YAML, kubectl, oc CLI
* **Application Management:** Autoscaling, Rolling Updates, ConfigMaps, Secrets

---

## 📚 Course Curriculum & Key Takeaways

### 📦 Module 1: Containers and Docker Foundations
* **Containerization Logic:** Isolating applications with all their dependencies.
* **Docker Architecture:** Deep dive into Docker Daemon (dockered), Client-Server architecture, and Registry management.
* **Dockerfile Mastery:** Optimizing images using critical instructions: `FROM`, `RUN`, `COPY`, `CMD`, `ENV`, `EXPOSE`, and `USER`.
* **Image Management:** Understanding Layered Architecture (Caching), the build process, and pushing images to registries (Docker Hub).
* **Persistence & Networking:** Implementing Volume structures for data persistence and managing container-to-container communication.

### ☸️ Module 2: Kubernetes (K8s) Essentials
* **The Need for Orchestration:** Automating the management of containers at scale.
* **K8s Architecture:** Mastering Control Plane (Master) and Worker Node components.
* **K8s Objects:** Managing Pods, ReplicaSets, and Deployments.
* **kubectl CLI:** Operational commands and declarative management using `YAML` manifests.

### 📈 Module 3: Application Management & Scaling
* **Scaling:** Dynamically adjusting application replicas based on traffic demand (Autoscaling).
* **Rolling Updates & Rollbacks:** Deploying updates without downtime and performing near-instant reverts in case of failure.
* **Configuration Management:** Decoupling sensitive data (`Secrets`) and environment variables (`ConfigMaps`) from the source code.

### 🛡️ Module 4: Enterprise Ecosystem (OpenShift & Istio)
* **Red Hat OpenShift:** Leveraging enterprise-grade Kubernetes, BuildConfigs, and Operators.
* **Istio Service Mesh:** Managing microservices traffic, security, and observability.

### 🏁 Module 5: Capstone Project
* **Guestbook Application:** A full-cycle project involving containerizing a guestbook app with Docker and deploying/managing it on an OpenShift cluster using Kubernetes best practices.

---

## 🚀 Applied Engineering Best Practices
Key professional standards implemented throughout this repository:
* **Security:** Running containers with non-root privileges (`USER` instruction) to minimize attack surfaces.
* **Performance:** Utilizing Layer Caching in Dockerfiles for accelerated build pipelines.
* **Resilience:** Implementing `HEALTHCHECK` mechanisms for self-healing system architectures.
* **Infrastructure as Code (IaC):** Defining the entire system state declaratively through YAML manifests.


---

*This repository represents my technical growth and commitment to mastering cloud-native engineering.*
