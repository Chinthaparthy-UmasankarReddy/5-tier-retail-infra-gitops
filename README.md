
# 🚢 Retail Microservices Platform — GitOps Cluster Engine

This repository serves as the single source of truth for the continuous deployment, orchestration, and zero-trust security configuration of the Retail Microservices Platform on Amazon EKS. Managed entirely via **ArgoCD**, this repository utilizes declarative Kubernetes manifests split into deterministic sequencing layers (Sync Waves) to achieve programmatic, zero-intervention infrastructure hydration.

---

## 🗺️ Repository Topology & Structure

The repository is organized following a strict hierarchical pattern designed for the ArgoCD `ApplicationSet` Git Directory Generator:

```text
5-retail-app-gitops/
├── argocd-root-applicationset.yaml  # Master ApplicationSet Automation Controller
├── argocd-retail-project.yaml       # AppProject Multi-Tenant RBAC Sandbox Boundary
└── kubernetes/
    └── apps/
        ├── 00-core-infra/           # Sync Wave -3 to -2: Shared foundational layer
        │   ├── 01-namespace-onboarding.yaml  # Limits, Quotas, and Namespace perimeter
        │   ├── 02-security-identities.yaml    # Least-privilege ServiceAccounts & IRSA
        │   ├── 03-external-secrets-vault.yaml# Dynamic AWS Secrets Manager Sync via ESO
        │   └── 04-network-isolation.yaml     # Zero-Trust Pod-to-Pod NetworkPolicies
        ├── 01-database/             # Sync Wave -1 to 0: Stateful Datastore & Migrations
        │   ├── db-statefulset.yaml           # Resilient PostgreSQL State Configuration
        │   └── db-seeder-job.yaml            # PostSync Relational Schema Seeder Engine
        ├── login-service/           # Sync Wave 1: Token Authentication Layer (NodeJS)
        │   └── deployment.yaml
        ├── catalog-service/         # Sync Wave 1: Inventory Stock Engine (Go)
        │   └── deployment.yaml
        ├── cards-service/           # Sync Wave 1: Encryption Tokenizer (NodeJS/TS)
        │   └── deployment.yaml
        ├── orders-service/          # Sync Wave 2: Transactional Core Processing (Java)
        │   └── deployment.yaml
        ├── payment-service/         # Sync Wave 2: Financial Clearing Router (Go)
        │   └── deployment.yaml
        └── frontend-service/        # Sync Wave 3: Ingress Web Proxy Entrypoint (Nginx)
            ├── deployment.yaml
            └── ingress.yaml

```

---

## 🌊 Pipeline Orchestration: Sync Waves & Lifecycle Hooks

To guarantee zero runtime connection crashes, components cascade online following a strictly ordered dependency matrix. ArgoCD blocks progress to the next wave until all pods in the active wave pass their `ReadinessProbes`.

```text
[Wave -3] Core Infra -> [Wave -2] Secrets Hydration -> [Wave -1] DB Pod -> [Wave 0] Schema Seed -> [Wave 1] Core APIs -> [Wave 2] Checkout APIs -> [Wave 3] Web Ingress

```

### Wave Lifecycle Matrix

1. **Wave `-3` (Core Infrastructure Base):** Provisions the `retail-prod` namespace, locks down compute constraints via `ResourceQuotas`/`LimitRanges`, maps out `Roles`/`RoleBindings`, and applies pod isolation `NetworkPolicies`.
2. **Wave `-2` (Secrets Hydration):** Launches the `SecretStore` and `ExternalSecret` configurations to extract credentials dynamically out of AWS Secrets Manager.
3. **Wave `-1` (Persistent Datastores):** Spins up the PostgreSQL `StatefulSet` and exposes its stable internal DNS endpoint via a ClusterIP service.
4. **Wave `0` (Database Schema Migration Hook):** Executes a finite Kubernetes `Job` that injects the relational schema (`inventory_ledger` table) and seed records into PostgreSQL.
5. **Wave `1` (Stateless Core Foundations):** Boots the `catalog-service`, `login-service`, and `cards-service` dependencies.
6. **Wave `2` (Upstream Core Consumers):** Brings the Java `orders-service` and Go `payment-service` online safely once their downstream components are fully active.
7. **Wave `3` (Presentation Layer Gateway):** Launches the Nginx static frontend UI and maps out the Application Load Balancer (ALB) via the `Ingress` controller to accept public internet web traffic.

---

## 🔒 Security Architecture Highlights

* **Multi-Tenant AppProject Isolation (`argocd-retail-project.yaml`):** Restricts the repository strictly to the `retail-prod` namespace and enforces explicit whitelists on allowed Kubernetes API primitives.
* **AWS IAM Roles for Service Accounts (IRSA):** Application pods assume scoped AWS IAM identities natively using OpenID Connect (OIDC) annotations on `retail-app-sa`, eliminating the risk of hardcoded master keys.
* **Dynamic Secret Extraction (External Secrets Operator):** Plaintext strings never touch Git. The operator intercepts the pipeline, authenticates with AWS Secrets Manager, and generates native, memory-cached Kubernetes secrets automatically.
* **Micro-Segmented Firewalls (`NetworkPolicies`):** Implements a default-deny ingress/egress posture. The frontend is restricted to talking only to microservice backend layers, while the PostgreSQL database blocks all connections except those originating from the `catalog-service` and `orders-service` pod selectors.

---

## 🚀 Cluster Bootstrap & Deployment Manual

### 1. Pre-requisites & EKS Readiness

Ensure your target EKS cluster has the following operational controller add-ons pre-installed:

* **ArgoCD Operator** (v2.10+)
* **AWS Load Balancer Controller** (For managing Ingress-to-ALB provisioning)
* **External Secrets Operator** (ESO)
* **Amazon VPC CNI** with Network Policy support enabled (`ENABLE_NETWORK_POLICY=true`)

### 2. Establish the Security & Project Boundary

Before bootstrapping the automation engine, provision the restricted project perimeter:

```bash
kubectl apply -f argocd-retail-project.yaml

```

### 3. Initialize the Master GitOps Controller Loop

Apply the master `ApplicationSet` resource. This manifest automatically tracks this GitHub repository, discovers all folders nested within `kubernetes/apps/*`, instantiates individual application nodes inside ArgoCD, and cascades the deployment pipeline:

```bash
kubectl apply -f argocd-root-applicationset.yaml

```

---

## 🧪 Post-Deployment Observability & Validation Matrix

Once synchronized, verify the health status of the ecosystem directly from your cluster management CLI terminal context:

### Check Global Synchronization States

```bash
# Verify all applications track as Healthy and Synced
argocd app list

# Audit individual pod workload health inside the production namespace
kubectl get pods -n retail-prod

```

### Test Database Network Firewalls

Validate that your network segmentation policies successfully drop illegal connection requests:

```bash
# Exec into the public frontend pod and attempt to connect directly to the database port
kubectl exec -it deployment/frontend-service -n retail-prod -- nc -zv retail-db-service 5432

# EXPECTED OUTPUT: Connection Timeout / Denied (Blocked by NetworkPolicy)

```

### Stream Live System Logging Telemetry

```bash
# View live structured log files for tracking database seeder jobs or checkouts
kubectl logs -f deployment/catalog-service -n retail-prod
kubectl logs -f deployment/orders-service -n retail-prod

```

---

## 🔄 GitOps Application Management Operations

To push an application update, structural environment configurations change, or image tag version upgrades (e.g., migrating from `v21` to `v22` of the frontend):

1. Modify the declarative target properties within the relevant microservice subfolder path.
2. Commit and push the manifest changes to the tracking branch (`main`).
3. ArgoCD intercepts the push event hook, recalculates the desired state delta, updates the out-of-sync parameters, and executes a zero-downtime rolling update deployment seamlessly!


Beyond core manifests, security parameters, and GitOps workflows, true day-2 operations focus heavily on **Advanced GitOps Patterns, Observability, Cost Optimization, Infrastructure Hardening, and High-Availability Strategy**.

---

## 🏗️ 1. Advanced GitOps Patterns (The App-of-Apps Setup)

While an `ApplicationSet` is powerful for folder-based auto-discovery, enterprise architectures often leverage the **App-of-Apps Pattern** for cluster bootstrapping. This pattern creates a parent ArgoCD Application that points to a folder containing *other* Application manifests, decoupling infrastructure blueprints from the application code.

### 🗂️ Multi-Cluster Multi-Environment Directory Topology

In a real production environment, you don't run Dev, Stage, and Prod in the same repository under a single flat folder. Instead, you separate them into a multi-layer repository layout using tools like **Kustomize** or **Helm** to manage overlays without duplicating raw YAML code:

```text
5-retail-app-gitops/
├── bootstrap/
│   ├── root-app-of-apps.yaml       # Master App-of-Apps definition
│   └── infrastructure-apps/        # Definitions for core tools (ALB, ESO, Prometheus)
└── kubernetes/
    ├── base/                        # Raw baseline manifests (Deployments, Services)
    │   ├── catalog-service/
    │   └── orders-service/
    └── environments/
        ├── dev/                     # Kustomize or Helm Overlays for Dev
        │   ├── kustomization.yaml   # Updates replica counts to 1, uses SPOT node selectors
        │   └── patches/
        └── prod/                    # Kustomize or Helm Overlays for Prod
            ├── kustomization.yaml   # Updates replica counts to 4, uses ON_DEMAND selectors
            └── patches/

```

---

## 📊 2. High-Fidelity Observability Matrix

A production EKS cluster is blind without structured metrics and trace collection. Your polyglot stack (Go, Node.js, Java) must be monitored continuously using the **LGTM Stack (Loki, Grafana, Tempo, Mimir)** or **Prometheus & Grafana**.

### 📈 Core Telemetry Implementation

* **Prometheus ServiceMonitors:** Instead of manual scraping configurations, define a Kubernetes Custom Resource Definition (CRD) matching your microservices to scrape target endpoint metrics automatically.
* **Log Aggregation via Vector or FluentBit:** Shard log generation files away from the local node storage disks. Stream stdout tracking arrays directly into an **Amazon CloudWatch Log Group** or a self-hosted **Grafana Loki** grid.
* **Distributed Tracing (OpenTelemetry + Jaeger/Tempo):** Because checkouts jump from the Frontend ➔ Orders Service ➔ Cards Service ➔ Payment Service, you must inject **W3C Trace Context Headers** into your application network calls to track latency bottlenecks across container networks seamlessly.

---

## 💰 3. Financial Engineering & Cost Optimization

EKS cluster compute spend can grow rapidly if left unchecked. Implementing automated cost constraints ensures your cloud efficiency remains highly optimized:

* **EKS Pod Sandboxing via Kubecost:** Deploy **Kubecost** alongside your ArgoCD core infrastructure wave. Kubecost aggregates real-time AWS infrastructure metrics and charts exactly how much monthly budget each microservice (e.g., `catalog-service` vs `payment-service`) consumes in terms of compute allocation.
* **Dev Environment Downscaling (KEDA / Cron):** Lower environment instances (Dev/Stage) do not need to run overnight or during weekends. Implement **KEDA (Kubernetes Event-driven Autoscaling)** or a native CronJob infrastructure script to downscale your deployment replica constraints to `0` at 8:00 PM and scale them back up to `2` at 8:00 AM automatically, cutting compute costs significantly.

---

## 🛡️ 4. EKS Infrastructure Hardening

To pass modern enterprise security audits, your EKS cluster must enforce several foundational security constraints:

### Pod Security Standards (PSS)

Enforce the native Kubernetes `restricted` pod security profile at the namespace label boundary to prevent container execution escape flaws:

```yaml
# Add these strict boundary control annotations to your 01-namespace-onboarding.yaml file
metadata:
  name: retail-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

```

*This instantly blocks pods from running as the root user, prevents privilege escalation, and restricts access to the underlying host network namespaces.*

### KMS Secret Encryption

By default, Kubernetes Secrets are stored in plaintext (Base64 encoded) within `etcd`. Ensure your Terraform EKS module defines an **AWS KMS Customer Managed Key (CMK)** to encrypt the cluster's `etcd` secret volume layer natively.

---

## 🌀 5. High-Availability & Disaster Recovery (DR)

Running your nodes across multiple Availability Zones (AZs) protects against a single data center outage, but enterprise resilience requires planning for larger disasters:

* **Velero Cluster Backup Management:** Install the **Velero Operator** within your cluster utilities plane. Velero runs automated cron backup loops that snapshot your cluster's entire state (all ConfigMaps, Custom Resource Definitions, and persistent volumes) and pushes them safely into an immutable, encrypted **Amazon S3 Bucket**.
* **Cross-Region Failover Architecture:** For high-criticality tier-1 systems, maintain a secondary, dormant EKS cluster in a completely different geographical AWS Region (e.g., failing over traffic from `ap-south-1` Bangalore to `us-east-1` N. Virginia). ArgoCD excels here—you simply point your master `ApplicationSet` target destination server matrix to both clusters simultaneously, achieving **Multi-Region GitOps synchronization** out of the box.

---

## 🏁 The Complete Cloud-Native Production Blueprint

By combining these advanced day-2 operations with your core setup, your deployment workflow achieves top-tier architecture status:

```text
1. Infrastructure [Terraform + Karpenter + KMS + Multi-AZ RDS]
       │
       ▼
2. Platform Access [AWS IRSA + OIDC + External Secrets Operator]
       │
       ▼
3. Continuous Delivery [ArgoCD App-of-Apps + AppProjects + Sync Waves]
       │
       ▼
4. Network Perimeter [Kubernetes NetworkPolicies + PSS Isolation]
       │
       ▼
5. Runtime Application [Polyglot Pods + HPA Elasticity + PDB Budgets]
       │
       ▼
6. Observability & FinOps [OpenTelemetry Tracer + Kubecost + Grafana Dashboard]

```

This completes the full evolutionary cycle of your retail microservices platform—moving from a local Docker Compose sandbox to an enterprise-grade, highly secure, automated cloud architecture.