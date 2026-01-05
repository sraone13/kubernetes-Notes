# Kubernetes Deployment Strategies:

### 1.Why Deployment Strategies Are Important

Deployment strategies help release new application versions without downtime.
If you don’t follow a proper strategy, applications may face service interruption.

Example of Downtime (Without Strategy):

V1 has 4 replicas → uninstalling takes 5 minutes
V2 has 4 replicas → installing takes 5 minutes
Total downtime = ~10 minutes

This is unacceptable in production systems.

Solution: Use proper deployment strategies.

Types of Deployment Strategies:

**Rolling Update (Default in Kubernetes)**

**Canary Deployment**

**Blue–Green Deployment**

### 1️.Rolling Update Deployment

**Definition:**
Rolling Update is the default deployment strategy in Kubernetes.
It updates the application pod by pod instead of stopping all pods at once.
Ensures zero or near-zero downtime.

**How It Works:**
Old pods are terminated gradually.
New version pods are created one by one.
Traffic is always served because some pods are always running.

**Key Benefits:**
No downtime
Safe and controlled rollout
Easy rollback

**Important Parameters:**
maxUnavailable – Max number of pods that can be unavailable during update
maxSurge – Extra pods created temporarily during update
Example: 25% maxSurge

Commands (Rolling Update)
```
kubectl create deployment nginx --image=nginx
```

Watch pods updating:
```
kubectl get pods -w
```
Update image:
```
kubectl set image deployment/nginx nginx=nginx:1.22 --record
```

Check rollout status:
```
kubectl rollout status deployment/nginx
```

Rollback if something goes wrong:
```
kubectl rollout undo deployment/nginx
```

Check rollout history:
```
kubectl rollout history deployment/nginx
```
### 2️.Canary Deployment:

Canary deployment releases a small percentage of traffic to a new version (V2).
Used to test stability in real production traffic.

How It Works:
V1 (stable) → receives most traffic
V2 (new) → receives small traffic (e.g., 10%)
Traffic gradually increased if no issues found

Example Traffic Split
V1 → 90%
V2 → 10%

Benefits:
Real user feedback
Reduced blast radius
Avoids full application failure
Testing directly in production

Kubernetes Implementation (Using Ingress):

Kubernetes does not natively support Canary, but it can be implemented using:
Ingress Controller (NGINX)
Load Balancer
Annotations

Components Involved:

Deployment (V1 & V2)
Service (V1 & V2)
Ingress Controller
Ingress Resources

Steps for Canary Deployment:

Install NGINX Ingress Controller in the cluster

Create:

**Deployment + Service for V1 (Production.yaml)**
Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production
  labels:
    app: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: production
  template:
    metadata:
      labels:
        app: production
    spec:
      containers:
      - name: production
        image: registry.k8s.io/ingress-nginx/e2e-test-echo:v1.2.5@sha256:d7b3143152261e918e2197ef86840986ba7fbbd5ca72e0e79d6ec4ea99d2abc3
        ports:
        - containerPort: 80
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
```

Service
```
apiVersion: v1
kind: Service
metadata:
  name: production
  labels:
    app: production
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: production
```

**Deployment + Service for V2 (Canary.yaml)**

Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary
  labels:
    app: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: canary
  template:
    metadata:
      labels:
        app: canary
    spec:
      containers:
      - name: canary
        image: registry.k8s.io/ingress-nginx/e2e-test-echo:v1.2.5@sha256:d7b3143152261e918e2197ef86840986ba7fbbd5ca72e0e79d6ec4ea99d2abc3
        ports:
        - containerPort: 80
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
```

Service
```
apiVersion: v1
kind: Service
metadata:
  name: canary
  labels:
    app: canary
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: canary
```

**Create two Ingress resources:**
One for V1 (ingress-prod.yaml)

Ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
  annotations:
spec:
  ingressClassName: nginx
  rules:
  - host: echo.prod.mydomain.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: production
            port:
              number: 80

One for V2 (ingress-canary.yaml)
```

Ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    nginx.ingress.kubernetes.io/canary: \"true\"
    nginx.ingress.kubernetes.io/canary-weight: \"50\"
spec:
  ingressClassName: nginx
  rules:
  - host: echo.prod.mydomain.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: canary
            port:
              number: 80
```

Canary Ingress Annotation Example
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "10"

**Traffic Verification Command:**
```
for i in $(seq 1 10); do
  curl -s --resolve echo.prod.mydomain.com:80:<IP> echo.prod.mydomain.com | grep "Hostname"
done
```

You will see responses from both V1 and V2.

Gradual Traffic Increase

10% → 30% → 50% → 100%

Done over days or a week

Once stable, V2 becomes full production

### 3️.Blue–Green Deployment
Definition
Blue–Green is a zero-downtime deployment strategy.
Maintains two identical environments:

Blue → Current production (V1)
Green → New version (V2)

Example:

**1.Blue Deployment (Current Production)**

Blue-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      env: blue
  template:
    metadata:
      labels:
        app: myapp
        env: blue
    spec:
      containers:
      - name: myapp
        image: nginx:1.21
        ports:
        - containerPort: 80
```	
		
**2.Green Deployment (New Version)**

green-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
      env: green
  template:
    metadata:
      labels:
        app: myapp
        env: green
    spec:
      containers:
      - name: myapp
        image: nginx:1.22
        ports:
        - containerPort: 80	
```
**3.Service (Switch Selector to Change Traffic)**
Service pointing to Blue

traffic-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    env: blue
  ports:
  - port: 80
    targetPort: 80
```		
4.Switch Traffic to Green (Edit Service)
```
selector:
  app: myapp
  env: green
```

**Verify traffic steps:**

Step 1: Verify Both Environments Are Running
```
kubectl get pods -l app=myapp
```
Confirm:
Blue pods are running (env=blue)
Green pods are running (env=green)
No CrashLoopBackOff or Pending pods

2: Verify Pod Labels (Very Important)
```
kubectl get pods --show-labels
```

✔️Check:
Blue pods → env=blue
Green pods → env=green

3: Verify Service Selector (Traffic Controller)
```
kubectl get svc myapp-service -o yaml
```

✔️ Confirm selector:
Before switch (Blue live):
```
selector:
  app: myapp
  env: blue
```

After switch (Green live):
```
selector:
  app: myapp
  env: green
```
This step confirms which version is serving production traffic.

4: Verify Traffic Internally (Cluster)
```
for i in $(seq 1 10); do
  curl http://myapp-service
done
```


**Expected:**
All responses come from only one version
No mixed responses

All requests go to Blue OR Green
Zero errors

**How It Works:**

Blue (V1) serves all traffic
Green (V2) is deployed and fully tested
Load Balancer switches traffic from Blue → Green
If issues occur, instantly rollback to Blue

**Benefits:**
Instant rollback
Very safe
No partial traffic issues
Drawbacks
Costly
Both environments run simultaneously
Requires double resources

**Key Interview Points**

Rolling Update is default in Kubernetes
Canary & Blue–Green require Ingress or Load Balancer
Canary is best for real-time production testing
Blue–Green is safest but expensive
