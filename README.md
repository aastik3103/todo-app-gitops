# Kubernetes Deployment Guide

This directory contains Kubernetes YAML configurations for deploying the Todo List API and PostgreSQL database.

## Files Overview

- `namespace.yaml` - Creates a dedicated namespace for the application
- `configmap.yaml` - Non-sensitive configuration data
- `secret.yaml` - Sensitive data (database password)
- `postgres-pvc.yaml` - Persistent volume claim for PostgreSQL data
- `postgres-deployment.yaml` - PostgreSQL database deployment
- `postgres-service.yaml` - PostgreSQL service (ClusterIP)
- `app-deployment.yaml` - Todo API application deployment
- `app-service.yaml` - Todo API service (LoadBalancer/NodePort)
- `ingress.yaml` - Ingress configuration for external access (optional)

## Prerequisites

1. Kubernetes cluster (minikube, GKE, EKS, AKS, or local cluster)
2. kubectl configured to access your cluster
3. Docker image built and pushed to a container registry (or available locally)

## Deployment Steps

### 1. Build and Push Docker Image

First, build your Docker image and push it to a container registry:

```bash
# Build the image
docker build -t todo-api:latest .

# Tag for your registry (replace with your registry)
docker tag todo-api:latest your-registry.com/todo-api:latest

# Push to registry
docker push your-registry.com/todo-api:latest
```

**For local/minikube:**
```bash
# Build directly into minikube
eval $(minikube docker-env)
docker build -t todo-api:latest .
```

### 2. Update Image Reference

Edit `app-deployment.yaml` and update the image field:
```yaml
image: your-registry.com/todo-api:latest
```

For local/minikube, you can use:
```yaml
image: todo-api:latest
imagePullPolicy: IfNotPresent
```

### 3. Update Secret

**Option A: Edit secret.yaml**
```yaml
stringData:
  DB_PASSWORD: "your-actual-password"
```

**Option B: Create secret via command line (recommended)**
```bash
kubectl create secret generic todo-secret \
  --from-literal=DB_PASSWORD=yourpassword \
  -n todo-app
```

### 4. Update Storage Class

Edit `postgres-pvc.yaml` and update the storageClassName to match your cluster:
```yaml
storageClassName: standard  # Change to match your cluster
```

Common values:
- `standard` (GKE default)
- `gp2` (EKS default)
- `default` (minikube)
- `managed-premium` (AKS)

### 5. Deploy to Kubernetes

Deploy in order:

```bash
# 1. Create namespace
kubectl apply -f namespace.yaml

# 2. Create ConfigMap
kubectl apply -f configmap.yaml

# 3. Create Secret
kubectl apply -f secret.yaml

# 4. Create PVC for PostgreSQL
kubectl apply -f postgres-pvc.yaml

# 5. Deploy PostgreSQL
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml

# 6. Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n todo-app --timeout=120s

# 7. Deploy Application
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml

# 8. (Optional) Deploy Ingress
kubectl apply -f ingress.yaml
```

**Or deploy all at once:**
```bash
kubectl apply -f .
```

### 6. Verify Deployment

```bash
# Check pods
kubectl get pods -n todo-app

# Check services
kubectl get svc -n todo-app

# Check deployments
kubectl get deployments -n todo-app

# View logs
kubectl logs -f deployment/todo-api-deployment -n todo-app
kubectl logs -f deployment/postgres-deployment -n todo-app
```

### 7. Access the Application

**Using LoadBalancer Service:**
```bash
# Get external IP
kubectl get svc todo-api-service -n todo-app

# Access via external IP
curl http://<EXTERNAL-IP>:3000/api/health
```

**Using NodePort Service:**
```bash
# Get node IP and port
kubectl get svc todo-api-service -n todo-app

# Access via node IP and nodePort
curl http://<NODE-IP>:<NODE-PORT>/api/health
```

**Using Port Forward (for testing):**
```bash
kubectl port-forward svc/todo-api-service 3000:3000 -n todo-app

# Access via localhost
curl http://localhost:3000/api/health
```

**Using Ingress:**
```bash
# Get ingress IP
kubectl get ingress -n todo-app

# Access via ingress hostname
curl http://todo-api.yourdomain.com/api/health
```

## Scaling

Scale the application:
```bash
kubectl scale deployment todo-api-deployment --replicas=3 -n todo-app
```

## Updating the Application

```bash
# Update image
kubectl set image deployment/todo-api-deployment \
  todo-api=your-registry.com/todo-api:v1.1.0 \
  -n todo-app

# Or apply updated deployment
kubectl apply -f app-deployment.yaml
kubectl rollout restart deployment/todo-api-deployment -n todo-app
```

## Troubleshooting

### Check pod status
```bash
kubectl describe pod <pod-name> -n todo-app
```

### Check events
```bash
kubectl get events -n todo-app --sort-by='.lastTimestamp'
```

### Debug pod
```bash
kubectl exec -it <pod-name> -n todo-app -- /bin/sh
```

### View logs
```bash
kubectl logs <pod-name> -n todo-app
kubectl logs -f deployment/todo-api-deployment -n todo-app
```

### Check service endpoints
```bash
kubectl get endpoints -n todo-app
```

## Cleanup

To remove all resources:
```bash
kubectl delete -f .
# Or delete namespace (removes everything)
kubectl delete namespace todo-app
```

## Configuration Options

### Resource Limits
Adjust CPU and memory limits in:
- `app-deployment.yaml` (for the API)
- `postgres-deployment.yaml` (for the database)

### Replicas
Change the number of replicas in `app-deployment.yaml`:
```yaml
replicas: 3  # Increase for high availability
```

### Service Type
Change service type in `app-service.yaml`:
- `ClusterIP` - Internal only
- `NodePort` - Expose on node IP
- `LoadBalancer` - Cloud load balancer

## Security Notes

1. **Never commit secrets.yaml with real passwords**
2. Use Kubernetes Secrets or external secret management
3. Consider using sealed-secrets or external-secrets operator
4. Enable RBAC for production deployments
5. Use NetworkPolicies to restrict pod communication

## Production Recommendations

1. Use a managed database service (RDS, Cloud SQL, etc.) instead of self-hosted PostgreSQL
2. Enable TLS/SSL for database connections
3. Use HorizontalPodAutoscaler for auto-scaling
4. Set up monitoring with Prometheus and Grafana
5. Configure proper backup strategies for PostgreSQL
6. Use resource quotas and limits
7. Enable pod security policies
8. Use cert-manager for SSL certificates

