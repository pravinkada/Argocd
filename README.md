# ArgoCD Setup on GCP (GKE)

install ArgoCD on Google Kubernetes Engine (GKE), set up GCP integration, deploy applications, and troubleshoot.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)  
- [Install ArgoCD](#install-argocd)  
- [GCP Setup](#gcp-setup)  
- [Access ArgoCD](#access-argocd)  
- [Deploy Apps](#deploy-apps)  
- [Verify](#verify)  
- [Troubleshooting](#troubleshooting)  
- [Cleanup](#cleanup)  

---

## ✅ Prerequisites

- GCP project and billing enabled  
- GKE cluster created and configured (`kubectl config use-context`)  
- Tools installed:
  - `kubectl`
  - `helm`
  - `argocd` CLI
  - `gcloud` SDK
- Permissions to create IAM resources

---

## 📦 Install ArgoCD

### Using Manifests

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Using Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd
```

---

## ⚙️ GCP Setup

### Create IAM Service Account

```bash
gcloud iam service-accounts create argocd-sa
```

### Grant Permissions

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:argocd-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/container.viewer"
```

### Generate Key and Create Secret

```bash
gcloud iam service-accounts keys create key.json \
  --iam-account=argocd-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic gcp-creds --from-file=key.json -n argocd
```

---

## 🌐 Access ArgoCD

### Option 1: LoadBalancer (Recommended for GCP)

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

Access via `https://<EXTERNAL-IP>`

### Option 2: Port Forward (Local)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open: `https://localhost:8080`

### Get Admin Password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

## 🚀 Deploy Apps

### Using CLI

```bash
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD> --insecure

argocd app create demo-app \
  --repo https://github.com/YOUR_USER/YOUR_REPO.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync demo-app
```

### Using YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USER/YOUR_REPO.git
    path: k8s
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f app.yaml
```

---

## 🔍 Verify

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
argocd app list
```

---

## 🐞 Troubleshooting

- **No EXTERNAL-IP**: Wait a few minutes or check LoadBalancer quota.
- **Can't login**: Use base64-decoded admin password.
- **App not syncing**: Check Git URL, branch, and path.
- **GCR pull errors**: Confirm IAM and secret setup.

---

## 🧹 Cleanup

```bash
kubectl delete ns argocd
gcloud iam service-accounts delete argocd-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com
rm key.json
```

---

## 📚 References

- [ArgoCD Docs](https://argo-cd.readthedocs.io)
- [GKE Docs](https://cloud.google.com/kubernetes-engine/docs)
