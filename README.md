# 🚀 Cloud-Native Jenkins on GKE using Gateway API

## 📌 Overview

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/60f911c3-8644-4028-a10e-556921f231f9" />


This project demonstrates a production-style deployment of stateless and stateful applications on Google Kubernetes Engine (GKE), exposed externally using the Kubernetes Gateway API with a reserved static public IP.

The implementation includes:

* GKE cluster provisioned with Gateway API enabled
* External HTTP Gateway backed by a GCP Application Load Balancer
* Stateless workloads (Nginx & HTTPd) deployed via kubectl
* Jenkins deployed via Helm with persistent storage
* Advanced routing with URL rewrite filters
* Custom GKE HealthCheckPolicy for load balancer synchronization

---

# 🏗 Architecture Summary

Client → Static Public IP → GKE Gateway → HTTPRoute → Kubernetes Services → Pods

Workloads are logically separated:

* `deploy` namespace → Stateless web apps
* `jenkins` namespace → Stateful CI/CD server
* `default` namespace → Gateway infrastructure

---

# 1️⃣ GKE Cluster Creation (Gateway API Enabled)

Cluster was created with Gateway API enabled at provisioning time:

```bash
gcloud container clusters create devops-cluster \
    --region=us-central1 \
    --gateway-api=standard
```

Once the cluster was ready:

```bash
kubectl get gatewayclass
```

Verified presence of:

```
gke-l7-regional-external-managed
```

This confirms the managed Gateway controller is installed and ready.

---

# 2️⃣ Static IP Reservation & External Gateway Deployment

To ensure consistent public access, a static global public IP was reserved:

```bash
gcloud compute addresses create devops-static-ip --global
```

A Gateway resource was then deployed referencing this static IP, which automatically provisioned a Google Cloud External Application Load Balancer.

The Gateway listens on HTTP (port 80) and serves as the centralized traffic entry point.

---

# 3️⃣ Stateless Applications Deployment (kubectl)

Created a dedicated namespace:

```bash
kubectl create namespace deploy
```

Deployed:

* Nginx
* Apache HTTPd

Using standard:

```bash
kubectl apply -f <manifest>.yaml
```

An HTTPRoute was created to bind to the external Gateway with path-based routing:

* `/nginx` → Nginx service
* `/httpd` → HTTPd service

---

# 4️⃣ Stateful Jenkins Deployment (Helm)

Jenkins was deployed in an isolated namespace:

```bash
kubectl create namespace jenkins
```

Helm was used due to:

* Complex configuration
* Persistent Volume Claims (PVC)
* Service customization
* Plugin management

Installation:

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm install my-jenkins jenkins/jenkins \
  -n jenkins \
  -f jenkins-values.yaml
```

Jenkins was configured to run under a path prefix:

```
--prefix=/jenkins
```

An HTTPRoute was created to expose Jenkins at:

```
http://<STATIC_IP>/jenkins
```

---

# 🚧 Challenges Faced & Resolutions

## 1️⃣ 404 Errors for Nginx & HTTPd via Gateway

### Problem

Accessing:

```
http://<STATIC_IP>/nginx
```

Returned 404 errors.

### Root Cause

The Gateway forwarded the full request path (`/nginx`) to the backend. The containers attempted to locate a directory named `nginx` inside their web root, which did not exist.

### Solution

Implemented URL rewriting using an HTTPRoute filter:

```yaml
filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplacePrefixMatch
      replacePrefixMatch: /
```

This stripped `/nginx` before forwarding traffic, allowing backend containers to serve from root (`/`).

---

## 2️⃣ 503 "No Healthy Upstream" for Jenkins

### Problem

Jenkins pod was healthy (2/2 READY), but accessing:

```
http://<STATIC_IP>/jenkins
```

Returned:

```
503 No Healthy Upstream
```

### Root Cause

Because Jenkins was configured with `--prefix=/jenkins`, the root path (`/`) returned 404/403.

However, GCP Load Balancer health checks default to probing `/`.

Since `/` failed, the backend was marked unhealthy.

### Solution

Created a GKE HealthCheckPolicy attached to the Jenkins service:

```yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
```

Configured health check path to:

```
/jenkins/login
```

Once the health check matched Jenkins’ actual serving path, the load balancer marked the backend healthy and traffic began flowing successfully.

---

---

# 📂 Repository Structure

```
.
├── 1-infrastructure/
│   └── gateway.yaml
├── 2-web-apps/
│   ├── nginx-deployment.yaml
│   ├── httpd-deployment.yaml
│   └── web-apps-route.yaml
├── 3-jenkins/
│   ├── jenkins-values.yaml
│   ├── jenkins-route.yaml
│   └── jenkins-health-check.yaml
└── README.md
```

---

# 🔍 Key Learnings

* Gateway API provides cleaner, more flexible routing compared to Ingress
* Path-based routing requires backend awareness or URL rewriting
* Cloud load balancer health checks must align with application serving paths
* Stateful workloads require special handling when using path prefixes
* Infrastructure state synchronization between Kubernetes and GCP can introduce transient errors

---

# 🎯 Final Outcome

✔ Fully functional External Load Balancer via Gateway API
✔ Static public IP mapped to Gateway
✔ Stateless web apps exposed with prefix-based routing
✔ Jenkins deployed with Helm and persistent storage
✔ Health check synchronization resolved
✔ End-to-end CI/CD ready architecture

---

# 🚀 Future Enhancements

* GitOps integration using ArgoCD
* HTTPS enablement with managed certificates
* Domain mapping with Cloud DNS
* Horizontal Pod Autoscaling (HPA)
* CI pipeline to auto-deploy workloads

---

## 🏁 Conclusion

This implementation showcases a modern, cloud-native approach to deploying both stateless and stateful applications on GKE using the Gateway API and Helm. It demonstrates deep understanding of Kubernetes networking, cloud load balancer behavior, and production-grade CI/CD deployment patterns.
