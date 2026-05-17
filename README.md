# Argo CD Demo App

This repository contains a small Kubernetes demo application managed with
Argo CD. The app deploys an Nginx workload into the `demo-app` namespace.

## Repository Layout

```text
.
|-- k8s
|   |-- deployment.yaml
|   |-- namespace.yaml
|   `-- service.yaml
`-- README.md
```

## Kubernetes Resources

The manifests in `k8s/` create:

- `Namespace/demo-app`
- `Deployment/nginx-deployment` with 2 Nginx replicas
- `Service/nginx-service` as a `ClusterIP` service on port 80

## Prerequisites

Install and configure:

- Docker
- Minikube
- kubectl
- Argo CD CLI

Start Minikube:

```bash
minikube start
```

Verify the cluster:

```bash
kubectl get nodes
```

## Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all Argo CD pods to become ready:

```bash
kubectl get pods -n argocd
```

## Access Argo CD

Forward the Argo CD API/UI service to your machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open the UI:

```text
https://localhost:8080
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o go-template='{{.data.password | base64decode}}{{"\n"}}'
```

Log in with the CLI:

```bash
argocd login localhost:8080 --username admin --password <PASSWORD> --insecure
```

When using a local port-forward, add `--grpc-web` to Argo CD CLI commands.

## Create the Argo CD Application

Create the app from this repository:

```bash
argocd --grpc-web app create nginx-app \
  --repo https://github.com/Copubah/argo-cd.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo-app
```

Sync the app:

```bash
argocd --grpc-web app sync nginx-app
```

Check the app status:

```bash
argocd --grpc-web app get nginx-app
```

Expected result:

```text
Sync Status:   Synced
Health Status: Healthy
```

## Verify the Demo App

Check the deployed resources:

```bash
kubectl get all -n demo-app
```

Expected pods:

```text
nginx-deployment-...   1/1   Running
nginx-deployment-...   1/1   Running
```

## Troubleshooting

### Argo CD server address unspecified

Log in to the Argo CD CLI first:

```bash
argocd login localhost:8080 --username admin --password <PASSWORD> --insecure
```

Then run commands with `--grpc-web`:

```bash
argocd --grpc-web app sync nginx-app
```

### Pod is Pending during port-forward

Check pod status and events:

```bash
kubectl get pods -n argocd
kubectl get events -n argocd --sort-by=.lastTimestamp
```

If `kube-proxy` is failing with `too many open files`, raise the Minikube
node inotify limits:

```bash
minikube ssh -- sudo sysctl -w \
  fs.inotify.max_user_watches=524288 \
  fs.inotify.max_user_instances=512
```

Restart `kube-proxy`:

```bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

## Cleanup

Delete the Argo CD application:

```bash
argocd --grpc-web app delete nginx-app
```

Delete the demo namespace:

```bash
kubectl delete namespace demo-app
```

Stop Minikube:

```bash
minikube stop
```
