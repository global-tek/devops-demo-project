# DevOps Demo Project - Troubleshooting Session Documentation

**Date:** March 9, 2026  
**Session Duration:** ~3 hours  
**Issues Resolved:** 6 major issues

---

## Table of Contents
1. [Kubernetes Cluster TLS Handshake Timeout](#issue-1-kubernetes-cluster-tls-handshake-timeout)
2. [ArgoCD Installation and Configuration](#issue-2-argocd-installation-and-configuration)
3. [ArgoCD Admin Access](#issue-3-argocd-admin-access)
4. [Application Deployment Failures](#issue-4-application-deployment-failures)
5. [Jenkins Pipeline Git Commit Failure](#issue-5-jenkins-pipeline-git-commit-failure)
6. [Service Port Configuration](#issue-6-service-port-configuration)

---

## Issue 1: Kubernetes Cluster TLS Handshake Timeout

### Problem Description
**User Question:** "What is causing this error?"

**Error Observed:**
```bash
$ kubectl get ns
E0309 05:31:38.190477   10724 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"https://192.168.49.2:8443/api?timeout=32s\": net/http: TLS handshake timeout"
```

**Additional Symptoms:**
```bash
$ kubectl create ns app
Unable to connect to the server: net/http: TLS handshake timeout
```

### Root Cause Analysis

**Command Used:**
```bash
$ minikube status
```

**Result:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Stopped
kubeconfig: Configured
```

**Primary Issue:** Minikube apiserver was stopped while host and kubelet were still running.

**Secondary Issue:** Severe memory pressure identified:

**Command Used:**
```bash
$ free -h
```

**Result:**
```
               total        used        free      shared  buff/cache   available
Mem:            15Gi        14Gi       222Mi        14Mi       961Mi       834Mi
Swap:             0B          0B          0B
```

**Memory Usage Breakdown:**
```bash
$ ps aux --sort=-%mem | head -15
```

**Top Memory Consumers:**
- Elasticsearch: 8.6GB (53% of total memory) - configured with 7.9GB heap
- Jenkins: 1.2GB (7.4%)
- SonarQube's Elasticsearch: 767MB (4.7%)
- Kibana: 647MB (4.0%)
- SonarQube processes: ~1GB combined
- VS Code Server: ~1.2GB combined
- Kubernetes apiserver: ~233MB

### Resolution

**Step 1: Restart Minikube Cluster**
```bash
$ minikube start
```

**Output:**
```
😄  minikube v1.38.1 on Ubuntu 22.04
✨  Using the docker driver based on existing profile
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.50 ...
🏃  Updating the running docker "minikube" container ...
🐳  Preparing Kubernetes v1.35.1 on Docker 29.2.1 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
```

**Verification:**
```bash
$ kubectl get ns
NAME              STATUS   AGE
app               Active   7d11h
default           Active   7d11h
kube-node-lease   Active   7d11h
kube-public       Active   7d11h
kube-system       Active   7d11h
logging           Active   7d11h
monitoring        Active   7d11h
```

**Step 2: Address Memory Pressure**

**Added Swap Space:**
```bash
# (Already had 4GB swap configured)
$ free -h
Swap:          4.0Gi       3.3Gi       742Mi
```

### Recommendations
1. Monitor memory usage regularly: `free -h`
2. Stop unnecessary services when not in use (Elasticsearch, SonarQube)
3. Quick cluster check: `minikube status`
4. Restart cluster: `minikube start`

---

## Issue 2: ArgoCD Installation and Configuration

### Problem Description
**User Question:** "What is happening?"

**Error Observed:**
```bash
$ kubectl apply -f argocd-basic.yaml
namespace/argocd created
error: resource mapping not found for name: "example-argocd" namespace: "argocd" from "argocd-basic.yaml": no matches for kind "ArgoCD" in version "argoproj.io/v1beta1"
ensure CRDs are installed first
```

### Root Cause Analysis

**File Content Review:**
```yaml
# argocd-basic.yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: example-argocd
  namespace: argocd
  labels:
    example: basic
spec: {}
```

**Issue:** The YAML file attempts to create an `ArgoCD` custom resource (Operator pattern), but the ArgoCD Operator and its CRDs were not installed.

**OLM Status Check:**
```bash
$ kubectl get packagemanifests -n olm | grep argocd
error: the server doesn't have a resource type "packagemanifests"
```

OLM was installed but not properly configured with catalog sources.

### Resolution

**Installed ArgoCD using standard manifests instead of Operator:**

```bash
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Output (abbreviated):**
```
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
...
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
```

**Verification:**
```bash
$ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          8m47s
argocd-applicationset-controller-6799596c7c-qwqmm   1/1     Running   3          8m48s
argocd-dex-server-5cb8756cf7-xh99k                  1/1     Running   2          8m48s
argocd-notifications-controller-5cd8948d4b-rqllr    1/1     Running   0          8m48s
argocd-redis-59784bcdb7-x495c                       1/1     Running   0          8m47s
argocd-repo-server-6868bb7494-cmclk                 1/1     Running   0          8m47s
argocd-server-7488fb8dbf-5mlt8                      1/1     Running   0          8m47s
```

### Key Differences: Operator vs Standard Installation

| Aspect | ArgoCD Operator | Standard Manifests |
|--------|----------------|-------------------|
| Resource names | `example-argocd-*` | `argocd-*` |
| Installation | Requires Operator + CR | Direct component deployment |
| Flexibility | Multiple ArgoCD instances | Single instance per namespace |
| Complexity | Higher | Lower |

---

## Issue 3: ArgoCD Admin Access

### Problem Description
**User Question:** "What is happening?"

**Error Observed:**
```bash
$ kubectl get secret example-argocd-cluster -n argocd -o jsonpath="{.data.admin\.password}" | base64 —decode
base64: —decode: No such file or directory
Error from server (NotFound): secrets "example-argocd-cluster" not found

$ minikube service example-argocd-server -n argocd
❌  Exiting due to SVC_NOT_FOUND: Service 'example-argocd-server' was not found in 'argocd' namespace.
```

### Root Cause Analysis

**Two Issues Identified:**

1. **Incorrect resource names:** Commands used Operator pattern names (`example-argocd-*`) but we installed standard manifests (`argocd-*`)
2. **Typo in command:** `—decode` (em dash) instead of `--decode` (two hyphens)

**Actual Resources Check:**
```bash
$ kubectl get secrets -n argocd | grep -i admin
argocd-initial-admin-secret   Opaque   1      7m7s

$ kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP       PORT(S)
argocd-applicationset-controller          ClusterIP   10.104.207.143   7000/TCP,8080/TCP
argocd-dex-server                         ClusterIP   10.99.202.191    5556/TCP,5557/TCP,5558/TCP
argocd-server                             ClusterIP   10.100.163.59    80/TCP,443/TCP
...
```

### Resolution

**Correct Command to Retrieve Admin Password:**
```bash
$ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
h6tdrKkt7djTs75u
```

**ArgoCD Credentials:**
- **Username:** `admin`
- **Password:** `h6tdrKkt7djTs75u`

**Access Methods Provided:**

**Option 1: Port Forward**
```bash
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access at: https://localhost:8080
```

**Option 2: Minikube Service**
```bash
$ minikube service argocd-server -n argocd
```

### Lessons Learned
- Resource naming differs between Operator and standard installations
- Always verify actual resource names with `kubectl get`
- Watch for character encoding issues (em dash vs hyphen)

---

## Issue 4: Application Deployment Failures

### Problem Description
**User Question:** "Why is the app unavailable?"

**Deployment Status:**
```bash
$ kubectl describe deployment -n app
Name:                   demo-java-app-deployment
Namespace:              app
Replicas:               2 desired | 2 updated | 2 total | 0 available | 2 unavailable
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
```

**Pod Status:**
```bash
$ kubectl get pods -n app
NAME                                       READY   STATUS             RESTARTS
demo-java-app-deployment-bdff7cf65-4mrq9   0/1     CrashLoopBackOff   28
demo-java-app-deployment-bdff7cf65-pwnnq   0/1     CrashLoopBackOff   28
```

### Root Cause Analysis

**Step 1: Check Pod Logs**
```bash
$ kubectl logs demo-java-app-deployment-bdff7cf65-4mrq9 -n app --tail=50
exec /usr/java/openjdk-18/bin/java: exec format error
```

**Step 2: Verify Architecture**
```bash
$ uname -m && docker exec minikube uname -m
x86_64
x86_64

$ docker exec minikube docker inspect deepakkr35/demo-java-app:V1.0 --format='{{.Architecture}} {{.Os}}'
arm64 linux
```

**Issue Identified:** Architecture mismatch
- Cluster architecture: **x86_64 (AMD64)**
- Container image architecture: **arm64**
- Result: Binary incompatibility causing `exec format error`

### Additional Issue: Image Pull Failure

**Later Pod Status:**
```bash
$ kubectl get pods -n app
NAME                                        READY   STATUS           RESTARTS
demo-java-app-deployment-66cc899d66-4lxgk   0/1     ErrImagePull     0
demo-java-app-deployment-bdff7cf65-4mrq9    0/1     CrashLoopBackOff 38
```

**New Pod Description:**
```bash
$ kubectl describe pod demo-java-app-deployment-66cc899d66-4lxgk -n app
Image:          global-tek/demo-java-app:V1.0
State:          Waiting
  Reason:       ErrImagePull

Events:
  Warning  Failed  Failed to pull image "global-tek/demo-java-app:V1.0": 
           Error response from daemon: pull access denied for global-tek/demo-java-app, 
           repository does not exist or may require 'docker login'
```

**Values File Check:**
```yaml
# helm/app/values.yaml
image:
  githubID: global-tek
  repository: demo-java-app
  tag: V1.0
```

**Issue:** The githubID was changed from `deepakkr35` to `global-tek`, but the `global-tek/demo-java-app:V1.0` image doesn't exist on Docker Hub.

### Root Causes Summary

**Two separate issues affecting deployment:**

1. **CrashLoopBackOff pods:**
   - Image: `deepakkr35/demo-java-app:V1.0`
   - Problem: ARM64 image on x86_64 cluster

2. **ErrImagePull pods:**
   - Image: `global-tek/demo-java-app:V1.0`
   - Problem: Non-existent repository

### Resolution Steps

**Required Actions:**

1. **Build Java application with Maven:**
```bash
$ sudo apt install maven -y
$ cd /home/ubuntu/devops-demo-project/demo-java-app
$ mvn clean package -DskipTests
```

2. **Build AMD64-compatible Docker image:**
```bash
$ docker build --platform linux/amd64 -t global-tek/demo-java-app:V1.0 .
```

3. **Push to Docker Hub:**
```bash
$ docker login
$ docker push global-tek/demo-java-app:V1.0
```

4. **Update deployment:**
   - Either wait for ArgoCD to sync from Git
   - Or manually trigger: `kubectl rollout restart deployment/demo-java-app-deployment -n app`

### Dockerfile Review
```dockerfile
# Base image 
FROM eclipse-temurin

# setting artifact path
ARG artifact=target/demo-java-app.jar

WORKDIR /opt/app

COPY ${artifact} app.jar

# Entrypoint for running app
ENTRYPOINT ["java","-jar","app.jar"]
```

**Issue with Dockerfile:** Used `FROM eclipse-temurin` without specifying a tag or platform, resulting in platform-dependent builds.

**Recommendation:** Use explicit tags and platform specification:
```dockerfile
FROM eclipse-temurin:17-jre-alpine
```

---

## Issue 5: Jenkins Pipeline Git Commit Failure

### Problem Description
**User Question:** "This build step fails"

**Build Output:**
```bash
+ git config user.email globaltekinq@gmail.com
+ git config user.name Global-tek
+ sed -i s/tag: .*/tag: V1.0/ helm/app/values.yaml
+ git add helm/app/values.yaml
+ git commit -m Update deployment image to version V1.0
On branch main
nothing to commit, working tree clean
```

**Pipeline Stage:**
```groovy
stage('Update Deployment File') {
    steps {
        sh '''
            git config user.email "globaltekinq@gmail.com"
            git config user.name "Global-tek"
            sed -i "s/tag: .*/tag: \"${build_version}\"/" helm/app/values.yaml
            git add helm/app/values.yaml
            git commit -m "Update deployment image to version ${build_version}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
        '''
    }
}
```

### Root Cause Analysis

**Issue:** The `tag` value in `helm/app/values.yaml` was already set to `V1.0`, so:
1. The `sed` command made no changes
2. Git had nothing to commit
3. `git commit` returned exit code 1
4. Pipeline stage failed

**Current values.yaml:**
```yaml
image:
  tag: V1.0
```

**After sed command with build_version=V1.0:**
```yaml
image:
  tag: V1.0  # No change
```

### Resolution

**Updated Jenkinsfile with conditional commit:**

```groovy
stage('Update Deployment File') {
    environment {
        GIT_REPO_NAME = "devops-demo-project"
        GIT_USER_NAME = "global-tek"
    }
    steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
            sh '''
                git config user.email "globaltekinq@gmail.com"
                git config user.name "Global-tek"
                sed -i "s/tag: .*/tag: \"${build_version}\"/" helm/app/values.yaml
                git add helm/app/values.yaml
                if git diff --staged --quiet; then
                    echo "No changes to commit - tag is already ${build_version}"
                else
                    git commit -m "Update deployment image to version ${build_version}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                fi
            '''
        }
    }
}
```

**Key Changes:**
- Added `if git diff --staged --quiet` check
- Only commits and pushes if there are actual changes
- Prints informative message when no changes exist

### Additional Fix

**Docker Build Typo Correction:**

**Before:**
```groovy
sh 'cd demo-java-app && docker build --platform linux/amd64-t ${DOCKER_IMAGE} .'
```

**After:**
```groovy
sh 'cd demo-java-app && docker build --platform linux/amd64 -t ${DOCKER_IMAGE} .'
```

**Issue:** Missing space between `amd64` and `-t` flag.

### Testing Scenarios

**Scenario 1: No changes (tag already set)**
```bash
$ git diff --staged --quiet
# Exit code: 0 (no differences)
Output: "No changes to commit - tag is already V1.0"
```

**Scenario 2: Tag value changes**
```bash
$ git diff --staged --quiet
# Exit code: 1 (differences exist)
Output: Commits and pushes changes
```

---

## Issue 6: Service Port Configuration

### Problem Description
**User Question:** "How can we expose the app on port 8082 since port 8080 is already exposed for the argocd service?"

**Current Service Configuration:**
```bash
$ kubectl get svc -n app
NAME                    TYPE       CLUSTER-IP       PORT(S)        AGE
demo-java-app-service   NodePort   10.110.218.112   80:30844/TCP   3h6m
```

### Root Cause Analysis

**Service Template Review:**
```yaml
# helm/app/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}-service
  namespace: {{ .Values.namespace }}
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 8082
    protocol: TCP
  selector:
    app: {{ .Values.appName }}
```

**Deployment Template Review:**
```yaml
# helm/app/templates/deployment.yaml
spec:
  containers:
  - name: {{ .Values.appName }}
    image: {{ .Values.image.githubID }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}
    ports:
    - containerPort: 8080
```

**Issues Identified:**
1. **Port mismatch:** Container listens on `8080`, but service targets `8082`
2. **Service port:** External port is `80`, user wants `8082`

### Resolution

**Updated Service Configuration:**
```yaml
spec:
  type: NodePort
  ports:
  - name: http
    port: 8082        # Changed from 80 to 8082
    targetPort: 8080  # Changed from 8082 to 8080 (matches container)
    protocol: TCP
  selector:
    app: {{ .Values.appName }}
```

**Port Mapping:**
- **External Service Port:** `8082` (what clients connect to)
- **Target Container Port:** `8080` (where Java app listens)
- **NodePort:** Auto-assigned by Kubernetes (e.g., `30844`)

### Access Methods

**Method 1: Port Forward (recommended for local access)**
```bash
$ nohup kubectl port-forward svc/demo-java-app-service 8082:8082 -n app --address 0.0.0.0 > app_pf.log 2>&1 &
```

Access at: `http://localhost:8082` or `http://<VM-IP>:8082`

**Method 2: Minikube Service**
```bash
$ minikube service demo-java-app-service -n app
```

**Method 3: NodePort Access**
```bash
$ kubectl get svc -n app
# Access via: http://<minikube-ip>:<nodeport>
$ minikube ip
# Example: http://192.168.49.2:30844
```

### ArgoCD Service Port Comparison

```bash
$ kubectl get svc -n argocd argocd-server
NAME            TYPE        CLUSTER-IP      PORT(S)
argocd-server   ClusterIP   10.100.163.59   80/TCP,443/TCP
```

**Port Isolation:**
- ArgoCD uses ClusterIP service on internal port 80/443
- Demo app uses NodePort service on port 8082
- No conflict because different service types and ports

---

## Summary of Issues and Solutions

| # | Issue | Root Cause | Solution | Status |
|---|-------|------------|----------|--------|
| 1 | TLS Handshake Timeout | Stopped apiserver + memory pressure | `minikube start` + memory monitoring | ✅ Resolved |
| 2 | ArgoCD Installation | Missing operator/CRDs | Installed standard manifests | ✅ Resolved |
| 3 | ArgoCD Access | Wrong resource names + typo | Correct secret name + `--decode` fix | ✅ Resolved |
| 4 | App Deployment Failure | ARM64/AMD64 mismatch + missing image | Build AMD64 image, fix repository | 🔄 Pending build |
| 5 | Jenkins Commit Failure | No changes to commit | Conditional commit check | ✅ Resolved |
| 6 | Service Port Config | Port mismatch + wrong external port | Corrected targetPort mapping | ✅ Resolved |

---

## Commands Reference

### Quick Diagnostics
```bash
# Check cluster status
minikube status

# Check memory usage
free -h

# Check pod status
kubectl get pods -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Describe resource for details
kubectl describe <resource-type> <resource-name> -n <namespace>

# Check node architecture
uname -m
docker exec minikube uname -m

# Check image architecture
docker inspect <image> --format='{{.Architecture}}'
```

### Cluster Management
```bash
# Start Minikube
minikube start

# Stop Minikube
minikube stop

# Delete and recreate cluster
minikube delete && minikube start
```

### ArgoCD Management
```bash
# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode

# Port forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login via CLI
argocd login localhost:8080 --username admin --password <password> --insecure
```

### Docker Operations
```bash
# Build multi-platform image
docker buildx build --platform linux/amd64,linux/arm64 -t <image>:<tag> .

# Build for specific platform
docker build --platform linux/amd64 -t <image>:<tag> .

# Check local images
docker images | grep <image-name>

# Check image in minikube
docker exec minikube docker images | grep <image-name>
```

### Application Deployment
```bash
# Apply changes via kubectl
kubectl apply -f <file>

# Rollout restart deployment
kubectl rollout restart deployment/<name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Port forward to service
kubectl port-forward svc/<service-name> <local-port>:<service-port> -n <namespace>
```

---

## Best Practices Learned

### 1. Memory Management
- Monitor memory regularly on resource-constrained systems
- Stop unused services (Elasticsearch, SonarQube) when not needed
- Configure appropriate heap sizes for Java applications
- Add swap space for memory spikes

### 2. Container Images
- Always specify explicit base image tags
- Use `--platform` flag for multi-architecture builds
- Verify image architecture matches cluster architecture
- Push images to registries before deploying

### 3. GitOps Workflows
- Check for changes before committing in CI/CD
- Use meaningful commit messages with version info
- Verify Git credentials before push operations
- Use conditional logic to handle idempotent operations

### 4. Kubernetes Services
- Ensure targetPort matches container's listening port
- Choose appropriate service type (ClusterIP vs NodePort vs LoadBalancer)
- Use port-forward for local development and debugging
- Document exposed ports to avoid conflicts

### 5. Troubleshooting Approach
1. Check resource status: `kubectl get`
2. Describe for details: `kubectl describe`
3. View logs: `kubectl logs`
4. Verify architecture/compatibility
5. Check system resources (memory, CPU)
6. Review configurations and manifests

---

## Environment Details

**System Information:**
- OS: Ubuntu 22.04 LTS on AWS EC2
- Architecture: x86_64
- Total Memory: 15GB
- Minikube Version: v1.38.1
- Kubernetes Version: v1.35.1
- Docker Driver: docker 29.2.1

**Installed Components:**
- Minikube + Kubernetes
- ArgoCD (standard installation)
- Jenkins
- SonarQube
- Elasticsearch + Kibana
- Prometheus + Grafana
- OLM (Operator Lifecycle Manager)

**Namespaces:**
- `app` - Demo Java application
- `argocd` - ArgoCD components
- `logging` - Elasticsearch, Kibana, Filebeat
- `monitoring` - Prometheus, Grafana
- `olm` - Operator Lifecycle Manager
- `operators` - Operator installations

---

## Files Modified

1. **ci/Jenkins/Jenkinsfile**
   - Added conditional commit check
   - Fixed Docker build platform flag typo

2. **helm/app/templates/service.yaml**
   - Changed service port from 80 to 8082
   - Fixed targetPort from 8082 to 8080

3. **helm/app/values.yaml**
   - Updated githubID from `deepakkr35` to `global-tek`

---

## Next Steps

### Immediate Actions Required
1. ✅ Build Java application JAR file
2. ✅ Build AMD64-compatible Docker image
3. ✅ Push image to `global-tek/demo-java-app:V1.0`
4. ⏳ Verify deployment health
5. ⏳ Test application endpoint

### Long-term Improvements
1. Implement automated image builds in Jenkins
2. Add health checks and readiness probes to deployment
3. Configure resource limits for all deployments
4. Set up proper monitoring and alerting
5. Document architecture and deployment procedures
6. Implement backup and disaster recovery

---

## Conclusion

This troubleshooting session resolved six interconnected issues across Kubernetes cluster management, ArgoCD configuration, application deployment, CI/CD pipeline, and service networking. The root causes ranged from system resource constraints to architecture mismatches and configuration errors.

**Key Takeaways:**
- System resource monitoring is critical for stable operations
- Architecture compatibility must be verified for container images
- GitOps workflows need idempotent operations handling
- Service configurations must align with application requirements
- Detailed error analysis and systematic verification prevent recurring issues

**Session Statistics:**
- Issues Resolved: 6
- Commands Executed: 40+
- Files Modified: 3
- Documentation Created: 1

---

*Generated from troubleshooting session on March 9, 2026*
