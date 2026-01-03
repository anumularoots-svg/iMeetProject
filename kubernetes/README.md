# iMeetPro Kubernetes Deployment Guide

## Prerequisites

1. **EKS Cluster** - Running with kubectl configured
2. **NGINX Ingress Controller** - Installed
3. **cert-manager** - Installed for SSL certificates
4. **ECR Images** - Backend and frontend images pushed

## Deployment Order

```bash
# 1. Create Namespaces
kubectl apply -f kubernetes/namespaces/namespaces.yaml

# 2. Deploy Databases
kubectl apply -f kubernetes/databases/mongodb/deployment.yaml
kubectl apply -f kubernetes/databases/redis/deployment.yaml

# 3. Wait for databases to be ready
kubectl get pods -n databases -w

# 4. Create Secrets (Update values first!)
kubectl apply -f kubernetes/secrets/secrets.yaml

# 5. Create ConfigMap
kubectl apply -f kubernetes/apps/backend/configmap.yaml

# 6. Deploy Backend
kubectl apply -f kubernetes/apps/backend/deployment.yaml

# 7. Deploy Frontend
kubectl apply -f kubernetes/apps/frontend/deployment.yaml

# 8. Deploy Celery Workers (Optional)
kubectl apply -f kubernetes/apps/celery/deployment.yaml

# 9. Deploy GPU Workers (Optional - requires GPU nodes)
# kubectl apply -f kubernetes/apps/gpu-workers/deployment.yaml

# 10. Create Ingress
kubectl apply -f kubernetes/ingress/ingress.yaml

# 11. Verify Deployment
kubectl get pods -n imeetpro
kubectl get pods -n databases
kubectl get ingress -n imeetpro
kubectl get certificate -n imeetpro
```

## Configuration

### Update ConfigMap (kubernetes/apps/backend/configmap.yaml)

Update these values for your environment:
- `DB_HOST` - Your RDS endpoint
- `LIVEKIT_URL` - Your LiveKit server URL
- `AWS_STORAGE_BUCKET_NAME` - Your S3 bucket name

### Update Secrets (kubernetes/secrets/secrets.yaml)

Encode your secrets in base64:
```bash
echo -n 'your-password' | base64
```

Update these values:
- `DB_USER` / `DB_PASSWORD` - RDS credentials
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` - AWS credentials
- `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` - LiveKit credentials
- `DJANGO_SECRET_KEY` - Django secret key

### Update Ingress (kubernetes/ingress/ingress.yaml)

Update these values:
- Hostnames (`www.lancieretech.com`, `api.lancieretech.com`)
- Email for Let's Encrypt (`akhil@lancieretech.com`)

## Troubleshooting

### Check Pod Logs
```bash
kubectl logs -l app=backend -n imeetpro --tail=50
kubectl logs -l app=frontend -n imeetpro --tail=50
```

### Check Pod Status
```bash
kubectl describe pod <pod-name> -n imeetpro
```

### Check Ingress
```bash
kubectl describe ingress imeetpro-ingress -n imeetpro
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

### Check Certificate
```bash
kubectl get certificate -n imeetpro
kubectl describe certificate imeetpro-tls -n imeetpro
```

## Important Notes

1. **Frontend Port**: The frontend nginx container listens on port **8080**, not 80. The service maps port 80 -> 8080.

2. **Backend Health Check**: Uses `/api/videos/lists` endpoint. Ensure this endpoint is accessible without authentication.

3. **MongoDB**: Using emptyDir for data. For production, configure PersistentVolumeClaim.

4. **Redis**: Using emptyDir for data. For production, configure PersistentVolumeClaim.

5. **RDS Security Group**: Ensure EKS nodes can access RDS (add VPC CIDR to RDS security group).

## Architecture

```
                    ┌─────────────────┐
                    │   CloudFront/   │
                    │    GoDaddy      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  NGINX Ingress  │
                    │  (LoadBalancer) │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
│    Frontend     │ │    Backend      │ │   Celery        │
│    (React)      │ │    (Django)     │ │   Workers       │
│    Port 8080    │ │    Port 8000    │ │                 │
└─────────────────┘ └────────┬────────┘ └────────┬────────┘
                             │                   │
         ┌───────────────────┼───────────────────┤
         │                   │                   │
┌────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
│    MongoDB      │ │    MySQL RDS    │ │     Redis       │
│    Port 27017   │ │    Port 3306    │ │    Port 6379    │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```
