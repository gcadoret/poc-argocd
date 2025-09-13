
# POC ArgoCD on Kubernetes (k3d)

This project is a Proof of Concept (POC) to deploy a "hello" web application on a local Kubernetes cluster (via k3d) using Argo CD for a GitOps approach.

## Prerequisites

Before starting, make sure you have the following tools installed.

### macOS

```bash
# Install Homebrew if you haven't already: https://brew.sh
brew install k3d kubectl argocd git

# Make sure Docker Desktop is installed and running.
```

### Debian

```bash
# Docker
apt-get update && apt-get install -y curl git ca-certificates
curl -fsSL https://get.docker.com | sh
usermod -aG docker $USER # Log out and log back in

# kubectl
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mv kubectl /usr/local/bin/

# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# argocd CLI
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

## 1. Create the Kubernetes cluster

We use k3d to create a lightweight local Kubernetes cluster.

```bash
k3d cluster create demo   --servers 1 --agents 1   -p "80:80@loadbalancer" -p "443:443@loadbalancer"

kubectl cluster-info
```

## 2. Install Argo CD

Install Argo CD in its own namespace.

```bash
kubectl create namespace argocd
kubectl apply -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

To access the Argo CD web interface, use a port-forward:

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:80 &
```

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in with the Argo CD CLI:

```bash
argocd login localhost:8081 --username admin --password <YOUR_PASSWORD> --insecure
```

## 3. Prepare the Git repository

This project already contains the required manifests. You need to push it to a Git repository (GitHub, GitLab, etc.) so that Argo CD can track it.

### File structure

```
.
├── app
│   ├── deployment.yaml
│   └── service.yaml
└── argocd
    └── application.yaml
```

### Application manifests (`app/`)

#### `app/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels: { app: hello }
spec:
  replicas: 2
  selector:
    matchLabels: { app: hello }
  template:
    metadata:
      labels: { app: hello }
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
```

#### `app/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  labels: { app: hello }
spec:
  selector: { app: hello }
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

### Argo CD application manifest (`argocd/`)

#### `argocd/application.yaml`

Edit the `argocd/application.yaml` file to point to **your own Git repository**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_USER>/<YOUR_REPO>.git
    targetRevision: main
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## 4. Deploy the application with Argo CD

Apply the application manifest so that Argo CD starts synchronizing your application.

```bash
kubectl apply -f argocd/application.yaml
```

You can check the application status via the Argo CD UI (`http://localhost:8081`) or with the CLI:

```bash
argocd app get hello-app
argocd app sync hello-app
```

## 5. Test the application

Use a port-forward to access the `hello` service:

```bash
kubectl port-forward svc/hello 8080:80 &
```

In another terminal, send a `curl` request:

```bash
curl -i http://localhost:8080/
```

You should see a response from the NGINX server.

## 6. GitOps in action

To update the application, simply modify the manifests in your Git repository (for example, change the image version in `app/deployment.yaml`), commit, and push the changes. Argo CD will automatically detect the new version and update the deployment.

## 7. Cleanup

To delete the application and the cluster:

```bash
argocd app delete hello-app --yes
k3d cluster delete demo
# Don't forget to kill the background processes (port-forward)
```
