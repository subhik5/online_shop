# 🚀 COMPLETE DEVOPS GUIDE (Docker → Kubernetes → Ingress → HPA)

---

# 🧠 WHAT ARE WE BUILDING?

We are taking a simple app and:

1. Run it using Docker 🐳
2. Deploy it on Kubernetes ☸️
3. Manage config (ConfigMap) ⚙️
4. Secure data (Secrets) 🔐
5. Expose app (Ingress) 🌐
6. Auto scale (HPA) ⚡
7. Test load (Load Testing) 🔥

---

# 🌳 KUBERNETES WORKFLOW (VERY IMPORTANT)

```text
Namespace
   ↓
Deployment
   ↓
Service
   ↓
ConfigMap + Secret
   ↓
Ingress
   ↓
HPA
```

👉 Always follow this order

---

# 🧰 STEP 0: INSTALL TOOLS

## 🐳 Docker

### What?

Docker helps us package our app into a container.

### Why?

So it runs the same everywhere.

### Install:

```bash
sudo apt update -y
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
```

---

## ☸️ Kubernetes (kubectl)

### What?

Tool to talk to Kubernetes.

### Why?

To create and manage deployments.

### Install:

```bash
curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

---

## 🧪 Kind (Kubernetes Cluster)

### What?

Creates local Kubernetes cluster.

### Why?

We need a cluster to run our app.

### Install:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

---

## 🚀 Create Cluster

```bash
kind create cluster
kubectl get nodes
```

---

# 🐳 STEP 1: DOCKERIZE APP

## What?

Create image of your app.

## Why?

Kubernetes runs containers, not raw code.

---

## Dockerfile

```Dockerfile
FROM node:20-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Build & Run

```bash
docker build -t online-shop:v2 .
docker run -d -p 80:80 online-shop:v2
```

---

# ☸️ STEP 2: NAMESPACE

## What?

Namespace = folder inside Kubernetes.

## Why?

To organize resources.

---

## namespace.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: online-shop-prod
```

```bash
kubectl apply -f namespace.yml
```

---

# 🚀 STEP 3: DEPLOYMENT

## What?

Deployment runs your app (pods).

## Why?

To manage replicas and updates.

---

## deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: online-shop
  namespace: online-shop-prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: online-shop
  template:
    metadata:
      labels:
        app: online-shop
    spec:
      containers:
      - name: online-shop
        image: subhik6/online-shop:v2
        ports:
        - containerPort: 80
```

---

# 🌐 STEP 4: SERVICE

## What?

Service exposes pods.

## Why?

Pods change IP, service gives stable access.

---

## service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: online-shop-service
  namespace: online-shop-prod
spec:
  type: NodePort
  selector:
    app: online-shop
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

---

# ⚙️ STEP 5: CONFIGMAP

## What?

Stores configuration.

## Why?

Avoid hardcoding values.

---

## configmap.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: online-shop-config
  namespace: online-shop-prod
data:
  APP_NAME: "OnlineShop"
  ENV: "production"
```

---

# 🔐 STEP 6: SECRET

## What?

Stores sensitive data.

## Why?

Passwords should not be visible.

---

## Encode values

```bash
echo -n "admin" | base64
echo -n "mypassword123" | base64
```

---

## secret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: online-shop-secret
  namespace: online-shop-prod
type: Opaque
data:
  DB_USER: YWRtaW4=
  DB_PASS: bXlwYXNzd29yZDEyMw==
```

---

# 🔗 STEP 7: CONNECT CONFIG + SECRET

Add in deployment:

```yaml
env:
- name: APP_NAME
  valueFrom:
    configMapKeyRef:
      name: online-shop-config
      key: APP_NAME
- name: ENV
  valueFrom:
    configMapKeyRef:
      name: online-shop-config
      key: ENV
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: online-shop-secret
      key: DB_USER
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: online-shop-secret
      key: DB_PASS
```

---

# 🌐 STEP 8: INGRESS

## What?

Ingress = entry point for users.

## Why?

No need for NodePort, clean URL.

---

## Install Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: online-shop-ingress
  namespace: online-shop-prod
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: online-shop-service
            port:
              number: 80
```

---

# ⚡ STEP 9: HPA (AUTO SCALING)

## What?

Automatically increases pods.

## Why?

Handle traffic without manual work.

---

## Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Fix TLS

```bash
kubectl edit deployment metrics-server -n kube-system
```

Add:

```yaml
- --kubelet-insecure-tls
```

---

## hpa.yml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: online-shop-hpa
  namespace: online-shop-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: online-shop
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

# 🔥 STEP 10: LOAD TESTING

## What?

Simulates traffic.

## Why?

To test scaling.

---

```bash
sudo apt install apache2-utils -y
ab -n 1000 -c 50 http://<YOUR-IP>/
```

---

# 📊 VERIFY EVERYTHING

```bash
kubectl get pods -n online-shop-prod
kubectl get svc -n online-shop-prod
kubectl get ingress -n online-shop-prod
kubectl get hpa -n online-shop-prod
kubectl top nodes
```

---

# 🎉 FINAL RESULT

✅ App running
✅ Config managed
✅ Secrets secured
✅ Ingress working
✅ Auto scaling working

---

# 👨‍💻 AUTHOR

This project is created by **surbhi kaushik** for learning DevOps step-by-step.

---

