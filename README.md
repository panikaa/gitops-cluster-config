# 🚀 GitOps Cluster Config

**ArgoCD-driven GitOps repository for managing Kubernetes applications and cluster configuration.**

This repository contains the declarative Kubernetes manifests for the **HCM Platform** running on **AWS EKS**, including:

* Application deployments (frontend + backend)
* Ingress (AWS Load Balancer Controller)
* ConfigMaps & Secrets
* Horizontal Pod Autoscalers
* Environment overlays
* ArgoCD Application definitions

All Kubernetes changes flow through Git → ArgoCD → EKS.
This is a **production-grade GitOps setup**, fully automated end-to-end.

---

## 📦 Repository Structure

```
gitops-cluster-config/
│
├── apps/
|   |   platform/
|   |   ├── kustomization.yaml
|   |   └── clustersecretstore.yaml
│   └── hcm/
│       ├── base/
│       │   ├── backend-deployment.yaml
│       │   ├── backend-service.yaml
│       │   ├── frontend-deployment.yaml
│       │   ├── frontend-service.yaml
│       │   ├── ingress.yaml
│       │   ├── configmap.yaml
│       │   ├── hpa.yaml
│       │   ├── secret.enc.yaml (SOPS optional)
│       │   └── kustomization.yaml
│       │
│       └── overlays/
│           └── prod/
│               └── kustomization.yaml
│
└── argo/
    ├── projects/
    |   └── hcm-project.yaml
    ├── install/
    |   └── argocd-install.yaml
    └── apps/
        ├── hcm-app.yaml
        └── platform-app.yaml
```  

### 🟦 **apps/hcm/base**

Base manifests — shared by all environments.
This includes:

* Deployments and Services
* Ingress configuration
* AWS LBC annotations
* Environment variables
* ConfigMap
* Horizontal Pod Autoscaler
* Internal secrets (can use SOPS or AWS Secrets Manager)

### 🟦 **apps/hcm/overlays/prod**

Production overrides, e.g.:

* replica count
* resource limits
* additional annotations
* production-grade ingress config

### 🟦 **argo/apps/hcm-app.yaml**

ArgoCD `Application` object that points to this repo.
ArgoCD automatically syncs all changes.

### 🟦 **argo/apps/platform-app.yaml**

External Secrets `Application` object that points to this repo.
Automatically manage secrets from AWS.

### 🟦 **argo/install/argocd-install.yaml**

Kubernetes `Application` object that intall ArgoCD in cluster.
Kubernetes automatically install Argo and all of depends (service accounts, namespace etc.)

### 🟦 **argo/projects/hcm-project.yaml**

ArgoCD `AppProject` object that points to this repo.
ArgoCD project for HCM.

---

## 🔄 GitOps Deployment Flow

The pipeline looks like this:

1. **Developer pushes code** to app repo (frontend/backend)
2. **GitHub Actions builds Docker images**
3. Images are tagged using the commit SHA and pushed to GHCR
4. GitHub Actions updates this GitOps repo:

   * `backend-deployment.yaml` → new image tag
   * `frontend-deployment.yaml` → new image tag
5. ArgoCD detects changes and deploys automatically:

   * rolling update
   * health checks
   * self-healing
6. AWS Load Balancer Controller updates the ALB Ingress

This is a full CI → CD → GitOps → Kubernetes production pipeline.

---

## 🔐 Secrets

Secrets may be stored as:

### Option A: Plain Kubernetes secret (base64)

```
kubectl create secret generic hcm-db --from-literal=DATABASE_URL=...
```

### Option B: SOPS encrypted secret

(one of the best GitOps patterns)

This repo supports `secret.enc.yaml` files if you decide to enable it.

---

## 🌐 AWS Integration

This GitOps repository assumes:

* **EKS cluster** deployed via Terraform
* **AWS Load Balancer Controller** (IRSA-enabled)
* **RDS PostgreSQL** as backend database
* **External Secrets** as backend for secret managment
* **GHCR registry** with an `imagePullSecret` defined in namespace `hcm`

Ingress uses AWS LBC annotations:

```
kubernetes.io/ingress.class: alb
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
```

---

## 🧪 Health Checks

Frontend & backend include:

```
/health
```

Used by:

* readiness probes
* liveness probes
* AWS TargetGroup health checks

---

## 🤖 ArgoCD Sync

ArgoCD is set with:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Meaning:

* **Every change in Git is deployed instantly**
* **ArgoCD corrects drift automatically**
* **Resources removed from Git are removed from the cluster**

---

## 🚀 How to Apply Manually (debug)

If needed:

### Sync:

```
argocd app sync hcm
```

### Refresh:

```
argocd app refresh hcm
```

### Get status:

```
argocd app get hcm
```

---

## 🧩 Deployment Example (image update)

GitHub Actions updates:

```
image: ghcr.io/panikaa/hcm-backend:<sha>
```

You may update manually:

```
sed -i 's|image: .*/hcm-backend:.*|image: ghcr.io/panikaa/hcm-backend:NEW_TAG|' apps/hcm/base/backend-deployment.yaml
git commit -am "update backend image"
git push
```

ArgoCD deploys automatically.

---

## 🏁 Summary

This repo contains the **desired state** of the HCM application deployed to AWS EKS.
It is the single source of truth for:

* Deployment manifests
* Environment configuration
* Ingress routing
* Autoscaling
* Secrets
* Sync rules for ArgoCD

Together with the Terraform and CI/CD repos, this forms a complete, production-grade DevOps pipeline.

