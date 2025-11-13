# Chapter 8: Debugging & Troubleshooting - When Things Go Wrong

## The Detective Analogy

Imagine you're a detective investigating a crime scene. You don't randomly guess what happened. Instead:

1. **Gather evidence** - Check logs, witness statements, fingerprints
2. **Follow the trail** - Connect the dots from clue to clue
3. **Eliminate suspects** - Rule out what's working fine
4. **Test hypotheses** - Try solutions and verify results

**In Kubernetes debugging:**
- **Evidence = Logs, events, describe output**
- **Trail = Pod → Container → Image → Config → Network**
- **Suspects = Pods, Services, Ingress, DNS, Storage**
- **Hypotheses = Fix attempts and validation**

## The Reality: Things Will Break

**Production reality:**
- Pods crash randomly
- Images fail to pull
- Services can't connect
- Storage fills up
- Configuration is wrong
- Network policies block traffic
- Resources run out

**The question isn't IF things will break, but WHEN.**

This chapter teaches you the systematic approach to debug anything in Kubernetes.

## The Debugging Workflow

```
1. Identify the problem
   ↓
2. Gather information
   ↓
3. Form hypothesis
   ↓
4. Test fix
   ↓
5. Verify solution
   ↓
6. Document learnings
```

## Essential Debugging Commands

Before diving into problems, master these commands:

### 1. Check Pod Status

```bash
# List all pods
kubectl get pods

# All pods with more details
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods -w

# All pods across all namespaces
kubectl get pods -A

# Pods with specific label
kubectl get pods -l app=nginx

# Show resource usage
kubectl top pods
```

### 2. Get Detailed Information

```bash
# Describe pod (shows events)
kubectl describe pod <pod-name>

# Describe deployment
kubectl describe deployment <name>

# Describe service
kubectl describe service <name>

# Describe node
kubectl describe node <node-name>
```

### 3. Check Logs

```bash
# View logs
kubectl logs <pod-name>

# Previous container logs (if crashed)
kubectl logs <pod-name> --previous

# Specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Follow logs (like tail -f)
kubectl logs <pod-name> -f

# Last 50 lines
kubectl logs <pod-name> --tail=50

# Logs from last hour
kubectl logs <pod-name> --since=1h
```

### 4. Interactive Debugging

```bash
# Execute command in pod
kubectl exec <pod-name> -- <command>

# Interactive shell
kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -- bash

# Specific container
kubectl exec -it <pod-name> -c <container-name> -- sh
```

### 5. Check Events

```bash
# Cluster events (recent first)
kubectl get events --sort-by='.lastTimestamp'

# Events for specific namespace
kubectl get events -n <namespace>

# Watch events
kubectl get events -w
```

## Problem 1: Pod Stuck in Pending

**Symptoms:**
```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# my-pod  0/1     Pending   0          5m
```

### Diagnosis

```bash
# Check pod details
kubectl describe pod my-pod

# Look for events at bottom:
# Events:
#   Type     Reason            Message
#   ----     ------            -------
#   Warning  FailedScheduling  0/1 nodes available: insufficient memory
```

### Common Causes & Solutions

#### Cause 1: Insufficient Resources

**Problem:** Node doesn't have enough CPU/memory.

```yaml
# Pod requesting too much
resources:
  requests:
    memory: "64Gi"  # Node only has 16Gi
```

**Solution:**
```bash
# Check node resources
kubectl describe nodes

# Check resource requests
kubectl describe pod my-pod | grep -A 5 "Requests"

# Reduce resource requests or add more nodes
```

#### Cause 2: No Nodes Match Selector

**Problem:** Pod has node selector but no nodes match.

```yaml
spec:
  nodeSelector:
    disktype: ssd  # No nodes have this label
```

**Solution:**
```bash
# Check node labels
kubectl get nodes --show-labels

# Remove selector or label nodes correctly
kubectl label nodes <node-name> disktype=ssd
```

#### Cause 3: PVC Not Bound

**Problem:** Pod waits for PersistentVolumeClaim.

```bash
kubectl get pvc
# NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES
# my-pvc    Pending   ...
```

**Solution:**
```bash
# Check PVC status
kubectl describe pvc my-pvc

# Check if storage class exists
kubectl get storageclass

# In minikube, enable storage provisioner
minikube addons enable storage-provisioner
```

## Problem 2: CrashLoopBackOff

**Symptoms:**
```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# my-pod  0/1     CrashLoopBackOff   5          3m
```

Pod starts, crashes, Kubernetes restarts it, crashes again, repeat.

### Diagnosis

```bash
# Check logs from crashed container
kubectl logs my-pod --previous

# Describe pod for events
kubectl describe pod my-pod
```

### Common Causes & Solutions

#### Cause 1: Application Error

**Problem:** App crashes immediately due to bug or config error.

```bash
# Check logs
kubectl logs my-pod --previous
# Error: Cannot connect to database at localhost:5432
```

**Solution:**
```yaml
# Fix configuration
env:
- name: DB_HOST
  value: "postgres-service"  # Not localhost
- name: DB_PORT
  value: "5432"
```

#### Cause 2: Missing Dependencies

**Problem:** App needs config file or secret that doesn't exist.

```bash
kubectl logs my-pod --previous
# Error: Config file /etc/config/app.conf not found
```

**Solution:**
```yaml
# Create ConfigMap and mount it
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

#### Cause 3: Failed Health Checks

**Problem:** Liveness probe fails, Kubernetes kills container.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5  # Too short!
  periodSeconds: 5
```

**Solution:**
```yaml
# Give app more time to start
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30  # Longer delay
  periodSeconds: 10
```

## Problem 3: ImagePullBackOff

**Symptoms:**
```bash
kubectl get pods
# NAME    READY   STATUS             RESTARTS   AGE
# my-pod  0/1     ImagePullBackOff   0          2m
```

### Diagnosis

```bash
kubectl describe pod my-pod
# Events:
#   Failed to pull image "myregistry.com/myapp:v1.0": 
#   rpc error: code = Unknown desc = Error response from daemon: 
#   pull access denied for myregistry.com/myapp
```

### Common Causes & Solutions

#### Cause 1: Image Doesn't Exist

**Problem:** Typo in image name or tag.

```yaml
image: nginx:1.999  # Version doesn't exist
```

**Solution:**
```bash
# Verify image exists
docker pull nginx:1.25

# Fix image name/tag
image: nginx:1.25
```

#### Cause 2: Private Registry Without Credentials

**Problem:** Image in private registry but no pull secret.

**Solution:**
```bash
# Create pull secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=my@email.com

# Use in pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: myregistry.com/myapp:v1.0
```

#### Cause 3: Network Issues

**Problem:** Can't reach registry (firewall, DNS).

```bash
# Test from node
minikube ssh
curl -I https://registry.hub.docker.com
```

## Problem 4: Service Not Reachable

**Symptoms:**
```bash
# Service exists
kubectl get svc my-service
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# my-service   ClusterIP   10.96.123.45    <none>        80/TCP

# But curl fails
kubectl run test --image=busybox -it --rm -- wget -O- my-service
# wget: can't connect to remote host: Connection refused
```

### Diagnosis

```bash
# Check service endpoints
kubectl get endpoints my-service

# If empty, no pods match selector
# NAME         ENDPOINTS
# my-service   <none>
```

### Common Causes & Solutions

#### Cause 1: Label Mismatch

**Problem:** Service selector doesn't match pod labels.

```yaml
# Service
selector:
  app: web  # Looking for app=web

# Pod
labels:
  app: webapp  # Has app=webapp (mismatch!)
```

**Solution:**
```bash
# Check pod labels
kubectl get pods --show-labels

# Fix service selector or pod labels to match
```

#### Cause 2: Wrong Port

**Problem:** Service targets wrong container port.

```yaml
# Service
ports:
- port: 80
  targetPort: 8080  # Service forwards to port 8080

# Container
ports:
- containerPort: 3000  # App listens on 3000 (mismatch!)
```

**Solution:**
```yaml
# Match targetPort to containerPort
ports:
- port: 80
  targetPort: 3000
```

#### Cause 3: Pods Not Ready

**Problem:** Pods exist but failing readiness probe.

```bash
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# web-pod     0/1     Running   0          5m
#             ↑ Not ready!
```

**Solution:**
```bash
# Check readiness probe
kubectl describe pod web-pod

# Fix probe configuration or app health endpoint
```

## Problem 5: Ingress Not Working

**Symptoms:**
```bash
# Ingress exists
kubectl get ingress
# NAME       HOSTS         ADDRESS   PORTS   AGE
# my-ingress demo.local             80      5m
#                        ↑ No address!
```

### Common Causes & Solutions

#### Cause 1: Ingress Controller Not Installed

```bash
# Check if controller exists
kubectl get pods -n ingress-nginx
# No resources found
```

**Solution:**
```bash
# Install controller
minikube addons enable ingress

# Or for production clusters, install NGINX Ingress
```

#### Cause 2: Wrong Ingress Class

```yaml
spec:
  ingressClassName: nginx  # Looking for 'nginx' class
```

```bash
# Check available classes
kubectl get ingressclass
# NAME    CONTROLLER
# traefik k8s.io/traefik
#         ↑ Only traefik available, not nginx
```

**Solution:**
```yaml
# Use correct class name
ingressClassName: traefik
```

#### Cause 3: Service Name Typo

```yaml
backend:
  service:
    name: my-servce  # Typo: 'servce' instead of 'service'
    port:
      number: 80
```

**Solution:**
```bash
# Verify service name
kubectl get svc

# Fix typo in Ingress
```

## Problem 6: Persistent Data Lost

**Symptoms:**
```bash
# Pod recreated, data gone
kubectl delete pod mysql
kubectl apply -f mysql.yaml
# Database empty!
```

### Diagnosis

```bash
# Check if PVC was used
kubectl get pvc

# Check pod volumes
kubectl describe pod mysql | grep -A 10 "Volumes"
```

### Common Causes & Solutions

#### Cause 1: Using emptyDir

**Problem:** emptyDir is deleted when pod dies.

```yaml
volumes:
- name: data
  emptyDir: {}  # Temporary storage only!
```

**Solution:**
```yaml
# Use PVC for persistence
volumes:
- name: data
  persistentVolumeClaim:
    claimName: mysql-pvc
```

#### Cause 2: PVC Deleted

**Problem:** PVC was accidentally deleted.

```bash
kubectl delete pod mysql
kubectl delete pvc mysql-pvc  # Oops!
```

**Solution:**
```bash
# Be careful with PVC deletion
# Always check before deleting:
kubectl get pvc
```

## Problem 7: High Memory/CPU Usage

**Symptoms:**
```bash
kubectl top pods
# NAME      CPU(cores)   MEMORY(bytes)
# my-pod    1950m        1890Mi
#           ↑ Very high!
```

### Diagnosis

```bash
# Check resource limits
kubectl describe pod my-pod | grep -A 5 "Limits"

# Check node resources
kubectl top nodes
```

### Solutions

#### Solution 1: Add Resource Limits

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

#### Solution 2: Horizontal Pod Autoscaling

```bash
# Auto-scale based on CPU
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10
```

## Debugging Techniques

### Technique 1: Port Forwarding

Test service locally without Ingress.

```bash
# Forward pod port to localhost
kubectl port-forward pod/my-pod 8080:80

# Forward service
kubectl port-forward service/my-service 8080:80

# Test
curl http://localhost:8080
```

### Technique 2: Temporary Debug Pod

Create temporary pod for network debugging.

```bash
# Run interactive busybox
kubectl run debug --image=busybox:1.36 -it --rm -- sh

# Inside pod:
wget -O- http://my-service
nslookup my-service
ping my-service
```

### Technique 3: Copy Files From Pod

```bash
# Copy file from pod to local
kubectl cp my-pod:/var/log/app.log ./app.log

# Copy to pod
kubectl cp ./config.yaml my-pod:/etc/config/
```

### Technique 4: Check DNS

```bash
# Run DNS test
kubectl run dns-test --image=busybox:1.36 -it --rm -- nslookup kubernetes.default

# Should resolve to cluster IP
```

### Technique 5: Network Policy Debugging

```bash
# Check if network policies exist
kubectl get networkpolicies

# Describe policy
kubectl describe networkpolicy <name>

# Temporarily disable by removing policies
kubectl delete networkpolicy --all  # Careful!
```

## The Debugging Checklist

When something breaks, follow this checklist:

### Level 1: Basic Checks
- [ ] Is pod running? `kubectl get pods`
- [ ] Check events: `kubectl describe pod <name>`
- [ ] Check logs: `kubectl logs <name>`
- [ ] Check previous logs: `kubectl logs <name> --previous`

### Level 2: Configuration
- [ ] Labels match? `kubectl get pods --show-labels`
- [ ] Service selector correct? `kubectl describe svc <name>`
- [ ] Endpoints exist? `kubectl get endpoints`
- [ ] ConfigMap/Secret exists? `kubectl get cm,secret`

### Level 3: Resources
- [ ] Resource limits appropriate? `kubectl describe pod <name>`
- [ ] Node has resources? `kubectl top nodes`
- [ ] PVC bound? `kubectl get pvc`
- [ ] Storage class exists? `kubectl get sc`

### Level 4: Network
- [ ] Port-forward works? `kubectl port-forward pod/<name> 8080:80`
- [ ] DNS resolves? `nslookup <service-name>`
- [ ] Network policies blocking? `kubectl get networkpolicy`
- [ ] Ingress controller running? `kubectl get pods -n ingress-nginx`

### Level 5: Deep Dive
- [ ] Check cluster events: `kubectl get events --sort-by='.lastTimestamp'`
- [ ] Check node logs: `kubectl logs <node-pod> -n kube-system`
- [ ] API server issues? `kubectl cluster-info`
- [ ] RBAC permissions? `kubectl auth can-i <verb> <resource>`

## Common Error Messages Decoded

### "CreateContainerConfigError"
**Meaning:** ConfigMap or Secret referenced doesn't exist.

**Fix:** Create missing ConfigMap/Secret or fix reference name.

### "ErrImagePull"
**Meaning:** Can't pull container image.

**Fix:** Check image name, tag, registry credentials.

### "RunContainerError"
**Meaning:** Container started but immediately exited.

**Fix:** Check logs with `kubectl logs <pod> --previous`.

### "Evicted"
**Meaning:** Node ran out of resources, pod was evicted.

**Fix:** Add resource limits, increase node capacity.

### "OOMKilled" (Out Of Memory)
**Meaning:** Container used more memory than limit.

**Fix:** Increase memory limit or optimize application.

### "InvalidImageName"
**Meaning:** Image name format is wrong.

**Fix:** Use format `registry/repo/image:tag`.

## The Challenge

Debug a broken application deployment. You're given broken YAML files. Find and fix ALL issues.

**Scenario:** E-commerce app with frontend, API, and database.

```yaml
# broken-app.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web  # Issue 1: Label mismatch
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:999  # Issue 2: Bad tag
        ports:
        - containerPort: 8080  # Issue 3: Wrong port

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80  # Issue 4: Doesn't match containerPort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: my-api:v1.0
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config  # Issue 5: ConfigMap doesn't exist
              key: db_host
```

**Your task:**
1. Deploy this and identify all failures
2. Fix each issue systematically
3. Verify everything works

**Hints:**
```bash
# Deploy
kubectl apply -f broken-app.yaml

# Start debugging
kubectl get pods
kubectl describe pod <name>
kubectl logs <name>
```

<details>
<summary>Solution (click to expand)</summary>

**Issues found:**

1. **Label mismatch**: Selector uses `app: web`, pod has `app: frontend`
2. **Bad image tag**: `nginx:999` doesn't exist
3. **Wrong containerPort**: nginx listens on 80, not 8080
4. **Port mismatch**: Service targets port 80, container uses 8080 (before fix)
5. **Missing ConfigMap**: `app-config` doesn't exist

**Fixed YAML:**

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  db_host: "database-service"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend  # Fixed: Match pod label
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25  # Fixed: Valid tag
        ports:
        - containerPort: 80  # Fixed: Correct port

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80  # Fixed: Matches containerPort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:1.0  # Fixed: Use existing image for demo
        args:
        - "-text=API Working"
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config  # Fixed: ConfigMap created above
              key: db_host
```

**Verification:**
```bash
kubectl apply -f fixed-app.yaml
kubectl get pods  # All Running
kubectl get svc
kubectl port-forward service/frontend-service 8080:80
curl http://localhost:8080  # Works!
```

</details>

## Key Debugging Tools Summary

| Tool | Use Case |
|------|----------|
| `kubectl get` | Quick status check |
| `kubectl describe` | Detailed info + events |
| `kubectl logs` | Application logs |
| `kubectl exec` | Interactive debugging |
| `kubectl port-forward` | Test connectivity |
| `kubectl top` | Resource usage |
| `kubectl get events` | Cluster-wide events |
| `kubectl cp` | Copy files to/from pod |

## Pro Tips

1. **Always check events first**: `kubectl describe pod <name>` shows most issues
2. **Use `-o wide` for more info**: `kubectl get pods -o wide`
3. **Watch in real-time**: `kubectl get pods -w`
4. **Check previous logs**: `kubectl logs <pod> --previous` for crashed containers
5. **Use labels for filtering**: `kubectl get pods -l app=nginx`
6. **Keep a debug pod**: `kubectl run debug --image=busybox -it --rm -- sh`
7. **Port-forward for quick tests**: Bypass Ingress/Service complexity
8. **Check endpoints**: If service doesn't work, verify endpoints exist

## What You Can Do Now

- Debug pods stuck in Pending, CrashLoopBackOff, ImagePullBackOff
- Troubleshoot service connectivity issues
- Investigate Ingress problems
- Check and interpret logs effectively
- Use exec to debug running containers
- Identify resource constraint issues
- Follow systematic debugging workflow

## What's Next

You can debug production issues. Now learn best practices to prevent them.

**Next**: Chapter 9: Best Practices (Coming Soon)

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate-Advanced  
**Key command**: `kubectl describe pod <name>` - Your first debugging step, always
