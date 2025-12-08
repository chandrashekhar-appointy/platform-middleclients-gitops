# App-of-Apps Deployment Guide

This guide shows you how to deploy the demo tenant using the **App-of-Apps pattern** for fully automated GitOps.

## What is App-of-Apps?

**App-of-Apps** is an ArgoCD pattern where:
- ONE "manager" Application watches a folder of Application YAML files
- It automatically creates/updates/deletes Applications based on Git
- You only apply the manager once; everything else is automated

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Repository                       │
│  platform-middleclients-gitops                              │
│                                                              │
│  tenants/demo-backstage-uat/argocd/                         │
│  ├── app-of-apps.yaml         ← Apply ONCE manually         │
│  ├── applications/                                           │
│  │   ├── project.yaml         ← Auto-discovered            │
│  │   └── omnix-app.yaml       ← Auto-discovered            │
│  └── values/omnix/                                           │
│      └── values.yaml           ← Auto-synced                │
└─────────────────────────────────────────────────────────────┘
                       │
                       │ (1) Manual bootstrap - ONE TIME
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                         ArgoCD                               │
│                                                              │
│  App-of-Apps: demo-backstage-uat-apps                       │
│  ├── Watches: applications/ folder                          │
│  ├── Auto-creates: AppProject (project.yaml)                │
│  └── Auto-creates: Application (omnix-app.yaml)             │
│      └── Auto-syncs: values.yaml → Kubernetes               │
└─────────────────────────────────────────────────────────────┘
                       │
                       │ (2) Automatic deployment
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  Namespace: uat                                              │
│  ├── Deployment: demo-omnix-app                             │
│  ├── Pod: demo-omnix-app-xxx (nginx:alpine)                 │
│  └── Service: demo-omnix-app                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

✅ ArgoCD installed in deployment-platform cluster
✅ kubectl access to the cluster
✅ Git repository pushed to GitHub

---

## Deployment Steps

### Step 1: Push Repository to GitHub

```bash
cd /Users/chandrashekhar29/appointy/poc/platform-middleclients-gitops

# Initialize and commit
git add .
git commit -m "Initial commit: App-of-Apps pattern for demo-backstage-uat"

# Push to GitHub
git push -u origin main
```

**Verify:** Check that your repository is accessible at:
```
https://github.com/AppointyTech/platform-middleclients-gitops
```

---

### Step 2: Deploy App-of-Apps (ONE-TIME BOOTSTRAP)

This is the **ONLY** manual kubectl command you'll ever need for this tenant!

```bash
# Apply the App-of-Apps
kubectl apply -f tenants/demo-backstage-uat/argocd/app-of-apps.yaml
```

**Expected Output:**
```
application.argoproj.io/demo-backstage-uat-apps created
```

---

### Step 3: Watch Automatic Discovery

The App-of-Apps will now automatically discover and create applications from the `applications/` folder.

```bash
# Watch the App-of-Apps status
kubectl get application -n argocd demo-backstage-uat-apps -w
```

**Status Progression:**
```
NAME                      SYNC STATUS   HEALTH STATUS
demo-backstage-uat-apps   OutOfSync     Missing
demo-backstage-uat-apps   Synced        Progressing
demo-backstage-uat-apps   Synced        Healthy
```

Press `Ctrl+C` when status shows `Synced` and `Healthy`.

---

### Step 4: Verify Applications Were Auto-Created

```bash
# List all applications in the demo-backstage-uat project
kubectl get applications -n argocd -l client=demo-backstage
```

**Expected Output:**
```
NAME                      SYNC STATUS   HEALTH STATUS
demo-backstage-uat-apps   Synced        Healthy        (App-of-Apps manager)
demo-backstage-omnix      Synced        Healthy        (Auto-created by App-of-Apps)
```

**Also check the AppProject:**
```bash
kubectl get appproject -n argocd demo-backstage-uat
```

**Expected Output:**
```
NAME                  AGE
demo-backstage-uat    30s
```

---

### Step 5: Verify in ArgoCD UI

1. **Open ArgoCD UI:** https://localhost:8080
2. **Login:**
   - Username: `admin`
   - Password: `xcyWq8nXSovWaDH5`
3. **You should see:**
   - **Application:** `demo-backstage-uat-apps` (App-of-Apps)
     - Status: ✓ Synced, Healthy
     - Click to see it manages 2 child resources (project + omnix app)
   - **Application:** `demo-backstage-omnix`
     - Status: ✓ Synced, Healthy
     - Click to see deployed resources (Deployment, Service, Pod)

---

### Step 6: Verify in Kubernetes

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

**Expected Resources:**
```
NAMESPACE   NAME                              READY   STATUS    RESTARTS   AGE
uat         demo-omnix-app-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demo-omnix-app   ClusterIP   10.XX.XXX.XXX   <none>        80/TCP    2m
```

---

### Step 7: Test the Application

```bash
# Port-forward to the nginx service
kubectl port-forward -n uat svc/demo-omnix-app 8081:80

# Open in browser (or curl)
open http://localhost:8081
# OR
curl http://localhost:8081
```

**Expected:** You should see the **nginx welcome page**.

---

### Step 8: Register in Backstage

1. **Open Backstage:** http://localhost:3000
2. **Navigate to:** Catalog → Create → Register Existing Component
3. **Enter URL:**
   ```
   https://github.com/AppointyTech/platform-middleclients-gitops/blob/main/tenants/demo-backstage-uat/catalog-info.yaml
   ```
4. **Click:** Analyze → Import
5. **Navigate to:** Catalog → Components → **demo-backstage-uat**
6. **Click:** CI/CD tab
7. **You should see:**
   - **ArgoCD Section** showing:
     - Application: `demo-backstage-omnix`
     - Status: ✓ Synced, Healthy
     - Last Synced: [timestamp]

---

## What Happens Next? (GitOps in Action)

### Scenario 1: Scale the Application

**Edit values.yaml:**
```bash
cd /Users/chandrashekhar29/appointy/poc/platform-middleclients-gitops

# Edit values file
nano tenants/demo-backstage-uat/argocd/values/omnix/values.yaml

# Change replicaCount from 1 to 2:
replicaCount: 2  # Changed from 1

# Commit and push
git add tenants/demo-backstage-uat/argocd/values/omnix/values.yaml
git commit -m "Scale demo app to 2 replicas"
git push
```

**What happens automatically:**
1. **ArgoCD detects change** (within 3 minutes)
2. **omnix-app Application shows OutOfSync**
3. **Auto-sync triggers** (because we have `automated: true`)
4. **Deployment scales to 2 pods**
5. **Status returns to Synced**

**Verify:**
```bash
# Wait 3-5 minutes, then check
kubectl get pods -n uat
# You should see 2 pods now!
```

### Scenario 2: Add a New Application

**Add a new Application YAML:**
```bash
# Create a new application file
nano tenants/demo-backstage-uat/argocd/applications/new-app.yaml

# Commit and push
git add tenants/demo-backstage-uat/argocd/applications/new-app.yaml
git commit -m "Add new application"
git push
```

**What happens automatically:**
1. **App-of-Apps detects new file** in applications/ folder
2. **Auto-creates ArgoCD Application** from new-app.yaml
3. **New application deploys** to cluster

**NO kubectl commands needed!**

### Scenario 3: Delete an Application

**Remove Application YAML from Git:**
```bash
git rm tenants/demo-backstage-uat/argocd/applications/omnix-app.yaml
git commit -m "Remove omnix application"
git push
```

**What happens automatically:**
1. **App-of-Apps detects deletion**
2. **Deletes omnix Application from ArgoCD**
3. **ArgoCD prunes resources** (deletes pods, services, etc.)

**This is the power of `prune: true`!**

---

## Troubleshooting

### App-of-Apps Shows OutOfSync

**Check what's different:**
```bash
# View App-of-Apps details
kubectl describe application -n argocd demo-backstage-uat-apps
```

**Manually sync:**
```bash
# Install ArgoCD CLI first (if not installed)
brew install argocd

# Login
argocd login localhost:8080 --username admin --password xcyWq8nXSovWaDH5 --insecure

# Sync
argocd app sync demo-backstage-uat-apps
```

### Applications Not Being Created

**Check App-of-Apps is watching correct path:**
```bash
kubectl get application -n argocd demo-backstage-uat-apps -o yaml | grep path
```

**Should show:**
```yaml
path: tenants/demo-backstage-uat/argocd/applications
```

**Check GitHub repo is accessible:**
```bash
# Verify repo URL
kubectl get application -n argocd demo-backstage-uat-apps -o yaml | grep repoURL
```

### Child Application Shows OutOfSync After Git Push

**Cause:** ArgoCD polls every 3 minutes by default

**Solutions:**
1. **Wait 3 minutes** for auto-sync
2. **Manually sync:**
   ```bash
   argocd app sync demo-backstage-omnix
   ```
3. **Enable webhook** for instant sync (advanced)

---

## Cleanup

To remove everything:

```bash
# Delete the App-of-Apps (this deletes all child applications too!)
kubectl delete application -n argocd demo-backstage-uat-apps

# Verify all applications are deleted
kubectl get applications -n argocd -l client=demo-backstage

# Clean up namespace (if needed)
kubectl delete namespace uat
```

**Note:** Because `prune: true` is enabled, deleting the App-of-Apps will:
1. Delete the AppProject
2. Delete the omnix Application
3. Delete all deployed resources (pods, services, etc.)

---

## Summary: One Command vs Traditional

### Traditional Approach (Manual):
```bash
kubectl apply -f project.yaml
kubectl apply -f application1.yaml
kubectl apply -f application2.yaml
kubectl apply -f application3.yaml
# ... repeat for every new application
```

### App-of-Apps Approach (Automated):
```bash
# ONE TIME ONLY:
kubectl apply -f app-of-apps.yaml

# AFTER THIS: Everything is GitOps!
# Just commit to Git, ArgoCD auto-deploys
```

---

## Next Steps

✅ **App-of-Apps deployed and working**
✅ **Application auto-discovered and deployed**
✅ **Registered in Backstage**

**What's next:**
1. Test GitOps workflow (scale replicas, see auto-sync)
2. Install Kubernetes plugin in Backstage (see pods in UI)
3. Create more complex applications
4. Build Backstage template to automate this entire setup

---

**You now have fully automated GitOps!** 🎉

Any changes to Git → ArgoCD auto-deploys → Backstage shows status

No more kubectl commands needed!
