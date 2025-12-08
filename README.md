# Platform Middle Clients - GitOps Repository

This repository contains ArgoCD configurations for deploying applications to Medium-tier clients using GitOps practices.

## Repository Structure

```
platform-middleclients-gitops/
├── tenants/
│   └── demo-backstage-uat/              # Example tenant for testing
│       ├── argocd/
│       │   ├── project.yaml             # ArgoCD AppProject
│       │   ├── applications/
│       │   │   └── omnix-app.yaml       # ArgoCD Application
│       │   └── values/
│       │       └── omnix/
│       │           └── values.yaml      # Custom Helm values
│       └── catalog-info.yaml            # Backstage catalog entity
└── README.md
```

## What is This Repository For?

This repository demonstrates the **GitOps pattern** for deploying applications using:
- **ArgoCD**: Continuous deployment tool that syncs Kubernetes resources from Git
- **Backstage**: Developer portal that provides visibility into deployments
- **Helm**: Package manager for Kubernetes applications

## Demo Tenant: demo-backstage-uat

### Purpose
Test deployment to learn ArgoCD + Backstage integration before building automated templates.

### What Gets Deployed
- **Application**: Nginx (simple web server for testing)
- **Helm Chart**: Omnix chart from https://github.com/lucky29-git/Omnix
- **Namespace**: `uat`
- **Resources**: 1 Deployment, 1 Service, 1 Pod

### Components

#### 1. ArgoCD AppProject (`argocd/project.yaml`)
**Purpose**: Security boundary that defines:
- Which Git repositories applications can use
- Which Kubernetes namespaces they can deploy to
- What types of resources they can create

**Key Configuration**:
- **Name**: `demo-backstage-uat`
- **Allowed Source Repos**: Omnix chart, this GitOps repo
- **Allowed Destinations**: `uat` namespace, `deployment-platform` cluster

#### 2. ArgoCD Application (`argocd/applications/omnix-app.yaml`)
**Purpose**: Defines WHAT to deploy and WHERE

**Key Configuration**:
- **Name**: `demo-backstage-omnix`
- **Project**: `demo-backstage-uat`
- **Sources**:
  - Helm chart from Omnix repo
  - Custom values from this repo
- **Destination**: `uat` namespace
- **Sync Policy**: Automated (auto-sync, auto-prune, auto-heal)

#### 3. Helm Values (`argocd/values/omnix/values.yaml`)
**Purpose**: Customizes the Helm chart for this specific deployment

**Key Settings**:
- **Image**: `nginx:alpine` (simple test application)
- **Replicas**: 1
- **Resources**: Minimal (50m CPU, 64Mi RAM)
- **Service**: ClusterIP (internal-only)
- **Probes**: Disabled (for simplicity)

#### 4. Backstage Entity (`catalog-info.yaml`)
**Purpose**: Registers this deployment in Backstage for visibility

**Key Annotations**:
- `argocd/app-name`: Links to ArgoCD application
- `backstage.io/kubernetes-id`: Links to Kubernetes resources
- `github.com/project-slug`: Links to this GitOps repo

---

## Deployment Instructions

### Prerequisites
- ArgoCD installed in `deployment-platform` cluster ✅
- Backstage running with ArgoCD plugin ✅
- `uat` namespace will be created automatically

### Step 1: Push This Repository to GitHub

```bash
cd /Users/chandrashekhar29/appointy/poc/platform-middleclients-gitops

# Initialize git if not already done
git init
git add .
git commit -m "Initial commit: Demo Backstage UAT tenant with Omnix deployment"

# Push to GitHub (assuming remote is already added)
git branch -M main
git push -u origin main
```

### Step 2: Deploy ArgoCD Project

```bash
# Apply the AppProject
kubectl apply -f tenants/demo-backstage-uat/argocd/project.yaml

# Verify project was created
kubectl get appproject -n argocd demo-backstage-uat
```

**Expected Output**:
```
NAME                  AGE
demo-backstage-uat    5s
```

### Step 3: Deploy ArgoCD Application

```bash
# Apply the Application
kubectl apply -f tenants/demo-backstage-uat/argocd/applications/omnix-app.yaml

# Watch the deployment
kubectl get application -n argocd demo-backstage-omnix -w
```

**Expected Status Progression**:
```
NAME                   SYNC STATUS   HEALTH STATUS
demo-backstage-omnix   OutOfSync     Missing
demo-backstage-omnix   Synced        Progressing
demo-backstage-omnix   Synced        Healthy
```

### Step 4: Verify in ArgoCD UI

1. Open ArgoCD UI: https://localhost:8080
2. Login with:
   - Username: `admin`
   - Password: `xcyWq8nXSovWaDH5`
3. You should see:
   - **Project**: `demo-backstage-uat`
   - **Application**: `demo-backstage-omnix`
   - **Status**: Synced ✓, Healthy ✓

### Step 5: Verify in Kubernetes

```bash
# Check namespace was created
kubectl get namespace uat

# Check pods are running
kubectl get pods -n uat

# Check deployment
kubectl get deployment -n uat demo-omnix-app

# Check service
kubectl get service -n uat demo-omnix-app
```

**Expected Resources**:
```
NAME                              READY   STATUS    RESTARTS   AGE
demo-omnix-app-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### Step 6: Register in Backstage

1. Open Backstage: http://localhost:3000
2. Navigate to: **Catalog** → **Create** → **Register Existing Component**
3. Enter URL:
   ```
   https://github.com/AppointyTech/platform-middleclients-gitops/blob/main/tenants/demo-backstage-uat/catalog-info.yaml
   ```
4. Click **Analyze** → **Import**

### Step 7: View in Backstage

1. Navigate to: **Catalog** → **Components** → **demo-backstage-uat**
2. Click the **CI/CD** tab
3. You should see:
   - **ArgoCD Section** with:
     - Application: `demo-backstage-omnix`
     - Status: ✓ Synced, Healthy
     - Link to ArgoCD UI

---

## Troubleshooting

### Application Shows "OutOfSync"
**Cause**: ArgoCD detected the application but hasn't synced yet

**Solution**:
```bash
# Manually trigger sync
argocd app sync demo-backstage-omnix

# Or wait for auto-sync (happens every 3 minutes)
```

### Application Shows "Degraded" or "Unhealthy"
**Cause**: Pods are failing to start

**Check**:
```bash
# View pod status
kubectl get pods -n uat

# View pod logs
kubectl logs -n uat -l app=demo-omnix

# Describe pod for events
kubectl describe pod -n uat -l app=demo-omnix
```

### ArgoCD Can't Pull from GitHub
**Cause**: Repository not accessible or wrong URL

**Solution**:
- Verify repository is public or ArgoCD has access
- Check the `repoURL` in `omnix-app.yaml` matches your GitHub repo

### Backstage Doesn't Show ArgoCD Tab
**Causes**:
1. Component not registered in Backstage
2. `argocd/app-name` annotation missing or incorrect
3. ArgoCD plugin not configured

**Check**:
```bash
# Verify annotation in catalog-info.yaml
grep "argocd/app-name" tenants/demo-backstage-uat/catalog-info.yaml
```

### "Namespace 'uat' not found"
**Cause**: Namespace wasn't created automatically

**Solution**:
```bash
# Create namespace manually
kubectl create namespace uat

# Then trigger sync again
argocd app sync demo-backstage-omnix
```

---

## Testing the Deployment

### Access the Nginx Application

```bash
# Port-forward to the service
kubectl port-forward -n uat svc/demo-omnix-app 8081:80

# Open in browser
open http://localhost:8081
```

You should see the **Nginx welcome page**.

### Update the Deployment (Test GitOps Flow)

1. **Edit** `tenants/demo-backstage-uat/argocd/values/omnix/values.yaml`
2. **Change** replica count:
   ```yaml
   replicaCount: 2  # Changed from 1
   ```
3. **Commit and push**:
   ```bash
   git add tenants/demo-backstage-uat/argocd/values/omnix/values.yaml
   git commit -m "Scale demo app to 2 replicas"
   git push
   ```
4. **Watch ArgoCD sync** (within 3 minutes):
   - ArgoCD detects change
   - Auto-syncs to cluster
   - Scales deployment to 2 pods
5. **Verify in Kubernetes**:
   ```bash
   kubectl get pods -n uat
   # You should see 2 pods running
   ```
6. **Verify in Backstage**: CI/CD tab shows updated sync time

---

## Next Steps

After successfully deploying this demo:

1. **Install Kubernetes Plugin in Backstage**
   - See pods, logs, and events in Backstage
   - More visibility into deployment health

2. **Create More Complex Applications**
   - Add multiple services (API, database, etc.)
   - Use real Helm values configurations

3. **Build Backstage Templates**
   - Automate this entire process
   - Generate all YAMLs from a form
   - See `tasks.md` and `reference-structure.md` for details

4. **Add More Tenants**
   - Copy `demo-backstage-uat` structure
   - Customize for different clients
   - Deploy to different namespaces

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         GitHub                               │
│  platform-middleclients-gitops (this repo)                  │
│  ├── project.yaml                                            │
│  ├── omnix-app.yaml                                          │
│  ├── values.yaml                                             │
│  └── catalog-info.yaml                                       │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ Git Pull (every 3 min)
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      ArgoCD                                  │
│  (Running in deployment-platform cluster)                   │
│  ├── Project: demo-backstage-uat                            │
│  └── Application: demo-backstage-omnix                      │
│      ├── Syncs: Helm chart + values                         │
│      ├── Deploys: To uat namespace                          │
│      └── Status: Synced, Healthy                            │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ kubectl apply
                 ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster                              │
│  Namespace: uat                                              │
│  ├── Deployment: demo-omnix-app                             │
│  ├── Pod: demo-omnix-app-xxx (nginx:alpine)                 │
│  └── Service: demo-omnix-app (ClusterIP)                    │
└─────────────────────────────────────────────────────────────┘
                 ▲
                 │ Queries status
                 │
┌─────────────────────────────────────────────────────────────┐
│                     Backstage                                │
│  Component: demo-backstage-uat                              │
│  └── CI/CD Tab                                               │
│      └── ArgoCD Section                                      │
│          └── Shows: demo-backstage-omnix status             │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Concepts Demonstrated

### GitOps Workflow
1. **Desired state** defined in Git (this repo)
2. **ArgoCD** continuously syncs Git → Kubernetes
3. **Developers** change Git, ArgoCD applies changes automatically
4. **No direct kubectl** commands needed for deployments

### Multi-Source Pattern
- **Source 1**: Helm chart (reusable template)
- **Source 2**: Custom values (client-specific config)
- **Result**: Same chart, different configurations per client

### Separation of Concerns
- **Helm Charts**: Generic application templates (rarely change)
- **GitOps Repo**: Client-specific configurations (change frequently)
- **ArgoCD**: Deployment automation (no manual intervention)
- **Backstage**: Developer visibility (self-service)

---

## Resources

- **ArgoCD Documentation**: https://argo-cd.readthedocs.io/
- **Omnix Helm Chart**: https://github.com/lucky29-git/Omnix
- **Backstage Documentation**: https://backstage.io/docs/
- **Internal Docs**:
  - `setup-backstage.md` - Backstage setup guide
  - `tasks.md` - Template development tasks
  - `reference-structure.md` - Reference GitOps structure

---

**Questions?** Check the troubleshooting section or review the deployment steps.
