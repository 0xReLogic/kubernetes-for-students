# Chapter 9: Best Practices - Production-Ready Kubernetes

## The Restaurant Kitchen Analogy

A successful restaurant kitchen isn't just about cooking food. It's about:

1. **Consistency** - Same dish tastes the same every time
2. **Hygiene** - Clean environment, safe food handling
3. **Efficiency** - No wasted ingredients, optimized workflow
4. **Scalability** - Handle rush hour without chaos
5. **Documentation** - Recipes anyone can follow

**In Kubernetes production:**
- **Consistency** = Reproducible deployments
- **Hygiene** = Security best practices
- **Efficiency** = Resource optimization
- **Scalability** = Auto-scaling and high availability
- **Documentation** = Clear YAML, good naming, labels

## Why Best Practices Matter

**Without best practices:**
```
"It works on my machine"
"Who deployed this?"
"Why is production down?"
"We got hacked"
"This costs $50,000/month in cloud bills"
```

**With best practices:**
```
"Deployment is automated and reproducible"
"Everything is tracked and auditable"
"Auto-recovery kicked in, no downtime"
"Security policies blocked the attack"
"We optimized and saved 60% on cloud costs"
```

## Category 1: Resource Management

### Always Set Resource Requests and Limits

**Bad:**
```yaml
containers:
- name: app
  image: myapp:v1
  # No resources defined - dangerous!
```

**Good:**
```yaml
containers:
- name: app
  image: myapp:v1
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

**Why it matters:**
- **Requests** - Scheduler uses this to place pods
- **Limits** - Prevents runaway containers from killing nodes
- Without limits, one container can consume all node resources

### Resource Guidelines

| App Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|----------|-------------|-----------|----------------|--------------|
| Small API | 100m | 200m | 128Mi | 256Mi |
| Medium Web | 250m | 500m | 256Mi | 512Mi |
| Large App | 500m | 1000m | 512Mi | 1Gi |
| Database | 500m | 2000m | 1Gi | 4Gi |

**Pro tip:** Start low, monitor with `kubectl top pods`, then adjust.

### Use Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa
```

## Category 2: High Availability

### Always Use Multiple Replicas

**Bad:**
```yaml
spec:
  replicas: 1  # Single point of failure
```

**Good:**
```yaml
spec:
  replicas: 3  # Survives failures
```

### Use Pod Disruption Budgets (PDB)

Ensure minimum pods available during maintenance.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # At least 2 pods must stay running
  selector:
    matchLabels:
      app: my-app
```

**Or use maxUnavailable:**
```yaml
spec:
  maxUnavailable: 1  # Only 1 pod can be down at a time
```

### Use Pod Anti-Affinity

Spread pods across nodes to survive node failures.

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

**Result:** Pods scheduled on different nodes when possible.

### Use Topology Spread Constraints

More control over pod distribution.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

**Result:** Pods spread evenly across availability zones.

## Category 3: Health Checks

### Always Configure Probes

**Liveness Probe** - Restart if unhealthy

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

**Readiness Probe** - Only receive traffic when ready

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

**Startup Probe** - For slow-starting apps

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 30  # 30 * 10 = 5 minutes to start
```

### Probe Best Practices

| Probe | Purpose | Typical Settings |
|-------|---------|------------------|
| Liveness | Detect deadlocks | initialDelay: 30s, period: 10s |
| Readiness | Traffic control | initialDelay: 5s, period: 5s |
| Startup | Slow apps | failureThreshold: 30 |

**Common mistake:** Setting `initialDelaySeconds` too short - app gets killed before it starts.

## Category 4: Security

### Never Run as Root

**Bad:**
```yaml
containers:
- name: app
  image: myapp:v1
  # Runs as root by default
```

**Good:**
```yaml
containers:
- name: app
  image: myapp:v1
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
```

### Drop All Capabilities

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
```

### Read-Only Root Filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**Full security context example:**

```yaml
containers:
- name: app
  image: myapp:v1
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
```

### Use Network Policies

Restrict pod-to-pod communication.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

**Result:** API only accepts traffic from frontend, only talks to database.

### Use Secrets Properly

**Never do this:**
```yaml
env:
- name: DB_PASSWORD
  value: "supersecret123"  # Exposed in YAML!
```

**Do this:**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

**Even better - use external secret managers:**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

## Category 5: Deployment Strategies

### Use Rolling Updates (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Extra pods during update
      maxUnavailable: 25%  # Pods that can be down
```

### Configure Proper Termination

```yaml
spec:
  terminationGracePeriodSeconds: 60  # Time for graceful shutdown
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
```

**Why:** Allows in-flight requests to complete before pod terminates.

### Use Image Tags Properly

**Bad:**
```yaml
image: myapp:latest  # Never use in production!
```

**Good:**
```yaml
image: myapp:v1.2.3  # Specific version
```

**Even better:**
```yaml
image: myapp@sha256:abc123...  # Immutable digest
```

### Always Set imagePullPolicy

```yaml
containers:
- name: app
  image: myapp:v1.2.3
  imagePullPolicy: IfNotPresent  # Or Always for latest
```

## Category 6: Labels and Organization

### Use Consistent Labels

**Recommended labels (Kubernetes standard):**

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/instance: my-app-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: e-commerce
    app.kubernetes.io/managed-by: helm
```

**Minimum recommended:**

```yaml
metadata:
  labels:
    app: my-app
    env: production
    version: v1.2.3
    team: backend
```

### Use Namespaces

Separate environments and teams.

```bash
# Create namespaces
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Deploy to specific namespace
kubectl apply -f deployment.yaml -n production
```

**Namespace naming convention:**
```
development
staging
production
team-backend
team-frontend
project-ecommerce
```

### Use Annotations for Metadata

```yaml
metadata:
  annotations:
    description: "Main API server for e-commerce platform"
    owner: "backend-team@company.com"
    documentation: "https://wiki.company.com/api"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

## Category 7: Configuration Management

### Externalize All Configuration

**Bad:**
```yaml
containers:
- name: app
  env:
  - name: LOG_LEVEL
    value: "debug"  # Hardcoded
```

**Good:**
```yaml
containers:
- name: app
  envFrom:
  - configMapRef:
      name: app-config
```

### Separate Configs by Environment

```
configs/
├── base/
│   └── configmap.yaml
├── development/
│   └── configmap.yaml
├── staging/
│   └── configmap.yaml
└── production/
    └── configmap.yaml
```

### Use Kustomize for Environment Management

**base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
```

**production/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
patchesStrategicMerge:
- replica-patch.yaml
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=info
  - ENV=production
```

```bash
# Deploy to production
kubectl apply -k production/
```

## Category 8: Monitoring and Observability

### Expose Metrics Endpoint

```yaml
containers:
- name: app
  ports:
  - containerPort: 8080
    name: http
  - containerPort: 9090
    name: metrics
```

### Add Prometheus Annotations

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

### Structured Logging

**Bad:**
```
Error occurred while processing request
```

**Good (JSON):**
```json
{"level":"error","timestamp":"2024-01-15T10:30:00Z","message":"Error processing request","request_id":"abc123","user_id":"user456","error":"connection timeout"}
```

### Key Metrics to Monitor

| Metric | What It Tells You |
|--------|-------------------|
| CPU usage | Processing load |
| Memory usage | Memory leaks |
| Request latency | User experience |
| Error rate | Application health |
| Pod restarts | Stability issues |
| Pending pods | Resource constraints |

## Category 9: Cost Optimization

### Right-Size Resources

```bash
# Check actual usage
kubectl top pods

# Compare with requests
kubectl describe pod <name> | grep -A 5 "Requests"

# Adjust accordingly
```

### Use Spot/Preemptible Instances

For non-critical workloads:

```yaml
spec:
  nodeSelector:
    cloud.google.com/gke-spot: "true"
  tolerations:
  - key: cloud.google.com/gke-spot
    operator: Equal
    value: "true"
    effect: NoSchedule
```

### Set Resource Quotas

Prevent runaway costs per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
```

### Use Limit Ranges

Set defaults for containers:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - default:
      memory: "512Mi"
      cpu: "500m"
    defaultRequest:
      memory: "256Mi"
      cpu: "250m"
    type: Container
```

## The Production Checklist

Before deploying to production, verify:

### Resources
- [ ] Resource requests set
- [ ] Resource limits set
- [ ] HPA configured for scaling

### Availability
- [ ] Multiple replicas (minimum 2)
- [ ] Pod Disruption Budget defined
- [ ] Anti-affinity rules configured

### Health
- [ ] Liveness probe configured
- [ ] Readiness probe configured
- [ ] Startup probe for slow apps

### Security
- [ ] Running as non-root
- [ ] Read-only root filesystem
- [ ] Capabilities dropped
- [ ] Network policies defined
- [ ] Secrets stored properly

### Deployment
- [ ] Specific image tags (not :latest)
- [ ] Rolling update strategy
- [ ] Graceful termination configured

### Organization
- [ ] Proper labels applied
- [ ] Namespace separation
- [ ] Annotations for documentation

### Observability
- [ ] Metrics endpoint exposed
- [ ] Prometheus annotations
- [ ] Structured logging

## Example: Production-Ready Deployment

Here's a complete example following all best practices:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: e-commerce
  annotations:
    description: "E-commerce API server"
    owner: "backend-team@company.com"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      terminationGracePeriodSeconds: 60
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api-server
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: api
        image: myregistry.com/api-server:v1.2.3
        imagePullPolicy: IfNotPresent
        
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        
        envFrom:
        - configMapRef:
            name: api-config
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-server

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## The Challenge

Review the following deployment and identify all best practice violations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: API_KEY
          value: "sk_live_abc123"
```

**Find all issues and rewrite following best practices.**

<details>
<summary>Solution (click to expand)</summary>

**Issues found:**

1. Single replica (no HA)
2. No labels beyond app
3. No annotations
4. Using `:latest` tag
5. No resource requests/limits
6. No security context
7. No probes
8. Hardcoded secret in env
9. No PDB
10. No HPA

**Fixed version:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-secrets
type: Opaque
stringData:
  api-key: "sk_live_abc123"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/version: "1.25"
    app.kubernetes.io/component: frontend
  annotations:
    description: "Web frontend server"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: "1.25"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101  # nginx user
        fsGroup: 101
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: nginx
        image: nginx:1.25
        imagePullPolicy: IfNotPresent
        
        ports:
        - containerPort: 80
          name: http
        
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: web-secrets
              key: api-key
        
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

</details>

## Key Takeaways

1. **Resources** - Always set requests and limits
2. **Availability** - Multiple replicas, PDB, anti-affinity
3. **Health** - Configure all three probe types
4. **Security** - Non-root, read-only, drop capabilities
5. **Deployment** - Rolling updates, specific tags, graceful termination
6. **Organization** - Labels, namespaces, annotations
7. **Observability** - Metrics, structured logging
8. **Cost** - Right-size resources, quotas

## What You Can Do Now

- Write production-ready Kubernetes manifests
- Configure proper security contexts
- Set up high availability with PDB and anti-affinity
- Implement proper health checks
- Organize resources with labels and namespaces
- Optimize costs with resource management
- Follow the production deployment checklist

## What's Next

You know all the concepts and best practices. Time to put it all together.

**Next**: Chapter 10: Capstone Projects (Coming Soon)

---

**Estimated time**: 2-3 hours  
**Difficulty**: Advanced  
**Key skill**: Writing production-grade Kubernetes manifests
