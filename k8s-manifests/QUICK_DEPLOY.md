# 🚀 Quick Deployment Guide

## Exact Structure (As Shown in Image)

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
│       ├── secrets.yaml              # optional
│       ├── configmap.yaml
│       └── namespace.yaml
│
├── argocd/
│   └── industrial-app.yaml
│
└── README.md
```

## ⚡ 5-Minute Setup

### 1. Clone & Customize (2 min)

```bash
# Extract the archive
tar -xzf k8s-manifests.tar.gz
cd k8s-manifests

# Update image tags
vim apps/industrial-app/backend/deployment.yaml
# Change: image: YOUR_ECR_URL/backend:YOUR_TAG

vim apps/industrial-app/frontend/deployment.yaml
# Change: image: YOUR_ECR_URL/frontend:YOUR_TAG
```

### 2. Update Domain (1 min)

```bash
vim apps/industrial-app/ingress.yaml
# Change: host: app.yourdomain.com

vim apps/industrial-app/configmap.yaml
# Change: mpesa-callback-url
```

### 3. Push to GitHub (1 min)

```bash
git init
git add .
git commit -m "Initial manifests"
git remote add origin https://github.com/YOUR_USERNAME/k8s-manifests.git
git push -u origin main
```

### 4. Deploy with ArgoCD (1 min)

```bash
# Update ArgoCD app with your repo URL
vim argocd/industrial-app.yaml

# Apply
kubectl apply -f argocd/industrial-app.yaml

# Watch deployment
argocd app get industrial-app
```

## 🎯 Key Points

### ✅ This Works With ArgoCD Because:
1. **apps/** folder contains all Kubernetes resources
2. **ArgoCD points to apps/industrial-app/** path
3. All YAML files automatically applied
4. Clean, scalable structure

### 🔄 Update Workflow:
```
1. Build images → Push to ECR
2. Update deployment.yaml with new tags
3. Git commit & push
4. ArgoCD syncs automatically!
```

### 🔐 Secrets:
```bash
# Create manually (don't commit secrets.yaml)
kubectl create secret generic app-secrets \
  --from-literal=mongo-uri='...' \
  --from-literal=jwt-secret='...' \
  --namespace=industrial-attachment
```

### 🎨 Why This Structure?
- ✅ Clean separation (apps vs argocd)
- ✅ Zero application code
- ✅ ArgoCD friendly
- ✅ Scales easily
- ✅ Industry standard

---

**That's it!** ArgoCD does the rest! 🎉
