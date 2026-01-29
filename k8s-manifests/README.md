# Industrial Attachment System - Kubernetes Manifests

Production-grade, Argo CD-friendly manifest repository layout following best practices.

## 📁 Repository Structure

```
k8s-manifests/
├── apps/
│   └── industrial-app/
│       ├── backend/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── hpa.yaml              # optional
│       ├── frontend/
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       ├── ingress.yaml
│       ├── secrets.yaml              # optional (use External Secrets)
│       ├── configmap.yaml
│       └── namespace.yaml
│
├── argocd/
│   └── industrial-app.yaml           # ArgoCD Application definition
│
└── README.md
```

## 🎯 Why This Structure Works

### ✅ Clean Separation
- **apps/** → All Kubernetes resources that will be applied
- **argocd/** → ArgoCD Application definitions (how ArgoCD deploys)

### ✅ ArgoCD Friendly
- ArgoCD points to `apps/industrial-app/` 
- All YAML files in that directory are automatically applied
- Easy to manage and scale

### ✅ No Application Code
- This repo contains **ZERO application code**
- Only **Kubernetes intent** (manifests)
- Application code is in separate repository

## 🚀 Quick Start

### Step 1: Clone and Customize

```bash
git clone https://github.com/YOUR_USERNAME/k8s-manifests.git
cd k8s-manifests
```

### Step 2: Update Image Tags

Edit the deployment files with your ECR images:

**apps/industrial-app/backend/deployment.yaml:**
```yaml
image: 123456789.dkr.ecr.us-east-1.amazonaws.com/industrial-attachment-backend:abc1234
```

**apps/industrial-app/frontend/deployment.yaml:**
```yaml
image: 123456789.dkr.ecr.us-east-1.amazonaws.com/industrial-attachment-frontend:abc1234
```

### Step 3: Update Domain

**apps/industrial-app/ingress.yaml:**
```yaml
- host: app.yourdomain.com
```

**apps/industrial-app/configmap.yaml:**
```yaml
mpesa-callback-url: "https://app.yourdomain.com/api/applications/mpesa/callback"
```

### Step 4: Configure Secrets

**Option A: Using External Secrets Operator (Recommended)**

Create an ExternalSecret that syncs from AWS Secrets Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: industrial-attachment
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-secrets
  data:
    - secretKey: mongo-uri
      remoteRef:
        key: industrial-attachment/mongo-uri
    # ... more secrets
```

**Option B: Create Manually in Cluster**

```bash
kubectl create secret generic app-secrets \
  --from-literal=mongo-uri='mongodb://...' \
  --from-literal=jwt-secret='...' \
  --from-literal=mpesa-consumer-key='...' \
  --from-literal=mpesa-consumer-secret='...' \
  --from-literal=mpesa-shortcode='...' \
  --from-literal=mpesa-passkey='...' \
  --from-literal=cloudinary-cloud-name='...' \
  --from-literal=cloudinary-api-key='...' \
  --from-literal=cloudinary-api-secret='...' \
  --namespace=industrial-attachment
```

**Option C: Use Sealed Secrets**

```bash
kubeseal --format yaml < secrets.yaml > sealed-secrets.yaml
# Commit sealed-secrets.yaml (safe to commit)
```

### Step 5: Create ECR Pull Secret

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region REGION) \
  --namespace=industrial-attachment
```

### Step 6: Push to GitHub

```bash
git init
git add .
git commit -m "Initial Kubernetes manifests"
git remote add origin https://github.com/YOUR_USERNAME/k8s-manifests.git
git push -u origin main
```

### Step 7: Deploy with ArgoCD

**Update ArgoCD application:**

Edit `argocd/industrial-app.yaml`:
```yaml
source:
  repoURL: https://github.com/YOUR_USERNAME/k8s-manifests.git
```

**Apply:**
```bash
kubectl apply -f argocd/industrial-app.yaml
```

**Monitor:**
```bash
argocd app get industrial-app
argocd app sync industrial-app
kubectl get pods -n industrial-attachment -w
```

## 🔄 Workflow

### Development Flow

```
1. Build new Docker images in App Repo
   ↓
2. Images pushed to ECR with tags (e.g., abc1234)
   ↓
3. Update deployment.yaml in THIS repo with new tags
   ↓
4. Commit and push
   ↓
5. ArgoCD automatically detects changes
   ↓
6. ArgoCD syncs and deploys new version
   ↓
7. Rolling update in Kubernetes
```

### Update Image Tags

```bash
# Edit deployment files
vim apps/industrial-app/backend/deployment.yaml
vim apps/industrial-app/frontend/deployment.yaml

# Update image tags
# image: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/repo:NEW_TAG

# Commit and push
git add apps/industrial-app/
git commit -m "Update to version NEW_TAG"
git push

# ArgoCD syncs automatically!
```

## 📝 File Descriptions

### apps/industrial-app/

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the namespace |
| `configmap.yaml` | Non-sensitive configuration |
| `secrets.yaml` | Sensitive configuration (template) |
| `backend/deployment.yaml` | Backend pods definition |
| `backend/service.yaml` | Backend service (ClusterIP) |
| `backend/hpa.yaml` | Auto-scaling rules (optional) |
| `frontend/deployment.yaml` | Frontend pods definition |
| `frontend/service.yaml` | Frontend service (ClusterIP) |
| `ingress.yaml` | External access configuration |

### argocd/

| File | Purpose |
|------|---------|
| `industrial-app.yaml` | ArgoCD Application - tells ArgoCD what to deploy |

## 🎯 ArgoCD Application Explained

```yaml
spec:
  source:
    path: apps/industrial-app  # ArgoCD reads all YAML files here
  
  destination:
    namespace: industrial-attachment  # Where to deploy
  
  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Fix drift automatically
```

**This means:**
- ArgoCD watches `apps/industrial-app/` directory
- Any YAML file added/modified/deleted triggers a sync
- ArgoCD keeps cluster in sync with Git (GitOps!)

## 🔐 Security Best Practices

### ✅ DO:
- Use External Secrets Operator or Sealed Secrets
- Store secrets in AWS Secrets Manager / Vault
- Use RBAC for namespace access
- Enable Network Policies
- Use Pod Security Standards

### ❌ DON'T:
- Commit plain secrets to Git
- Store passwords in ConfigMaps
- Use default service accounts
- Allow pods to run as root

## 📊 Monitoring

### ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

### Check Application Status

```bash
# ArgoCD
argocd app get industrial-app
argocd app sync industrial-app
argocd app logs industrial-app

# Kubernetes
kubectl get all -n industrial-attachment
kubectl get pods -n industrial-attachment -w
kubectl logs -f deployment/backend -n industrial-attachment
```

## 🔄 Scaling

### Manual Scaling

```bash
kubectl scale deployment/backend --replicas=5 -n industrial-attachment
```

### Auto-Scaling (HPA)

HPA is already configured in `backend/hpa.yaml`:
- Min replicas: 3
- Max replicas: 10
- Target CPU: 70%
- Target Memory: 80%

## 🐛 Troubleshooting

### ArgoCD Not Syncing

```bash
# Check app status
argocd app get industrial-app

# Force refresh
argocd app get industrial-app --refresh

# Force sync
argocd app sync industrial-app --force

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller
```

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n industrial-attachment

# Describe pod
kubectl describe pod POD_NAME -n industrial-attachment

# Check logs
kubectl logs POD_NAME -n industrial-attachment

# Check events
kubectl get events -n industrial-attachment --sort-by='.lastTimestamp'
```

### Image Pull Errors

```bash
# Check secret
kubectl get secret ecr-registry-secret -n industrial-attachment

# Recreate secret (ECR tokens expire every 12 hours)
kubectl delete secret ecr-registry-secret -n industrial-attachment
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region REGION) \
  --namespace=industrial-attachment
```

## 🎓 Best Practices

1. **One app per directory** - Each app in `apps/` is independent
2. **Use namespaces** - Isolate applications
3. **GitOps everything** - All changes via Git commits
4. **Automated sync** - Let ArgoCD handle deployments
5. **Self-healing enabled** - ArgoCD fixes configuration drift
6. **Health checks** - Liveness and readiness probes configured
7. **Resource limits** - Prevent resource exhaustion
8. **Auto-scaling** - HPA configured for production load

## ✅ Pre-Deployment Checklist

- [ ] Updated image tags in deployment files
- [ ] Updated domain in ingress.yaml
- [ ] Updated callback URL in configmap.yaml
- [ ] Created app-secrets in cluster
- [ ] Created ecr-registry-secret in cluster
- [ ] Pushed manifests to GitHub
- [ ] Updated ArgoCD application with correct repo URL
- [ ] Applied ArgoCD application
- [ ] Verified ArgoCD can access GitHub repo
- [ ] Monitoring configured

## 🎉 That's It!

This layout:
- ✅ Works cleanly with ArgoCD
- ✅ Scales easily (add more apps to `apps/`)
- ✅ Clear separation of concerns
- ✅ Industry best practice
- ✅ Zero application code (manifests only)

---

**Ready to deploy!** Push to GitHub and watch ArgoCD work its magic! 🚀
