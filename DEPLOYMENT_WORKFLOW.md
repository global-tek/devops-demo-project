# Code Change Deployment Workflow

## Problem Statement
Changes to `DemoApplication.java` or `index.html` are not reflected in the running application.

## Root Cause
- Source code changes are local only
- Docker image with same tag (`V1.0`) contains old code
- Kubernetes uses cached image due to `imagePullPolicy: IfNotPresent`
- ArgoCD syncs Helm manifests, not source code

## Solution: Proper CI/CD Workflow

### Option 1: Manual Build & Deploy (Quick Testing)

#### Step 1: Update Version Number
```bash
# Choose a new version tag
export NEW_VERSION="V1.1"  # or V1.2, V2.0, etc.
```

#### Step 2: Build the Application
```bash
cd /home/ubuntu/devops-demo-project/demo-java-app

# Build JAR file
mvn clean package -DskipTests
```

#### Step 3: Build Docker Image
```bash
# Build for amd64 architecture (important!)
docker build --platform linux/amd64 \
  -t betechinc/demo-java-app:${NEW_VERSION} .
```

#### Step 4: Push to Docker Hub
```bash
# Login to Docker Hub (if not already logged in)
docker login

# Push the new image
docker push betechinc/demo-java-app:${NEW_VERSION}
```

#### Step 5: Update Helm Values
```bash
cd /home/ubuntu/devops-demo-project

# Update the image tag in values.yaml
sed -i "s/tag: .*/tag: ${NEW_VERSION}/" helm/app/values.yaml

# Commit and push to trigger ArgoCD sync
git add helm/app/values.yaml
git commit -m "Update app to version ${NEW_VERSION}"
git push
```

#### Step 6: Wait for ArgoCD Auto-Sync
```bash
# ArgoCD will automatically detect changes and deploy
# Or manually sync:
kubectl get applications -n argocd

# Check deployment progress
kubectl get pods -n app -w
```

---

### Option 2: Jenkins Pipeline (Automated)

Your Jenkins pipeline already exists at `ci/Jenkins/Jenkinsfile`. To trigger it:

#### Prerequisites
```bash
# Ensure Jenkins is running
docker ps | grep jenkins

# Access Jenkins UI (get admin password)
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Trigger Build
1. Access Jenkins UI (usually on port 8080)
2. Navigate to your pipeline job
3. Click "Build with Parameters"
4. Enter new version (e.g., `V1.1`)
5. Click "Build"

The pipeline will:
- ✅ Checkout code from GitHub
- ✅ Build JAR with Maven
- ✅ Run SonarQube analysis
- ✅ Build Docker image (amd64 platform)
- ✅ Push to Docker Hub
- ✅ Update `helm/app/values.yaml` with new version
- ✅ Commit and push changes
- ✅ ArgoCD auto-syncs the deployment

---

### Option 3: Force Restart (Same Version - NOT RECOMMENDED)

If you rebuilt the image with the same tag:

```bash
# This is a workaround, not best practice
kubectl rollout restart deployment/demo-java-app-deployment -n app

# Or delete pods to force image pull
kubectl delete pods -n app -l app=demo-java-app
```

**⚠️ Warning:** This only works if you:
1. Rebuilt the Docker image with same tag
2. Pushed it to Docker Hub
3. Changed imagePullPolicy to `Always` (not recommended for production)

---

## Best Practices

### 1. Always Use New Version Tags
```bash
# Semantic versioning
V1.0 → V1.1 (minor changes)
V1.1 → V2.0 (major changes)
V1.0 → V1.0.1 (patches)
```

### 2. Automate with Jenkins
- Define the pipeline in `Jenkinsfile`
- Use GitHub webhooks to trigger on commits
- Let Jenkins handle build/push/deploy

### 3. Use Git Branches
```bash
# Development branch
git checkout -b feature/new-ui
# Make changes to index.html
git commit -m "Update UI"
git push origin feature/new-ui

# Merge to main after testing
git checkout main
git merge feature/new-ui
# This triggers Jenkins build
```

### 4. Monitor Deployment
```bash
# Watch ArgoCD status
kubectl get applications -n argocd

# Watch pod rollout
kubectl rollout status deployment/demo-java-app-deployment -n app

# View pod logs
kubectl logs -n app -l app=demo-java-app --tail=50 -f
```

---

## Quick Reference Commands

### Check Current Version
```bash
# View deployed image version
kubectl get deployment demo-java-app-deployment -n app \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# View values.yaml version
grep "tag:" helm/app/values.yaml
```

### Build and Deploy New Version
```bash
# 1. Set version
export VERSION="V1.2"

# 2. Build app
cd demo-java-app && mvn clean package -DskipTests && cd ..

# 3. Build & push image
cd demo-java-app && \
docker build --platform linux/amd64 -t betechinc/demo-java-app:${VERSION} . && \
docker push betechinc/demo-java-app:${VERSION} && \
cd ..

# 4. Update Helm and commit
sed -i "s/tag: .*/tag: ${VERSION}/" helm/app/values.yaml && \
git add helm/app/values.yaml && \
git commit -m "Deploy version ${VERSION}" && \
git push

# 5. Watch deployment
kubectl get pods -n app -w
```

---

## Troubleshooting

### Changes Still Not Visible?

#### Check 1: Image Was Actually Built
```bash
docker images | grep demo-java-app
# Should show your new version tag
```

#### Check 2: Image Was Pushed to Registry
```bash
# Try pulling the image
docker pull betechinc/demo-java-app:V1.1
```

#### Check 3: Helm Values Updated
```bash
cat helm/app/values.yaml | grep tag:
# Should show new version
```

#### Check 4: Git Changes Were Pushed
```bash
git log --oneline -5
# Should show your commit
```

#### Check 5: ArgoCD Detected Changes
```bash
kubectl get application demo-java-app -n argocd -o yaml | grep revision
# Compare with: git rev-parse HEAD
```

#### Check 6: Pods Are Running New Image
```bash
kubectl describe pod -n app -l app=demo-java-app | grep "Image:"
# Should show new version
```

#### Check 7: Pod Logs Show Startup
```bash
kubectl logs -n app -l app=demo-java-app --tail=20
# Should show Spring Boot startup messages
```

---

## Architecture Overview

### Full CI/CD Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ DEVELOPER WORKFLOW (CI - Continuous Integration)               │
└─────────────────────────────────────────────────────────────────┘

[1. Edit Source Files]
   ├─ index.html (CSS, HTML changes)
   ├─ DemoApplication.java (Java code)
   └─ pom.xml (dependencies)
          ↓
[2. Maven Build]
   └─ mvn clean package
          ↓
   Creates: target/demo-java-app.jar
          ↓
[3. Docker Build] ⚠️ USE NEW TAG!
   └─ docker build -t betechinc/demo-java-app:V1.1 .
          ↓
   Creates: Docker Image (V1.1) with your code
          ↓
[4. Docker Push]
   └─ docker push betechinc/demo-java-app:V1.1
          ↓
   Stored in: Docker Hub Registry
          ↓
[5. Update Deployment Config]
   └─ sed -i 's/tag: V1.0/tag: V1.1/' helm/app/values.yaml
          ↓
[6. Git Commit & Push]
   └─ git add helm/app/values.yaml
   └─ git commit -m "Deploy V1.1"
   └─ git push origin main
          ↓
   Pushed to: GitHub Repository

┌─────────────────────────────────────────────────────────────────┐
│ ARGOCD WORKFLOW (CD - Continuous Deployment)                   │
└─────────────────────────────────────────────────────────────────┘

[7. ArgoCD Detects Changes]
   └─ Polls GitHub every 3 minutes
   └─ Detects: values.yaml changed (tag: V1.0 → V1.1)
          ↓
[8. ArgoCD Sync]
   └─ Updates Kubernetes Deployment manifest
   └─ Changes image to: betechinc/demo-java-app:V1.1
          ↓
[9. Kubernetes Rolling Update]
   └─ Pulls new image from Docker Hub
   └─ Creates new pods with V1.1
   └─ Terminates old pods with V1.0
          ↓
[10. Your Changes Are Live! ✅]
    └─ New CSS gradient visible
    └─ New title text visible
```

### Why Source Code Changes Don't Auto-Deploy

```
❌ WHAT DOESN'T WORK:
┌──────────────────┐
│ Edit index.html  │
│ Edit .java files │
└────────┬─────────┘
         │
         ↓
    [ArgoCD] ← Doesn't watch source code!
         ↓
    No deployment ❌

✅ WHAT WORKS:
┌──────────────────┐
│ Edit source code │
└────────┬─────────┘
         │
         ↓
┌────────────────────┐
│ Build new image    │
│ with NEW tag       │
└────────┬───────────┘
         │
         ↓
┌────────────────────┐
│ Update values.yaml │
│ tag: V1.0 → V1.1   │
└────────┬───────────┘
         │
         ↓
┌────────────────────┐
│ Git push           │
└────────┬───────────┘
         │
         ↓
    [ArgoCD] ← Detects values.yaml change!
         ↓
    Deployment happens ✅
```

---

## Summary

**Every code change requires:**
1. ✅ New version tag (increment V1.0 → V1.1)
2. ✅ Build JAR (`mvn package`)
3. ✅ Build Docker image with new tag
4. ✅ Push image to Docker Hub
5. ✅ Update `helm/app/values.yaml` with new tag
6. ✅ Commit and push to Git
7. ✅ ArgoCD auto-syncs and deploys

**Without these steps, Kubernetes will continue running the old cached image.**
