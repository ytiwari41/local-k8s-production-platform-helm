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
cluster/
  kind-config.yaml
ingress/
  ingress-nginx.yaml
observability/
  prometheus-values.yaml
README.md
```

---

## Notes

- Ensure Docker Desktop is running before starting.
- You can customize the values in the YAML files under `ingress/` and `observability/` as needed.
- For troubleshooting, use `kubectl get pods -A` and `kubectl logs`.

---