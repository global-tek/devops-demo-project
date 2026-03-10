# Quick Reference Guide - Common Issues & Solutions

**Last Updated:** March 9, 2026  
**For:** DevOps Demo Project

---

## 🚨 Emergency Commands

```bash
# Cluster not responding?
minikube status
minikube start

# Out of memory?
free -h
ps aux --sort=-%mem | head -20
# Stop heavy services temporarily
docker stop elasticsearch kibana sonarqube

# Pods crashing?
kubectl get pods -A
kubectl logs <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

# ArgoCD not syncing?
kubectl get applications -n argocd
kubectl -n argocd get application <app-name> -o yaml
# Force sync from CLI
argocd app sync <app-name>
```

---

## 🔍 Issue Identification Matrix

| Symptom | Likely Cause | Quick Check | Solution |
|---------|--------------|-------------|----------|
| `TLS handshake timeout` | Apiserver stopped | `minikube status` | `minikube start` |
| `exec format error` | Architecture mismatch | `docker inspect <image>` | Rebuild for correct arch |
| `ImagePullBackOff` | Image doesn't exist | `docker pull <image>` | Build and push image |
| `CrashLoopBackOff` | App error or config issue | `kubectl logs <pod>` | Check logs, fix code/config |
| `0/X pods available` | Resource limits or errors | `kubectl describe deployment` | Check events and pod status |
| Pipeline commit fails | No changes to commit | Check Git diff | Add conditional check |
| Port 80 conflicts | Another service using it | `kubectl get svc -A` | Use different port |
| Low memory | Too many services | `free -h` | Stop unused services |

---

## 📋 Common Command Patterns

### Kubernetes Basics
```bash
# Get all resources in namespace
kubectl get all -n <namespace>

# Watch pod status in real-time
kubectl get pods -n <namespace> -w

# Get detailed error information
kubectl describe <resource-type> <name> -n <namespace>

# View logs (last 50 lines)
kubectl logs <pod-name> -n <namespace> --tail=50

# Follow logs in real-time
kubectl logs -f <pod-name> -n <namespace>

# Get events (useful for debugging)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Delete and recreate pod
kubectl delete pod <pod-name> -n <namespace>

# Restart deployment
kubectl rollout restart deployment/<name> -n <namespace>
```

### Docker Operations
```bash
# Build for specific architecture
docker build --platform linux/amd64 -t <image>:<tag> .

# Check image architecture
docker inspect <image>:<tag> | grep Architecture

# View images inside minikube
docker exec minikube docker images

# Clean up unused images
docker system prune -a

# Push to registry
docker login
docker push <image>:<tag>

# Pull and test image
docker pull <image>:<tag>
docker run -p 8080:8080 <image>:<tag>
```

### ArgoCD
```bash
# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode && echo

# Port forward to UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# List applications
kubectl get applications -n argocd

# Sync application
kubectl patch application <app-name> -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# View application details
kubectl get application <app-name> -n argocd -o yaml
```

### Minikube
```bash
# Check status
minikube status

# Start/Stop/Restart
minikube start
minikube stop
minikube delete && minikube start

# Access service
minikube service <service-name> -n <namespace>

# Get minikube IP
minikube ip

# SSH into minikube node
minikube ssh

# View dashboard
minikube dashboard
```

### System Monitoring
```bash
# Memory usage
free -h

# Top processes by memory
ps aux --sort=-%mem | head -20

# Top processes by CPU
ps aux --sort=-%cpu | head -20

# Disk usage
df -h

# Container stats
docker stats --no-stream

# Kubernetes resource usage
kubectl top nodes
kubectl top pods -n <namespace>
```

---

## 🏗️ Architecture Verification Checklist

```bash
# 1. Check host architecture
uname -m

# 2. Check Kubernetes node architecture
kubectl get nodes -o wide

# 3. Check minikube architecture
docker exec minikube uname -m

# 4. Check Docker image architecture
docker inspect <image>:<tag> --format='{{.Architecture}}'

# 5. Check image inside minikube
docker exec minikube docker inspect <image>:<tag> --format='{{.Architecture}}'

# ✅ All should match: x86_64 (or all arm64)
```

---

## 🔧 Fix Templates

### Fix ARM64/AMD64 Mismatch

```bash
# Option 1: Build for specific platform
cd /path/to/app
docker build --platform linux/amd64 -t <image>:<tag> .
docker push <image>:<tag>

# Option 2: Multi-platform build
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <image>:<tag> --push .

# Option 3: Use base image with explicit platform
# In Dockerfile:
FROM --platform=linux/amd64 eclipse-temurin:17-jre-alpine
```

### Fix Git Commit in CI/CD

```bash
# In Jenkinsfile or CI script:
git add <files>
if git diff --staged --quiet; then
    echo "No changes to commit"
else
    git commit -m "Update message"
    git push
fi
```

### Fix Service Port Conflicts

```yaml
# In service.yaml:
spec:
  type: NodePort
  ports:
  - name: http
    port: 8082          # External port (what clients use)
    targetPort: 8080    # Container port (where app listens)
    protocol: TCP
```

### Fix Memory Issues

```bash
# Immediate: Stop heavy services
docker stop elasticsearch
docker stop kibana
docker stop sonarqube

# Short-term: Add swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Long-term: Reduce heap sizes
# Edit Elasticsearch config:
-Xms2048m -Xmx2048m  # Instead of 7894m
```

---

## 📝 Pre-Deployment Checklist

Before deploying an application:

- [ ] ✅ Image exists in registry (`docker pull <image>`)
- [ ] ✅ Image architecture matches cluster (`docker inspect`)
- [ ] ✅ Namespace exists (`kubectl get ns`)
- [ ] ✅ Secrets/ConfigMaps created if needed
- [ ] ✅ Service ports don't conflict (`kubectl get svc -A`)
- [ ] ✅ Resource limits defined in deployment
- [ ] ✅ Health checks configured (readiness/liveness probes)
- [ ] ✅ Sufficient cluster resources (`kubectl top nodes`)
- [ ] ✅ Image pull secrets configured if private registry

---

## 🎯 Port Management

**Reserved Ports:**
- 8080: ArgoCD ApplicationSet Controller
- 8081: ArgoCD Repo Server
- 8082: Demo Java App (configured)
- 8083: ArgoCD Server Metrics
- 9000: SonarQube
- 5601: Kibana
- 9200: Elasticsearch
- 3000: Grafana
- 9090: Prometheus

**Port Forward Background Process:**
```bash
# Start in background
nohup kubectl port-forward svc/<service> <local>:<remote> \
  -n <namespace> --address 0.0.0.0 > pf.log 2>&1 &

# Check running port forwards
ps aux | grep port-forward

# Kill specific port forward
pkill -f "port-forward.*<service>"

# Kill all port forwards
pkill -f port-forward
```

---

## 🐛 Debug Workflows

### Pod Won't Start
```
1. kubectl get pods -n <namespace>
2. kubectl describe pod <pod-name> -n <namespace>
3. Check Events section for errors
4. kubectl logs <pod-name> -n <namespace>
5. Common issues:
   - ImagePullBackOff → Check image name/tag
   - CrashLoopBackOff → Check application logs
   - Pending → Check resource availability
```

### Service Not Accessible
```
1. kubectl get svc -n <namespace>
2. kubectl get endpoints -n <namespace>
3. Check if pods are ready:
   kubectl get pods -n <namespace>
4. Test from within cluster:
   kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
     curl http://<service-name>.<namespace>:8080
5. Check service selector matches pod labels
```

### ArgoCD App Degraded
```
1. kubectl get application <app-name> -n argocd
2. Check sync status:
   kubectl describe application <app-name> -n argocd
3. View in UI or:
   argocd app get <app-name>
4. Force sync:
   argocd app sync <app-name> --prune
5. Check target resources exist and are healthy
```

---

## 💡 Pro Tips

1. **Always check logs first:** `kubectl logs` reveals most issues
2. **Events are your friend:** `kubectl get events --sort-by='.lastTimestamp'`
3. **Use labels effectively:** Makes filtering and debugging easier
4. **Keep images small:** Faster pulls, less storage, fewer vulnerabilities
5. **Tag images properly:** Never use `latest` in production
6. **Monitor resource usage:** Prevent OOM kills and CPU throttling
7. **Test locally first:** `docker run` before `kubectl apply`
8. **Use health checks:** Kubernetes can auto-restart unhealthy pods
9. **Version your configs:** GitOps makes rollbacks easy
10. **Document as you go:** Future you will thank present you

---

## 🔗 Related Documentation

- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Detailed issue resolution guide
- [WORKFLOW.md](WORKFLOW.md) - Visual workflows and decision trees
- [README.md](README.md) - Project overview and setup

---

## 📞 When All Else Fails

```bash
# Nuclear option: Reset everything
minikube delete
minikube start --memory=8192 --cpus=4

# Redeploy ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Check cluster health
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

---

**Remember:** 
- Read error messages carefully
- Check one thing at a time
- Document what you try
- Ask for help when stuck

*Quick Reference Card - Keep this handy!*
