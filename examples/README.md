# Kubernetes Examples

This directory contains all YAML files used in the chapters, organized for easy reference.

## Structure

```
examples/
├── chapter-02/   # Pods Deep Dive
├── chapter-03/   # Deployments
├── chapter-04/   # Services
├── chapter-05/   # ConfigMaps & Secrets
├── chapter-06/   # Persistent Storage
├── chapter-07/   # Ingress
├── chapter-10/   # Capstone Projects
└── ...
```

## Usage

All files are tested and ready to use:

```bash
# Apply any example
kubectl apply -f examples/chapter-03/nginx-deployment.yaml

# Apply all files in a directory
kubectl apply -f examples/chapter-03/

# Delete resources
kubectl delete -f examples/chapter-03/nginx-deployment.yaml
```

## Notes

- All examples use standard public images (nginx, busybox, etc.)
- Files include comments explaining each field
- Designed for minikube but work on any Kubernetes cluster
- Use as templates for your own applications

---

## Chapter 10: Capstone Projects

Complete, production-ready application examples.

### Project 1: Blog Platform

**File:** `chapter-10/project-1-blog-platform.yaml`

A full-stack blog application with:
- Frontend: NGINX serving HTML/CSS/JavaScript
- Backend: Node.js REST API
- Database: MySQL with persistent storage
- Features: Create and read blog posts

**Quick Start:**
```bash
# Deploy everything
kubectl apply -f examples/chapter-10/project-1-blog-platform.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all -n blog-platform --timeout=180s

# Access via NodePort (minikube)
minikube service blog-frontend -n blog-platform

# Or use port-forward
kubectl port-forward svc/blog-frontend 8080:80 -n blog-platform
# Open: http://localhost:8080
```

**Components:**
- Namespace: `blog-platform`
- MySQL: 1 replica with 1Gi persistent storage
- API: 2 replicas with auto-scaling ready
- Frontend: 2 replicas with NGINX reverse proxy
- All services have health checks and resource limits

**Testing:**
```bash
# Check all pods are running
kubectl get pods -n blog-platform

# Check services
kubectl get svc -n blog-platform

# Test API directly
kubectl port-forward svc/blog-api 3000:3000 -n blog-platform
curl http://localhost:3000/health

# Check logs
kubectl logs -l tier=backend -n blog-platform
kubectl logs -l tier=database -n blog-platform
```

**Challenge Extensions:**
1. Add DELETE endpoint for posts
2. Implement post categories
3. Add search functionality
4. Implement pagination (10 posts per page)
5. Add user authentication
6. Set up Ingress with custom domain

**Cleanup:**
```bash
kubectl delete namespace blog-platform
```
