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
