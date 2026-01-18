# local-k8s-production-platform-helm

## Local Kubernetes Production Platform Setup

This guide will help you set up a local Kubernetes environment on your MacBook using [Kind](https://kind.sigs.k8s.io/), [Helm](https://helm.sh/), [ArgoCD](https://argo-cd.readthedocs.io/), and monitoring components (Prometheus, Grafana).

---

## Prerequisites

- **Docker Desktop**: [Install Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/) and ensure it is running.

---

## 1. Install Required Tools

```sh
brew install helm
brew install kind
```

---

## 2. Create a Kind Cluster

```sh
kind create cluster --name devops-lab --config cluster/kind-config.yaml
```
- The `cluster/kind-config.yaml` file defines your cluster configuration.

---

## 3. Install NGINX Ingress Controller

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  -f ingress/ingress-nginx.yaml
```
- The `ingress/ingress-nginx.yaml` file contains custom values for the ingress controller.

---

## 4. Install ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
- This sets up ArgoCD in the `argocd` namespace.

---

## 5. Install Monitoring Stack (Prometheus & Grafana)

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f observability/prometheus-values.yaml
```
- The `observability/prometheus-values.yaml` file contains your monitoring stack configuration.

---

## 6. Install Metrics Server

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 7. (Optional) Refresh ArgoCD Application

If you need to force ArgoCD to refresh the monitoring application:

```sh
kubectl annotate application monitoring \
  -n argocd \
  argocd.argoproj.io/refresh=hard
```

---

## 8. Access Grafana Dashboard

```sh
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```
- Open [http://localhost:3000](http://localhost:3000)
- Login: `admin` / `admin` (default, unless changed in your values file)

---

## 9. Access ArgoCD UI

- Get the ArgoCD admin password:

  ```sh
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

- Port-forward the ArgoCD server:

  ```sh
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```

- Open [https://localhost:8080](https://localhost:8080)
- Login: `admin` and the password from above

---

## Project Structure

```
local-k8s-production-platform-helm/
├── argocd/
│   └── app-of-apps.yaml        # Root ArgoCD application
├── apps/
│   ├── sample-app.yaml         # Example application
│   └── monitoring.yaml         # Monitoring (Prometheus + Grafana)
├── helm/
│   └── monitoring/
│       └── values.yaml         # Optional Helm values
└── README.md

```

---

## Notes

- Ensure Docker Desktop is running before starting.
- You can customize the values in the YAML files under `ingress/` and `observability/` as needed.
- For troubleshooting, use `kubectl get pods -A` and `kubectl logs`.

---

## 10. Deploy Sample App and Monitoring via ArgoCD

ArgoCD is used to manage and deploy both the sample application and monitoring components.  
You can define ArgoCD `Application` resources that point to your Helm charts or manifests.

### Example: Deploy Monitoring Stack via ArgoCD

Create an `Application` manifest (e.g., `argocd/monitoring-app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://prometheus-community.github.io/helm-charts'
    chart: kube-prometheus-stack
    targetRevision: 56.6.0 # use the desired version
    helm:
      valueFiles:
        - ../../observability/prometheus-values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:

```sh
kubectl apply -f argocd/monitoring-app.yaml
```

### Example: Deploy Sample App via ArgoCD

Create an `Application` manifest (e.g., `argocd/sample-app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/your-sample-app-repo.git'
    path: helm-chart
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: sample-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:

```sh
kubectl apply -f argocd/sample-app.yaml
```

# Force refresh in ArgoCD
kubectl annotate application platform-root -n argocd argocd.argoproj.io/refresh=hard

# Check status
kubectl get applications -n argocd

### Managing Applications

- After applying, ArgoCD will automatically deploy and manage these components.
- You can view and sync them from the ArgoCD UI (`https://localhost:8080`).

---