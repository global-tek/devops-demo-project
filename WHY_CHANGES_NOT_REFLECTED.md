# Why Your Code Changes Are Not Reflected

## The Problem
You changed:
- `demo-java-app/src/main/resources/templates/index.html` (CSS gradient)
- `demo-java-app/src/main/java/com/deepak/demo/DemoApplication.java` (title text)

But the running app still shows the old version.

## Why This Happens

### Understanding Container Images

Your running pods use this Docker image:
```
betechinc/demo-java-app:V1.0
```

This image was built at some point in the past and contains:
- ✅ Compiled Java bytecode (from old DemoApplication.java)
- ✅ Static files (old index.html with old CSS)
- ✅ Spring Boot JAR file

**This image is immutable** - it's frozen in time and stored in Docker Hub.

### What ArgoCD Actually Does

```yaml
# ArgoCD watches this (helm/app/values.yaml):
image:
  githubID: betechinc
  repository: demo-java-app
  tag: V1.0  ← ArgoCD only sees THIS
```

ArgoCD's job:
1. ✅ Monitor Git repository for changes to Helm charts
2. ✅ Sync Kubernetes deployments when `values.yaml` or templates change
3. ❌ Does NOT build Docker images
4. ❌ Does NOT compile Java code
5. ❌ Does NOT watch source code files

### The Current Situation

```
Your Local Files:              Docker Hub:                Running Pods:
─────────────────              ──────────                ─────────────
index.html (NEW) ───X───→     V1.0 (OLD) ───────→      Using V1.0 (OLD)
DemoApplication.java (NEW)      ↑                            ↑
                                │                            │
                         Built weeks ago              Pulled from Docker Hub
                                                      when tag=V1.0 in values.yaml
```

**Your changes only exist in your local filesystem!**

## The Fix: Complete CI/CD Pipeline

### Step 1: Build New JAR File
```bash
cd /home/ubuntu/devops-demo-project/demo-java-app
mvn clean package -DskipTests
```
This compiles your NEW Java code and packages with NEW index.html

### Step 2: Build New Docker Image with NEW TAG
```bash
# Use a NEW version tag (V1.0 → V1.1)
docker build --platform linux/amd64 -t betechinc/demo-java-app:V1.1 .
```
This creates a NEW image containing your changes

### Step 3: Push to Docker Hub
```bash
docker push betechinc/demo-java-app:V1.1
```
Now the image with your changes is available in the registry

### Step 4: Update Helm Values
```bash
cd /home/ubuntu/devops-demo-project

# Update the tag
sed -i 's/tag: V1.0/tag: V1.1/' helm/app/values.yaml

# Commit and push
git add helm/app/values.yaml
git commit -m "Deploy version V1.1 with UI and text updates"
git push
```

### Step 5: ArgoCD Auto-Sync
ArgoCD detects the change in `values.yaml`:
```diff
  image:
    githubID: betechinc
    repository: demo-java-app
-   tag: V1.0
+   tag: V1.1
```

Then ArgoCD:
1. ✅ Updates the Deployment manifest
2. ✅ Kubernetes pulls `betechinc/demo-java-app:V1.1` from Docker Hub
3. ✅ Creates new pods with your changes
4. ✅ Your changes are now visible!

## Why ArgoCD Cannot Watch Source Code

### It's Not a Bug, It's Architecture

**ArgoCD is a GitOps CD (Continuous Deployment) tool:**
- Deploys pre-built artifacts (Docker images)
- Ensures cluster state matches Git
- Declarative configuration only

**You need a CI (Continuous Integration) tool for building:**
- Jenkins (you already have this!)
- GitHub Actions
- GitLab CI
- CircleCI

### The Proper Workflow

```
1. Developer commits code changes
        ↓
2. CI Tool (Jenkins) detects commit
        ↓
3. Jenkins builds JAR file
        ↓
4. Jenkins builds Docker image with NEW tag
        ↓
5. Jenkins pushes image to Docker Hub
        ↓
6. Jenkins updates values.yaml with new tag
        ↓
7. Jenkins commits and pushes values.yaml
        ↓
8. ArgoCD detects values.yaml change
        ↓
9. ArgoCD syncs deployment
        ↓
10. New pods run with your changes ✅
```

## Using Jenkins (Automated)

Your Jenkins pipeline already does all of this! Located at:
`ci/Jenkins/Jenkinsfile`

### To Use Jenkins:

1. **Start Jenkins** (if not running):
```bash
docker ps | grep jenkins
# If not running: docker start jenkins
```

2. **Access Jenkins UI**:
```bash
# Get admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Access on port 8080
```

3. **Trigger Build**:
- Click your pipeline job
- Click "Build with Parameters"
- Enter version: `V1.1` (or any new version)
- Click "Build"

4. **Jenkins will automatically**:
   - ✅ Checkout your code from GitHub
   - ✅ Build JAR with your changes
   - ✅ Build Docker image as `betechinc/demo-java-app:V1.1`
   - ✅ Push to Docker Hub
   - ✅ Update `helm/app/values.yaml`
   - ✅ Commit and push to Git
   - ✅ ArgoCD auto-syncs

5. **Wait 1-2 minutes**:
```bash
kubectl get pods -n app -w
```
You'll see new pods spinning up with your changes!

## Quick Manual Deployment (All Steps)

If you want to deploy manually right now:

```bash
# Set new version
export VERSION="V1.1"

# Navigate to project
cd /home/ubuntu/devops-demo-project

# Build JAR
cd demo-java-app
mvn clean package -DskipTests

# Build Docker image
docker build --platform linux/amd64 -t betechinc/demo-java-app:${VERSION} .

# Push to Docker Hub (requires login)
docker push betechinc/demo-java-app:${VERSION}

# Update Helm values
cd ..
sed -i "s/tag: .*/tag: ${VERSION}/" helm/app/values.yaml

# Commit and push
git add helm/app/values.yaml
git commit -m "Deploy version ${VERSION}"
git push

# Watch ArgoCD sync
kubectl get applications -n argocd

# Watch pods restart
kubectl get pods -n app -w
```

## Verification

After deployment completes:

```bash
# Check image version in pods
kubectl get pods -n app -o jsonpath='{.items[0].spec.containers[0].image}'
# Should show: betechinc/demo-java-app:V1.1

# Access your app
# Your port-forward is already running on 8082
# Open browser to: http://<your-ip>:8082
```

## Summary

**What ArgoCD Does:**
- ✅ Syncs Kubernetes manifests from Git
- ✅ Ensures deployed state matches desired state
- ✅ Provides GitOps workflow

**What ArgoCD Does NOT Do:**
- ❌ Build Docker images
- ❌ Compile source code
- ❌ Watch .java or .html files
- ❌ Run CI pipelines

**For Source Code Changes:**
1. Build new Docker image
2. Use NEW version tag (never reuse same tag)
3. Push to Docker Hub
4. Update values.yaml with new tag
5. Commit and push
6. ArgoCD syncs automatically

**The image tag is the trigger for deployment, not the source code!**
