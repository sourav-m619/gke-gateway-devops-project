
# Modern GKE Networking & CI/CD Infrastructure



## 📌 Project Overview
This repository contains the Kubernetes manifests and infrastructure documentation for a multi-tenant, cloud-native environment hosted on Google Kubernetes Engine (GKE). 

The project demonstrates the modern **Kubernetes Gateway API**, provisioning a Regional External Application Load Balancer to intelligently route traffic to isolated environments using a reserved static public IP. The architecture successfully serves highly available web applications (Nginx, Apache HTTPd) alongside a stateful Jenkins CI/CD pipeline, showcasing advanced traffic shaping, path translation, and custom cloud-native health checks.

## 🛠️ Technologies & Tools Used
* **Cloud Provider:** Google Cloud Platform (GCP)
* **Container Orchestration:** Google Kubernetes Engine (GKE) (Provisioned via `gcloud` CLI)
* **Networking:** Kubernetes Gateway API, GCP Regional External Load Balancer
* **CI/CD:** Jenkins (Deployed via Helm)
* **Web Servers:** Nginx, Apache HTTPd (Deployed via `kubectl`)
* **Version Control:** Git & GitHub

---

## 🏗️ Step-by-Step Implementation

### 1. Cluster Provisioning & Gateway API Enablement
The foundation of the project is a GKE cluster with the Gateway API explicitly enabled using the Google Cloud CLI:
```bash
# Example command used to create the cluster with Gateway API enabled
gcloud container clusters create devops-cluster \
    --region=us-central1 \
    --gateway-api=standard
Once the cluster was up, the successful installation of the Gateway controller was verified by checking the available GatewayClasses:

Bash
kubectl get gatewayclass
# Verified the presence of 'gke-l7-regional-external-managed'
2. Static IP Reservation & Gateway Deployment
To ensure consistent access, a static public IP address was reserved in GCP. A Gateway resource was then deployed in the default namespace, referencing this static IP to provision the underlying GCP External Application Load Balancer.

3. Stateless Application Deployment (kubectl)
Nginx and HTTPd were deployed into a dedicated deploy namespace using standard kubectl apply -f commands.
An HTTPRoute was created to bind to the external Gateway, routing traffic to these services based on URL prefixes (/nginx and /httpd).

4. Stateful CI/CD Deployment (Helm)
Jenkins was deployed into its own isolated jenkins namespace. To manage the complex configurations and Persistent Volume Claims (PVCs), Helm was utilized. The Helm values.yaml was customized to configure Jenkins to operate under a specific URL path (--prefix=/jenkins).

🚧 Challenges Faced & Resolutions
During the implementation of this architecture, several complex networking and infrastructure state issues were encountered and resolved.

1. Application 404 Errors via Gateway Routing
Problem: Accessing http://<STATIC_IP>/nginx or /httpd resulted in a 404 Not Found error from the web servers, despite the Gateway correctly routing the traffic to the pods.

Root Cause: The Gateway was passing the exact request path (/nginx) to the backend containers. The Nginx and Apache servers were looking for a directory named nginx in their web root, which did not exist.

Solution: Engineered a Path Translation mechanism. Implemented a URLRewrite filter within the HTTPRoute specification (ReplacePrefixMatch: /). This dynamically stripped the /nginx prefix at the Load Balancer level, ensuring the backend pods received a standard request for the root path (/).

2. GCP Load Balancer "No Healthy Upstream" (503 Error) for Jenkins
Problem: The Jenkins pod was running successfully (2/2 READY), but the external Gateway IP returned a 503 No Healthy Upstream error when attempting to access http://<STATIC_IP>/jenkins.

Root Cause: Because Jenkins was deployed with a custom environment variable (--prefix=/jenkins), the root path (/) returned a 404/403. However, the automated GCP Load Balancer health check defaults to pinging the root path. GCP continuously received error codes, marked the backend service as dead, and refused to route traffic.

Solution: Implemented a GKE HealthCheckPolicy resource attached to the Jenkins service. This explicitly configured the underlying Google Cloud Load Balancer to check the /jenkins/login path instead of /. Once the external infrastructure synchronized with the internal cluster state, traffic flowed successfully.

3. Git Authentication 403 Permission Denied
Problem: Unable to push local IaC repositories to GitHub, resulting in a 403 Permission Denied fatal error.

Root Cause: GitHub deprecated password authentication for terminal Git operations, and the local OS Credential Manager was aggressively caching outdated credentials.

Solution: Generated a Fine-Grained Personal Access Token (PAT) with explicit repository Read/Write scopes. Bypassed the local credential cache by embedding the token directly into the Git remote origin URL, permanently restoring seamless version control access.

📂 Repository Structure
Plaintext
.
├── 1-infrastructure/
│   └── gateway.yaml               # External-HTTP Gateway configuration
├── 2-web-apps/
│   ├── nginx-deployment.yaml      # Nginx Deployment & Service
│   ├── httpd-deployment.yaml      # HTTPd Deployment & Service
│   └── web-apps-route.yaml        # HTTPRoute with URLRewrite filters
├── 3-jenkins/
│   ├── jenkins-values.yaml        # Helm values overriding default paths
│   ├── jenkins-route.yaml         # HTTPRoute for /jenkins traffic
│   └── jenkins-health-check.yaml  # GCP HealthCheckPolicy for LB syncing
└── README.md                      # Architecture and documentation
🚀 Future Enhancements
GitOps Integration: Deploying ArgoCD to automatically sync cluster state with this GitHub repository.
EOF
