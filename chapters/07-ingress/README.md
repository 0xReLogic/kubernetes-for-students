# Chapter 7: Ingress - The Front Door

## The Hotel Lobby Analogy

Imagine a hotel with hundreds of rooms. Guests don't knock on individual room doors from outside the building. Instead:

1. **They enter through the main lobby** (single entrance)
2. **Reception desk routes them** to the right room based on their reservation
3. **Security checks happen at entrance** (SSL/TLS)
4. **One address for the whole hotel** (not separate addresses per room)

**In Kubernetes:**
- **Hotel rooms are your Services** (nginx, api, database)
- **The lobby is Ingress** (single entry point)
- **Reception routing is Ingress rules** (based on hostname or path)
- **Hotel address is your domain** (example.com)

## The Reality: Why NodePort Isn't Enough

In Chapter 4, you learned about Services with NodePort. Let's see the problem:

**With NodePort:**
```bash
# Each service needs different port
nginx-service     -> NodePort 30080
api-service       -> NodePort 30081
blog-service      -> NodePort 30082

# Users access via:
http://your-server:30080  # nginx
http://your-server:30081  # api
http://your-server:30082  # blog
```

**Problems:**
1. **Port management nightmare** - Remember which port is which?
2. **No SSL/TLS** - Insecure HTTP traffic
3. **No path-based routing** - Can't use `/api` and `/blog` on same domain
4. **No host-based routing** - Can't use `api.example.com` and `blog.example.com`
5. **Unprofessional** - Production apps don't use ports in URLs

**With Ingress:**
```bash
# Clean URLs, all on port 80/443
http://example.com          -> nginx-service
http://example.com/api      -> api-service
http://example.com/blog     -> blog-service

# Or different domains
http://www.example.com      -> nginx-service
http://api.example.com      -> api-service
http://blog.example.com     -> blog-service
```

**Benefits:**
- Single entry point (port 80 for HTTP, 443 for HTTPS)
- SSL/TLS termination
- Path-based routing (`/api`, `/blog`)
- Host-based routing (subdomains)
- Load balancing
- Professional production setup

## What is Ingress?

**Ingress** is a Kubernetes resource that manages external access to services via HTTP/HTTPS.

**Two components needed:**
1. **Ingress Resource** - YAML definition of routing rules
2. **Ingress Controller** - The actual software that implements those rules

**Think of it as:**
- Ingress Resource = Restaurant menu (what's available)
- Ingress Controller = Waiter (makes it happen)

## Ingress Controllers

Kubernetes doesn't include an Ingress controller by default. You must install one.

**Popular options:**
- **NGINX Ingress Controller** - Most common, we'll use this
- **Traefik** - Simple, popular in Docker Swarm migrations
- **HAProxy** - High performance
- **Kong** - API gateway features
- **Istio Gateway** - Service mesh integration
- **Cloud-specific** - AWS ALB, GCP Load Balancer, Azure App Gateway

We'll use **NGINX Ingress Controller** because it's the most widely adopted.

## Experiment 1: Install NGINX Ingress Controller

First, install the Ingress controller in minikube:

```bash
# Enable ingress addon in minikube
minikube addons enable ingress

# Verify installation
kubectl get pods -n ingress-nginx
# Should see:
# ingress-nginx-controller-xxx   Running

# Check Ingress class
kubectl get ingressclass
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       1m
```

**What just happened:**
- NGINX Ingress Controller deployed in `ingress-nginx` namespace
- It watches for Ingress resources you create
- Acts as reverse proxy and load balancer

## Experiment 2: Simple Path-Based Routing

Let's create two apps and route them based on URL path.

### Step 1: Deploy Two Services

```yaml
# File: ingress-demo-apps.yaml
---
# App 1: Hello World
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Hello from App 1"
        - "-listen=:5678"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - port: 5678
    targetPort: 5678

---
# App 2: Goodbye World
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodbye-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: goodbye
  template:
    metadata:
      labels:
        app: goodbye
    spec:
      containers:
      - name: goodbye
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Goodbye from App 2"
        - "-listen=:5678"
        ports:
        - containerPort: 5678

---
apiVersion: v1
kind: Service
metadata:
  name: goodbye-service
spec:
  selector:
    app: goodbye
  ports:
  - port: 5678
    targetPort: 5678
```

```bash
kubectl apply -f ingress-demo-apps.yaml

# Verify services
kubectl get svc
# hello-service      ClusterIP   10.x.x.x    5678/TCP
# goodbye-service    ClusterIP   10.x.x.x    5678/TCP
```

### Step 2: Create Ingress Resource

```yaml
# File: ingress-path-based.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 5678
      - path: /goodbye
        pathType: Prefix
        backend:
          service:
            name: goodbye-service
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-path-based.yaml

# Check Ingress
kubectl get ingress
# NAME                  CLASS   HOSTS        ADDRESS        PORTS   AGE
# path-based-ingress    nginx   demo.local   192.168.49.2   80      1m

# Describe for details
kubectl describe ingress path-based-ingress
```

### Step 3: Test the Routing

```bash
# Get minikube IP
minikube ip
# Example output: 192.168.49.2

# Test hello path
curl -H "Host: demo.local" http://$(minikube ip)/hello
# Output: Hello from App 1

# Test goodbye path
curl -H "Host: demo.local" http://$(minikube ip)/goodbye
# Output: Goodbye from App 2

# Or add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
# 192.168.49.2  demo.local

# Then access via browser:
# http://demo.local/hello
# http://demo.local/goodbye
```

**The breakdown:**
- `/hello` routes to `hello-service`
- `/goodbye` routes to `goodbye-service`
- `rewrite-target: /` removes path prefix before forwarding
- Single IP, different paths, different services

## Experiment 3: Host-Based Routing (Virtual Hosts)

Route traffic based on domain/subdomain instead of path.

```yaml
# File: ingress-host-based.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  # Rule 1: hello.local -> hello-service
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 5678
  
  # Rule 2: goodbye.local -> goodbye-service
  - host: goodbye.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: goodbye-service
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-host-based.yaml

# Test with different hosts
curl -H "Host: hello.local" http://$(minikube ip)
# Output: Hello from App 1

curl -H "Host: goodbye.local" http://$(minikube ip)
# Output: Goodbye from App 2

# Add to /etc/hosts or Windows hosts file:
# 192.168.49.2  hello.local
# 192.168.49.2  goodbye.local

# Access via browser:
# http://hello.local
# http://goodbye.local
```

**Use case:** Different apps on different subdomains, all using port 80.

## Experiment 4: Default Backend

What if someone visits a path/host that doesn't match any rule?

```yaml
# File: ingress-with-default.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default
  template:
    metadata:
      labels:
        app: default
    spec:
      containers:
      - name: default
        image: hashicorp/http-echo:1.0
        args:
        - "-text=404 - Page Not Found"
        - "-listen=:5678"

---
apiVersion: v1
kind: Service
metadata:
  name: default-service
spec:
  selector:
    app: default
  ports:
  - port: 5678

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 5678
  rules:
  - host: demo.local
    http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-with-default.yaml

# Valid path
curl -H "Host: demo.local" http://$(minikube ip)/hello
# Output: Hello from App 1

# Invalid path
curl -H "Host: demo.local" http://$(minikube ip)/invalid
# Output: 404 - Page Not Found

# Invalid host
curl -H "Host: wrong.local" http://$(minikube ip)/anything
# Output: 404 - Page Not Found
```

## Experiment 5: HTTPS with TLS/SSL

Production apps need HTTPS. Let's add SSL certificates.

### Step 1: Create Self-Signed Certificate (for testing)

```bash
# Generate private key and certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=demo.local/O=demo"

# Create Kubernetes Secret from cert
kubectl create secret tls demo-tls \
  --cert=tls.crt \
  --key=tls.key

# Verify secret
kubectl get secret demo-tls
```

### Step 2: Create Ingress with TLS

```yaml
# File: ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - demo.local
    secretName: demo-tls
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-tls.yaml

# Test HTTPS (with self-signed cert warning)
curl -k https://demo.local  # -k ignores cert validation

# Or in browser (accept security warning):
# https://demo.local
```

**In production:** Use real certificates from Let's Encrypt with cert-manager.

## Experiment 6: URL Rewriting

Sometimes you need to modify URLs before forwarding.

```yaml
# File: ingress-rewrite.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    # Remove /api prefix before forwarding to backend
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      # User visits: /api/users
      # Backend receives: /users
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: hello-service
            port:
              number: 5678
```

**Common annotations:**
```yaml
annotations:
  # Rewrite URL
  nginx.ingress.kubernetes.io/rewrite-target: /

  # SSL redirect
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  
  # CORS headers
  nginx.ingress.kubernetes.io/enable-cors: "true"
  
  # Custom timeouts
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  
  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "10"
  
  # Basic auth
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
```

## Experiment 7: Real-World Multi-Service Setup

Deploy a complete app with frontend, backend API, and admin panel.

```yaml
# File: complete-app-ingress.yaml
---
# Frontend (React/Vue/etc)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80

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

---
# Backend API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
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
        image: hashicorp/http-echo:1.0
        args:
        - "-text=API Response: {status: ok}"
        - "-listen=:8080"
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 8080

---
# Admin Panel
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Admin Panel"
        - "-listen=:3000"
        ports:
        - containerPort: 3000

---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  selector:
    app: admin
  ports:
  - port: 3000

---
# Ingress for all services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: complete-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      # Frontend at root
      - path: /()(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      
      # API at /api/*
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      # Admin at /admin/*
      - path: /admin(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

```bash
kubectl apply -f complete-app-ingress.yaml

# Add to hosts file:
# 192.168.49.2  myapp.local

# Test different paths
curl http://myapp.local/           # Frontend
curl http://myapp.local/api/users  # API
curl http://myapp.local/admin      # Admin
```

## PathType Explained

Three options for matching paths:

### 1. Exact
Matches the exact path only.

```yaml
path: /hello
pathType: Exact
# Matches: /hello
# Not: /hello/, /hello/world
```

### 2. Prefix
Matches based on path prefix.

```yaml
path: /api
pathType: Prefix
# Matches: /api, /api/, /api/users, /api/v1/users
```

### 3. ImplementationSpecific
Controller-specific behavior (NGINX supports regex).

```yaml
path: /api(/|$)(.*)
pathType: ImplementationSpecific
# NGINX-specific regex matching
```

## The Challenge

Deploy a blog application with the following requirements:

**Services to deploy:**
1. Blog frontend (nginx) - accessible at `http://blog.local/`
2. Blog API (any image) - accessible at `http://blog.local/api/`
3. Admin dashboard - accessible at `http://admin.blog.local/`

**Requirements:**
- Use single Ingress resource with multiple rules
- Path `/api/` should forward to API service
- Subdomain `admin.blog.local` should route to admin dashboard
- Main domain `blog.local` should route to frontend

**Hints:**

```yaml
# You'll need:
# - 3 Deployments
# - 3 Services
# - 1 Ingress with:
#   - host: blog.local
#     paths: / and /api
#   - host: admin.blog.local
#     paths: /
```

**Try it yourself first.** Solution below.

<details>
<summary>Solution (click to expand)</summary>

**File: `blog-challenge.yaml`**
```yaml
---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blog-frontend
  template:
    metadata:
      labels:
        app: blog-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: blog-frontend-service
spec:
  selector:
    app: blog-frontend
  ports:
  - port: 80

---
# API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blog-api
  template:
    metadata:
      labels:
        app: blog-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Blog API: {posts: [], users: []}"
        - "-listen=:8080"
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: blog-api-service
spec:
  selector:
    app: blog-api
  ports:
  - port: 8080

---
# Admin Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog-admin
  template:
    metadata:
      labels:
        app: blog-admin
    spec:
      containers:
      - name: admin
        image: hashicorp/http-echo:1.0
        args:
        - "-text=Blog Admin Dashboard"
        - "-listen=:3000"
        ports:
        - containerPort: 3000

---
apiVersion: v1
kind: Service
metadata:
  name: blog-admin-service
spec:
  selector:
    app: blog-admin
  ports:
  - port: 3000

---
# Ingress with multiple hosts and paths
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  # Main blog domain
  - host: blog.local
    http:
      paths:
      # API at /api/*
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: blog-api-service
            port:
              number: 8080
      
      # Frontend at root (must be last for proper matching)
      - path: /()(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: blog-frontend-service
            port:
              number: 80
  
  # Admin subdomain
  - host: admin.blog.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-admin-service
            port:
              number: 3000
```

**Deploy and test:**
```bash
kubectl apply -f blog-challenge.yaml

# Add to /etc/hosts or Windows hosts file:
# 192.168.49.2  blog.local
# 192.168.49.2  admin.blog.local

# Test all endpoints
curl http://blog.local/
# Output: nginx default page

curl http://blog.local/api/posts
# Output: Blog API: {posts: [], users: []}

curl http://admin.blog.local/
# Output: Blog Admin Dashboard

# Or visit in browser:
# http://blog.local
# http://blog.local/api/posts
# http://admin.blog.local
```

</details>

## Common Ingress Patterns

### Pattern 1: Single Domain, Multiple Paths

```yaml
# example.com/        -> frontend
# example.com/api     -> api
# example.com/admin   -> admin
rules:
- host: example.com
  http:
    paths:
    - path: /admin
      backend: admin-service
    - path: /api
      backend: api-service
    - path: /
      backend: frontend-service
```

### Pattern 2: Subdomains

```yaml
# www.example.com    -> frontend
# api.example.com    -> api
# admin.example.com  -> admin
rules:
- host: www.example.com
  backend: frontend-service
- host: api.example.com
  backend: api-service
- host: admin.example.com
  backend: admin-service
```

### Pattern 3: Environment-Based

```yaml
# dev.example.com     -> dev-app
# staging.example.com -> staging-app
# example.com         -> prod-app
rules:
- host: dev.example.com
  backend: dev-service
- host: staging.example.com
  backend: staging-service
- host: example.com
  backend: prod-service
```

## Common Mistakes

### Mistake 1: Forgetting to Install Ingress Controller

```bash
kubectl apply -f ingress.yaml
# Ingress created, but nothing happens

kubectl get ingress
# ADDRESS column is empty
```

**Solution:** Install controller first (`minikube addons enable ingress`).

### Mistake 2: Wrong Path Order

```yaml
# WRONG - Greedy path comes first
paths:
- path: /           # Matches everything
  backend: frontend
- path: /api        # Never reached!
  backend: api
```

**Solution:** Specific paths first, general paths last.

### Mistake 3: Missing rewrite-target

```yaml
# User visits: /api/users
# Backend receives: /api/users (includes /api prefix)
# Backend expects: /users

# Error: Backend returns 404
```

**Solution:** Add rewrite annotation to strip prefix.

### Mistake 4: TLS Secret in Wrong Namespace

```bash
# Secret in namespace 'default'
kubectl create secret tls my-tls --cert=cert.crt --key=key.key

# Ingress in namespace 'prod'
kubectl apply -f ingress.yaml -n prod
# Error: Secret not found
```

**Solution:** Create secret in same namespace as Ingress.

### Mistake 5: Conflicting Ingress Rules

```yaml
# Ingress 1
host: example.com
path: /

# Ingress 2
host: example.com
path: /api
```

**Problem:** Both Ingress resources compete for same host.

**Solution:** Use single Ingress or different hosts.

## Cleanup

```bash
kubectl delete deployment hello-app goodbye-app default-backend frontend api admin blog-frontend blog-api blog-admin
kubectl delete service hello-service goodbye-service default-service frontend-service api-service admin-service blog-frontend-service blog-api-service blog-admin-service
kubectl delete ingress --all
kubectl delete secret demo-tls
```

## Key Takeaways

1. **Ingress is the production way** to expose services externally
2. **Ingress Controller required** - NGINX is most common
3. **Path-based routing** - Multiple services on one domain
4. **Host-based routing** - Multiple domains/subdomains
5. **TLS/SSL termination** - HTTPS made easy
6. **Annotations control behavior** - Rewrite, CORS, auth, etc.

## Ingress vs LoadBalancer vs NodePort

| Feature | NodePort | LoadBalancer | Ingress |
|---------|----------|--------------|---------|
| Port range | 30000-32767 | 80, 443 | 80, 443 |
| Cost | Free | $$$ (cloud LB) | $ (one LB) |
| SSL/TLS | Manual | Manual | Built-in |
| Path routing | No | No | Yes |
| Host routing | No | No | Yes |
| Production ready | No | Yes | Yes |

## When to Use What

**NodePort:**
- Development/testing only
- Quick demos
- Single service exposure

**LoadBalancer:**
- Cloud environments
- Simple setup (one service)
- No path/host routing needed

**Ingress:**
- Production environments
- Multiple services
- Path/host-based routing
- SSL termination
- Cost optimization (one LB for all services)

## What You Can Do Now

- Expose multiple services through single entry point
- Route traffic based on URL path or hostname
- Add SSL/TLS for HTTPS
- Configure URL rewriting
- Set up production-ready external access
- Use annotations for advanced features

## What's Next

You can expose apps professionally. But when things break, how do you debug? Time to master troubleshooting.

**Next**: Chapter 8: Debugging & Troubleshooting (Coming Soon)

---

**Estimated time**: 2-3 hours  
**Difficulty**: Intermediate  
**Key command**: `kubectl get ingress` - Check your routing rules
