# k8s-nginx-ha-argocd

A GitOps-driven, highly available NGINX deployment on Kubernetes, managed by Argo CD.

---

## Overview

This project demonstrates a production-style GitOps workflow using:

- **Kubernetes** — container orchestration with namespace isolation
- **NGINX** — lightweight web server running as 2 replicas for high availability
- **ConfigMap** — HTML content decoupled from the container image
- **Ingress** — external HTTP routing via the NGINX Ingress Controller
- **Argo CD** — continuous delivery with automated sync and self-healing

---

## Project Structure

```
k8s-nginx-ha-argocd/
│
├── app/
│   ├── namespace.yaml     # Isolated web-app namespace
│   ├── configmap.yaml     # HTML content as a ConfigMap
│   ├── deployment.yaml    # 2-replica NGINX Deployment
│   ├── service.yaml       # ClusterIP Service
│   └── ingress.yaml       # Ingress for external HTTP access
│
├── argocd/
│   └── application.yaml   # Argo CD Application manifest
│
└── README.md
```

---

## Kubernetes Manifests

### `app/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web-app
```
Creates an isolated `web-app` namespace to contain all application resources.

---

### `app/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
  namespace: web-app
data:
  index.html: |
    <h1>ArgoCD Deployed App</h1>
    <p>GitOps in action</p>
```
Stores the HTML page as a ConfigMap. Content is version-controlled in Git and mounted into the NGINX container — no image rebuild needed for content changes.

---

### `app/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: nginx-html
```
Runs **2 NGINX replicas** for high availability. The ConfigMap is mounted as a volume at `/usr/share/nginx/html`, serving the HTML content defined above.

---

### `app/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: web-app
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```
Creates a stable internal `ClusterIP` Service that load-balances traffic across the 2 NGINX pods on port 80.

---

### `app/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: web-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```
Exposes the application to browser traffic by routing all HTTP requests at `/` to the `nginx-service`.

---

### `argocd/application.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/k8s-nginx-ha-argocd
    targetRevision: HEAD
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: web-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Registers the app with Argo CD. Points to the `app/` directory in your Git repo and deploys into the `web-app` namespace with:

| Setting | Effect |
|---|---|
| `prune: true` | Removes resources deleted from Git |
| `selfHeal: true` | Reverts any manual changes made directly to the cluster |

> **Before applying:** replace `YOUR-USERNAME` with your actual GitHub username.

---

## Prerequisites

| Tool | Purpose |
|---|---|
| `kubectl` | Kubernetes CLI |
| Kubernetes cluster | Minikube, kind, or a cloud cluster |
| NGINX Ingress Controller | Handles Ingress resources |
| Argo CD | Installed in the `argocd` namespace |

---

## Setup

### 1. Fork and clone this repository

```bash
git clone https://github.com/YOUR-USERNAME/k8s-nginx-ha-argocd.git
cd k8s-nginx-ha-argocd
```

### 2. Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

### 3. Register the Argo CD Application

```bash
kubectl apply -f argocd/application.yaml
```

Argo CD will detect the `app/` manifests and sync them into the cluster automatically.

### 4. Verify the deployment

```bash
kubectl get all -n web-app
```

You should see the Deployment with 2/2 pods running, the Service, and the Ingress.

### 5. Access the application

**Minikube:**
```bash
minikube addons enable ingress
minikube tunnel
```
Then open [http://localhost](http://localhost).

**Cloud cluster:**
```bash
kubectl get ingress -n web-app
# Use the EXTERNAL-IP shown
```

---

## GitOps Workflow

Any change pushed to the `main` branch is automatically applied to the cluster by Argo CD.

**Example — update the HTML page:**

```bash
# 1. Edit the content in app/configmap.yaml
# 2. Commit and push
git add app/configmap.yaml
git commit -m "Update homepage content"
git push
```

Argo CD detects the change and syncs within ~3 minutes (or immediately with reduced polling).

---

## Troubleshooting

**Pods not starting**
```bash
kubectl describe pod -n web-app
kubectl logs -n web-app -l app=nginx
```

**Argo CD not syncing**
```bash
kubectl get application -n argocd nginx-app
kubectl describe application -n argocd nginx-app
```

**Ingress not reachable**
```bash
kubectl get ingress -n web-app
kubectl describe ingress nginx-ingress -n web-app
```

---

## License

MIT
