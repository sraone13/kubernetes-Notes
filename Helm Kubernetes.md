### 1️. What is Helm?

Helm is a package manager for Kubernetes.
It helps you install, upgrade, rollback, and uninstall Kubernetes applications easily using reusable packages called Charts.

Examples of apps installed using Helm:

NGINX
Argo CD
Prometheus
Grafana
AWS Load Balancer Controller

### 2️.Why Helm is Needed?

### Without Helm ❌

DevOps/SRE teams must:
Write multiple YAML files manually
Maintain scripts for each Kubernetes controller
Handle multiple versions separately
Repeat same configs across environments

### With Helm ✅
One command to install/update apps
Versioned deployments
Reusable templates
Environment-specific configuration
Easy rollback

### 👉 Single command replaces hundreds of YAML lines


### 3️.Key Helm Concepts

🔹 Chart:
A Helm Chart is a bundle/package of Kubernetes manifests.

🔹 Release:
A release is a running instance of a chart in a cluster.
Same chart can be installed multiple times using different release names.

🔹 Repository (Repo)
A repository stores packaged charts (.tgz files).

Example repos:
Bitnami
EKS charts
Custom internal repos

### 4️.Helm Chart Structure
mychart/
├── Chart.yaml        # Metadata (name, version, description)
├── values.yaml       # Default configuration values
├── templates/        # Kubernetes YAML templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
├── charts/           # Dependency charts (optional)

Chart.yaml contains:
Chart name
Version
App version
Maintainer

**Install Helm on Ubuntu (Recommended Method)**
### 🔹 Step 1: Download Helm install script
```
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 -o get_helm.sh
```
### 🔹 Step 2: Give execute permission
```
chmod +x get_helm.sh
```
### 🔹 Step 3: Install Helm
```
./get_helm.sh
```
### 🔹 Step 4: Verify installation
```
helm version
```
Expected output (example):
version.BuildInfo{Version:"v3.14.x", GoVersion:"go1.21.x"}


### 5️.Common Helm Commands for Repository Management
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
```
helm repo list
```
```
helm repo update
```
**Search Charts:**
```
helm search repo bitnami
```
```
helm search repo bitnami | grep -i argocd
```

### 6️.Installing Applications Using Helm
Install NGINX
```
helm install my-nginx bitnami/nginx
```
Install Prometheus
```
helm install prometheus bitnami/prometheus
```
**Install AWS Load Balancer Controller (EKS)**
```
helm repo add eks https://aws.github.io/eks-charts
```
```
helm repo update
```
```
helm install alb eks/aws-load-balancer-controller
```

### 7️.Upgrade, Rollback & Uninstall
Upgrade
```
helm upgrade my-nginx bitnami/nginx
```
Rollback
```
helm rollback my-nginx 1
```
Uninstall
```
helm uninstall my-nginx
helm uninstall prometheus
```

### 8️.Example: E-Commerce Project Using Helm Charts
Project Structure
best-commerce/
├── payments/
├── shipping/

Create Charts
```
mkdir -p best-commerce/{payments,shipping}
```
```
cd best-commerce
```
```
helm create payments
helm create shipping
```

### 9️.Modify Charts (Payments / Shipping)

Inside each Payments: 
templates/deployment.yaml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command:
            - sh
            - -c
            - "echo {{ .Values.appMessage }}; sleep 3600"
```		

Values.yaml:
```
image:
  repository: busybox
  tag: latest
  pullPolicy: IfNotPresent

appMessage: "Payments Service"
```


Chart.yaml
Keep as it is


Inside each Shipping: 
templates/deployment.yaml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command:
            - sh
            - -c
            - "echo {{ .Values.appMessage }}; sleep 3600"
```			

Values.yaml:
```
image:
  repository: busybox
  tag: latest
  pullPolicy: IfNotPresent

appMessage: "Shipping Service"
```

Chart.yaml
Keep as it is


### 10.Packaging Helm Charts run below commands from best-commerce
```
helm package payments
helm package shipping
```

✔ Creates .tgz files (zip-like packages)

### 11.Creating Helm Repository Index
```
helm repo index .
```
```
ls -ltr
```
```
cat index.yaml
```

index.yaml contains metadata for all charts

Repo can host multiple charts (payments & shipping)

1️.Publishing Helm Charts

Upload .tgz and index.yaml to:
GitHub Pages
S3 bucket
Artifactory / Nexus

Teams can add repo and reuse charts
helm repo add my-repo https://myrepo.example.com


Helm chartReference:
https://www.youtube.com/watch?v=7A5cH8iqgHU
https://github.com/iam-veeramalla/helm-zero-to-hero/blob/main/01-what-is-helm.md
