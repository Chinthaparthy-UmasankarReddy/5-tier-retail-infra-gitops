This setup handles the end-to-end lifecycle: installing ArgoCD on Amazon EKS, configuring security, setting up the complete directories, implementing the automation manifests using the **App-of-Apps design pattern**, and establishing the deployment pipeline.

---

## Part 1: Step-by-Step ArgoCD Cluster Installation Guide

Execute these commands in your local PowerShell terminal connected to your Amazon EKS cluster data plane.

### 1. Provision the Namespace & Deploy Core Control Plane

Create an isolated management boundary for the GitOps engines and apply the official community architecture:

```powershell
# Create the administrative namespace
kubectl create namespace argocd

# Apply the stable multi-tenant control plane deployment
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

### 2. Expose the API Control Server Locally

By default, the ArgoCD control server operates inside a private network footprint (`ClusterIP`). Establish a secure loopback tunnel to access the UI on your Windows 11 machine:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443

```

*Keep this terminal tab executing in the background.*

### 3. Retrieve the Auto-Generated Administrative Password

ArgoCD generates a random bootstrap password secured inside a Kubernetes secret layout. Run this decoded inquiry to extract it:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}

```

> **Access Steps:** Open your web browser to `https://localhost:8080`, log in using the username `admin`, and paste the extracted password string.

---

## Part 2: GitOps Infrastructure Repository Blueprint

This is the exact directory layout that must reside inside your **`5-retail-app-gitops`** repository. It leverages the **App-of-Apps** pattern to chain your configurations through a single root engine.

### Production Directory Visual Matrix

```text
5-retail-app-gitops/
│
├── argocd-infra/
│   ├── root-app-of-apps.yaml
│   └── applicationsets/
│       └── retail-apps-set.yaml
│
└── apps/
    ├── central-shared-chart/
    │   ├── Chart.yaml
    │   └── templates/
    │       ├── deployment.yaml
    │       └── service.yaml
    │
    └── microservices/
        ├── login-service/
        │   └── values.yaml
        ├── catalog-service/
        │   └── values.yaml
        └── cards-service/
            └── values.yaml

```

---

## Part 3: Infrastructure System Source Code

### 1. `argocd-infra/root-app-of-apps.yaml`

This root deployment file acts as the master anchor. It forces ArgoCD to scan your infrastructure configuration directory and orchestrate everything under it:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app-of-apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://github.com/Chinthaparthy-UmasankarReddy/5-retail-app-gitops.git'
    targetRevision: dev
    path: argocd-infra/applicationsets
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

### 2. `argocd-infra/applicationsets/retail-apps-set.yaml`

This controller automatically discovers and loops through your microservices directories, configuring all 5 backend applications using a single template block. It incorporates a **Sync Options** sequence to eliminate namespace race conditions:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: retail-microservices
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: 'https://github.com/Chinthaparthy-UmasankarReddy/5-retail-app-gitops.git'
        revision: dev
        directories:
          - path: apps/microservices/*
  template:
    metadata:
      name: '{{path.basename}}-app'
    spec:
      project: default
      sources:
        - repoURL: 'https://github.com/Chinthaparthy-UmasankarReddy/5-retail-app-gitops.git'
          targetRevision: dev
          path: apps/central-shared-chart
          helm:
            valueFiles:
              - '$git-repo/apps/microservices/{{path.basename}}/values.yaml'
        - repoURL: 'https://github.com/Chinthaparthy-UmasankarReddy/5-retail-app-gitops.git'
          targetRevision: dev
          ref: git-repo
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true

```

### 3. `apps/central-shared-chart/Chart.yaml`

```yaml
apiVersion: v2
name: central-shared-chart
description: Unified blueprints manifest engine for retail services
type: application
version: 1.0.0
appVersion: "1.0.0"

```

### 4. `apps/central-shared-chart/templates/deployment.yaml`

This dynamic blueprint reads parameters from each service's individual `values.yaml` file to inject the container images and balance replica configurations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.serviceName }}
  namespace: production
  labels:
    app: {{ .Values.serviceName }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  selector:
    matchLabels:
      app: {{ .Values.serviceName }}
  template:
    metadata:
      labels:
        app: {{ .Values.serviceName }}
    spec:
      containers:
        - name: {{ .Values.serviceName }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080

```

### 5. `apps/central-shared-chart/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.serviceName }}
  namespace: production
  labels:
    app: {{ .Values.serviceName }}
spec:
  type: {{ .Values.service.type | default "ClusterIP" }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8080
      protocol: TCP
  selector:
    app: {{ .Values.serviceName }}

```

### 6. Example Configuration Overrides (`values.yaml`)

Each backend service needs only a minimal `values.yaml` file to map its parameters to the shared templates.

* **`apps/microservices/login-service/values.yaml`**
```yaml
serviceName: login-service
replicaCount: 1
image:
  repository: umasankar33/5-retail-app-source-login-service
  tag: latest
service:
  port: 8081
  type: ClusterIP

```


* **`apps/microservices/cards-service/values.yaml`**
```yaml
serviceName: cards-service
replicaCount: 1
image:
  repository: umasankar33/5-retail-app-source-cards-service
  tag: latest
service:
  port: 8083
  type: ClusterIP

```



---

## Part 4: Production Infrastructure README Guide

Save this markdown file as **`README.md`** at the root of your **`5-retail-app-gitops`** repository to manage cluster configurations cleanly.

---

```markdown
# 🌐 5-Tier Retail Infrastructure Configuration Engine (GitOps Management)

This repository functions as the declarative git single source of truth for managing runtime components across your Amazon EKS data plane clusters. It implements advanced automation via ArgoCD ApplicationSets utilizing a centralized Helm orchestration paradigm.

---

## 🛠️ Step-by-Step Cluster Initial Execution Pipeline

### Step 1: Push Configurations to Your Remote Infrastructure Repo
Ensure all deployment configuration directories are saved and pushed to your isolated tracking workspace branch:
```powershell
git checkout dev
git add .
git commit -m "infra: implement app-of-apps pattern with strict namespace sync controls"
git push origin dev

```

### Step 2: Initialize GitOps Pipeline Automation

Execute the master root anchor deployment manifest onto your target cluster context. This bootstraps the system and triggers downstream resource generation:

```powershell
kubectl apply -f argocd-infra/root-app-of-apps.yaml

```

### Step 3: Verify Automated Deployment Sync Status

Monitor your target cluster boundary as the engine creates the `production` namespace and initializes the container workloads:

```powershell
# Check the sync health of your backend infrastructure deployment resources
kubectl get applications -n argocd

# Stream pod status checks within your application runtime namespace context
kubectl get pods -n production -w

```

---

## 🛠️ Automated Operations Runbook & Maintenance Tasks

### How to Introduce or Onboard a New Microservice

To deploy a new microservice tier (e.g., an internal reporting engine), you do not need to configure any new ArgoCD control structures. The `ApplicationSet` generator automatically discovers new service targets dynamically:

1. Create a new child configuration subdirectory: `apps/microservices/reporting-service/`
2. Populate the folder with a standard override configuration blueprint (`values.yaml`):
```yaml
serviceName: reporting-service
replicaCount: 2
image:
  repository: umasankar33/reporting-service
  tag: latest
service:
  port: 8086

```


3. Commit and push the folder to your `dev` branch. ArgoCD will instantly discover the new files, build out the application card in the dashboard interface, and map the workloads inside the target namespace!

### How to Force Synchronization and Cache Clearing

If your application hits a temporary structural state mismatch (such as a transient image pull bug or a stale cache layer), force the management plane to clear out its old memory tracking layers:

```powershell
# Force the root application anchor engine to clear tracking memory states
kubectl patch app root-app-of-apps -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"dev"}}}'

# Execute a hard programmatic refresh across the internal microservices structure
kubectl rollout restart deployment cards-service -n production

```

```

```