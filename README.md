## 🚀 MERN DevOps Deployment Guide
This guide provides a step-by-step process to deploy a MERN (MongoDB, Express.js, React, Node.js) application using Docker, Kubernetes, Helm, and Jenkins. It is designed for developers and DevOps engineers with intermediate knowledge of containerization and CI/CD pipelines. By the end, you'll have a fully functional MERN application running in a Kubernetes cluster with automated deployments.

#### 📑 Table of Contents
  - 📖 Introduction
  - 📋 Prerequisites
  - 🛠️ Repository Setup
  - 📂 Backend Repository Identification
  - 📂 Frontend Repository Identification
  - ⚙️ Backend Deployment (Express.js)
    - 🐳 Docker Deployment
    - ☸️ Kubernetes Deployment
  - 🖥️ Frontend Deployment (React)
    - 🐳 Docker Deployment
    - ☸️ Kubernetes Deployment
  - ⛵ Helm Charts
    - ⚙️ Backend Helm Configuration
    - 🖥️ Frontend Helm Configuration
  - 🤖 Jenkins CI/CD Pipeline
    - 📋 Prerequisites
    - 📚 Shared Library Setup
    - 🔐 Credentials Setup
    - ⚙️ Pipeline Configuration
  - 🛡️ Troubleshooting
  - 📜 License
  - 📞 Contact

#### 🔗 Prerequisites
Before you begin, ensure you have the following installed and configured:
- **Git** (to clone repositories)
- **Docker & Docker Compose**
- **kubectl** (Kubernetes CLI)
- **Helm**
- **Jenkins** (with required plugins and credentials)
- Access to a **Kubernetes cluster** (e.g., Minikube, EKS, GKE, KIND)

#### 🛠️ Repository Setup
```sh
git --version
docker --version
kubectl version --client
helm version
jenkins --version  # if installed locally
```

### 🌉 Create a Docker Network

Before running the services, create a Docker bridge network:

```bash
docker network create --driver bridge learner-insights
```

### 📛 Create a Kubernetes Namespace

Before deploying your services, it's a best practice to isolate them within a dedicated Kubernetes namespace.

A predefined namespace file is available at: `k8s/namespace.yml`
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: learner-insights
```
🚀 Apply the Namespace
Use the following command to create the namespace in your cluster:
```bash
kubectl apply -f k8s/namespace.yml
```

#### 📂 Backend Repository Identification
🔗 **Repository**: `https://github.com/psagar-dev/learner-insights-backend`

**Purpose:** This repository contains the backend code for the MERN application, built with Node.js and Express.js, handling API requests, MongoDB interactions, and authentication.

Technologies:

- **Node.js**: Runtime for the backend server.
- **Express.js**: Framework for building RESTful APIs.
- **MongoDB**: Database for storing application data.
- **Mongoose**: ODM for MongoDB.
- **JWT**: For authentication and authorization.
- **Jest** (optional): For unit testing.

```bash
git clone https://github.com/psagar-dev/learner-insights-backend.git
cd learner-insights-backend
npm install
npm start
```
#### 📂 Frontend Repository Identification
🔗 **Repository**: `https://github.com/psagar-dev/learner-insights-frontend`

**Purpose**: This repository contains the frontend code for the MERN application, built with React, providing a user interface for interacting with the backend APIs.

Technologies:

- **React**: Library for building the user interface.
- **Axios**: For making HTTP requests to the backend.
- **React Router**: For client-side routing.
- **Nginx**: For serving the production build.

```bash
git clone https://github.com/psagar-dev/learner-insights-frontend.git
cd learner-insights-frontend
npm install
npm run dev
```

## ⚙️ Backend -  Express.js(Node Js)

#### 🐳 Steps to Deploy an Application on Docker

📁 Create a `Dockerfile` inside the `learner-insights-backend` directory:
```Dockerfile
# Stage 1: Install Deps & build
FROM node:22-alpine AS builder

# Set the working directory in the container to /app
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy the rest of the application code to the working directory
COPY . .

# remove devDependencies
RUN npm prune --production

# Stage 2: Runtime: only prod artifacts on Alpine
#FROM node:lts-alpine as production
FROM node:22-alpine AS production

# Set the working directory in the container to /app
WORKDIR /app

# Install curl for healthchecks
RUN apk add --no-cache curl

# Copy only the necessary files from the builder stage
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app ./

# Build the application
EXPOSE 3001

# CMd to run the application
CMD ["node", "index.js"]
```

### 🔍 Local Testing & Validation

Build the Docker image:
```bash
docker image build --no-cache -t securelooper/learner-insights-backend:latest .
```
![learner insights backend build](images/learner-insights-backend/build.png)

Push the image to Docker Hub.
```bash
docker image push securelooper/learner-insights-backend:latest
```

Run the container:

```bash
docker container run -d -p 3001:3001 --name learner-insights-backend -e ATLAS_URI=mongodb://root:root@192.168.31.100:27017/learner-insights?authSource=admin -e HASH_KEY=thisIsMyHashKey -e JWT_SECRET_KEY=thisIsMyJwtSecretKey -e PORT=3001 --network learner-insights securelooper/learner-insights-backend
```

**Base URL:** `http://localhost:3001/health`

![learner insights backend build](images/learner-insights-backend/container-run-result.png)

### ☸️ Steps to Deploy an Application on Kubernetes

#### 🍃 MongoDB
If using a local MongoDB instance, follow these steps:

🗂️ 1. Create Secret
📄 **File**: `k8s/mongo/mongo-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
  namespace: mongodb
type: Opaque
data:
  MONGO_INITDB_ROOT_USERNAME: cm9vdA==
  MONGO_INITDB_ROOT_PASSWORD: cm9vdA==
```
📌 **Apply Secret**

```bash
kubectl apply -f k8s/mongo/mongo-secret.yaml
```
🔍 **Verify Secret**
```bash
kubectl get secret -n mongodb
```
![secret for learner insights backend](images/mongo-secret.png)

#### 📄 2 Create StatefulSet
**File:** `k8s/mongo/mongo.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mongodb
spec:
  serviceName: "mongo"         # headless service to enable stable DNS
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      terminationGracePeriodSeconds: 10 # recommended for MongoDB
      containers:
      - name: mongo
        image: mongo:latest           # using official MongoDB image
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db     # Mongo data directory
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: MONGO_INITDB_ROOT_USERNAME
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: MONGO_INITDB_ROOT_PASSWORD
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        livenessProbe:
          exec:
            command:
              - mongosh
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
              - mongosh
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

📌 **Apply StatefulSet**
```bash
kubectl apply -f k8s/mongo/mongo.yaml
```
🔍 **Verify Pods**
```bash
kubectl get pods -n mongodb
```
📜 **View Logs to Confirm Communication**
```
kubectl logs statefulset/mongo -n mongodb
```
OR
```
kubectl logs pod/<pod_name> -n mongodb
```
![Mongo DB StateFul Set](images/statefulset-mondb.png)

#### 📄 2 Create Service
**File:** `k8s/mongo/mongo-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongodb
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None                # This makes it headless
  selector:
    app: mongo
```

📌 **Apply Service**
```bash
kubectl apply -f k8s/mongo/mongo-service.yaml
```
🔍 **Verify Service**
```bash
kubectl get svc -n mongodb
```
![svc for learner insights backend](images/mongo-svc.png)
🔁 **Test Inter-Service Communication**

Run a shell inside the user pod:
```bash
kubectl exec -it statefulset/mongo -n mongodb -- sh
```
From inside the pod, test communication:
```bash
mongosh mongo-0.mongo.mongodb.svc.cluster.local:27017/testdb -u root -p root --authenticationDatabase admin
```
![Terminal for mongodb](images/Terminal-for-mongodb.png)

---

#### 🛠️ Backend
#### 🗂️ 1. Create Secret
📄 **File**: `k8s/backend/secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
  namespace: learner-insights
type: Opaque
data:
  ATLAS_URI: bW9uZ29kYjovL3Jvb3Q6cm9vdEBtb25nby0wLm1vbmdvLm1vbmdvZGIuc3ZjLmNsdXN0ZXIubG9jYWw6MjcwMTcvbGVhcm5lci1pbnNpZ2h0cz9hdXRoU291cmNlPWFkbWlu
  HASH_KEY: dGhpc0lzTXlIYXNoS2V5Cg==           
  JWT_SECRET_KEY: dGhpc0lzTXlKd3RTZWNyZXRLZXkK
  PORT: MzAwMQo=
```
📌 **Apply Secret**

```bash
kubectl apply -f k8s/backend/secret.yaml
```
🔍 **Verify Secret**
```bash
kubectl get secret -n learner-insights
```
![secret for learner insights backend](images/learner-insights-backend/secret-verify.png)

#### 📄 2 Create Deployment
**File:** `k8s/backend/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
    tier: backend
    environment: production
  namespace: learner-insights
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
        environment: production
    spec:
      restartPolicy: Always
      containers:
      - name: learner-insights-backend
        image: securelooper/learner-insights-backend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3001
        envFrom:
        - secretRef:
            name: backend-secrets
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 40
          periodSeconds: 5
```
📌 **Apply Deployment**
```bash
kubectl apply -f k8s/backend/deployment.yaml
```
🔍 **Verify Pods**
```bash
kubectl get pods -n learner-insights
```
📜 **View Logs to Confirm Communication**
```
kubectl logs deploy/backend -n learner-insights
```
OR
```
kubectl logs pod/<pod_name> -n learner-insights
```
![deploy & Logs for learner insights backend](images/learner-insights-backend/deployment-apply-logs.png)

#### 📄 3 Create Service
**File:** `k8s/backend/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: learner-insights
  labels:
    app: backend
    tier: backend
    environment: production
spec:
  selector:
    app: backend
  ports:
  - port: 3001
    targetPort: 3001
  type: LoadBalancer
```

📌 **Apply Service**
```bash
kubectl apply -f k8s/backend/service.yaml
```
🔍 **Verify Service**
```bash
kubectl get svc -n learner-insights
```
![svc for learner insights backend](images/learner-insights-backend/svc.png)
🔁 **Test Inter-Service Communication Using cURL**

Run a shell inside the user pod:
```
kubectl exec -it deploy/backend -n learner-insights -- sh
```
From inside the pod, test communication:
```
curl http://backend.learner-insights.svc.cluster.local:3001/health
```
![Communication test for learner insights backend](images/learner-insights-backend/text-communication-in-pod.png)

#### 🧪 4 Test with Port Forwarding
```bash
kubectl port-forward service/backend -n=learner-insights 3001:3001 --address=0.0.0.0
```
🌍 Access the service in your browser or tool like Postman:
run it
```bash
http://localhost:3001/health
```
![Communication test for learner insights backend](images/learner-insights-backend/container-run-result.png)

## 📦 Frontend -  ReactJs

#### 🐳 Steps to Deploy an Application on Docker

📁 Create a `Dockerfile` inside the `learner-insights-frontend` directory:
```Dockerfile
# STAGE 1: Build
FROM node:22-alpine AS builder

# Accept build-time API URL
ARG REACT_APP_API_BASE_URL
ENV REACT_APP_API_BASE_URL=$REACT_APP_API_BASE_URL

# Set the working directory in the container to /app
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy the rest of the application code to the working directory
COPY . .

RUN npm run build

# STAGE 2: Serve
FROM nginx:alpine

# Install curl for healthchecks
RUN apk add --no-cache curl

# Set the working directory in the container to /usr/share/nginx/html
RUN rm -rf /usr/share/nginx/html/*

# Copy only the necessary files from the builder stage
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom Nginx config
COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf

# Expose
EXPOSE 80

# Build the application
CMD ["nginx", "-g", "daemon off;"]
```

### 🔍 Local Testing & Validation

Build the Docker image:
```bash
docker image build --no-cache --build-arg REACT_APP_API_BASE_URL=http://192.168.31.100:3001 -t securelooper/learner-insights-frontend .
```
![learner insights frontend build](images/learner-insights-frontend/build.png)

Push the image to Docker Hub.
```bash
docker image push securelooper/learner-insights-frontend:latest
```

Run the container:

```bash
docker container run -d -p 3000:80 --name learner-insights-frontend --network learner-insights securelooper/learner-insights-frontend
```

**Base URL:** `http://localhost:3000`
![learner insights frontend Container](images/learner-insights-frontend/container-run-result-01.png)

![learner insights frontend Container](images/learner-insights-frontend/container-run-result.png)

### ☸️ Steps to Deploy an Application on Kubernetes

#### 📄 1 Create Deployment
**File:** `k8s/backend/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
    tier: frontend
    environment: production
  namespace: learner-insights
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
        environment: production
    spec:
      restartPolicy: Always
      containers:
      - name: learner-insights-frontend
        image: securelooper/learner-insights-frontend:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 40
          periodSeconds: 5
```
📌 **Apply Deployment**
```bash
kubectl apply -f k8s/frontend/deployment.yaml
```
🔍 **Verify Pods**
```bash
kubectl get pods -n learner-insights
```
📜 **View Logs to Confirm Communication**
```
kubectl logs deploy/frontend -n learner-insights
```
OR
```
kubectl logs pod/<pod_name> -n learner-insights
```
![learner insights frontend Container](images/learner-insights-frontend/deployment-apply-logs.png)

#### 📄 3 Create Service
**File:** `k8s/frontend/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: learner-insights
  labels:
    app: frontend
    tier: frontend
    environment: production
spec:
  selector:
    app: frontend
  ports:
  - port: 3000
    targetPort: 80
  type: LoadBalancer
```

📌 **Apply Service**
```bash
kubectl apply -f k8s/frontend/service.yaml
```
🔍 **Verify Service**
```bash
kubectl get svc -n learner-insights
```
![learner insights frontend SVC](images/learner-insights-frontend/svc.png)

🔁 **Test Inter-Service Communication Using cURL**

Run a shell inside the user pod:
```
kubectl exec -it deploy/frontend -n learner-insights -- sh
```
From inside the pod, test communication:
```
curl http://frontend.learner-insights.svc.cluster.local:3000
```
![learner insights frontend SVC](images/learner-insights-frontend/text-communication-in-pod.png)
#### 🧪 4 Test with Port Forwarding
```bash
kubectl port-forward service/frontend -n=learner-insights 3000:3000 --address=0.0.0.0
```
🌍 Access the service in your browser or tool like Postman:
run it
```bash
http://localhost:3000
```
![learner insights frontend Container](images/learner-insights-frontend/container-run-result-01.png)

![learner insights frontend Container](images/learner-insights-frontend/container-run-result.png)

## ⛵ Automated Deploy with Helm Chart

### 📂 Helm Chart Structure

```
helm/
├── backend/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── secret.yaml
└── frontend/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
```
Create charts:
```
helm create backend
helm create frontend
```

### 🔧 Backend Helm Configuration

#### 1. Create Secret
📄 **File:** `helm/backend/templates/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secretRefName }}
type: Opaque
data:
  ATLAS_URI: {{ .Values.secret.atlasUri }}
  HASH_KEY: {{ .Values.secret.hashKey }}        
  JWT_SECRET_KEY: {{ .Values.secret.jwtSecretKey }}
  PORT: {{ .Values.secret.port }}
```

####  2. Create Deployment
📄 **File:** `helm/backend/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "learner-insights.fullname" . }}-backend
  labels:
    app: {{ include "learner-insights.name" . }}-backend
    tier: {{ include "learner-insights.name" . }}-backend
    environment: {{ .Values.environment }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "learner-insights.name" . }}-backend
  template:
    metadata:
      labels:
        app: {{ include "learner-insights.name" . }}-backend
        tier: {{ include "learner-insights.name" . }}-backend
        environment: {{ .Values.environment }}
    spec:
      restartPolicy: {{ .Values.restartPolicy }}
      containers:
      - name: {{ include "learner-insights.name" . }}-backend
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ required "containerPort is required!" .Values.containerPort | int }}
        envFrom:
        - secretRef:
            name: {{ .Values.secretRefName }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.livenessProbe }}
        livenessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.readinessProbe }}
        readinessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
```

#### 3. Create Service
📄 **File:** `helm/backend/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "learner-insights.fullname" . }}-backend
  labels:
    app: {{ include "learner-insights.name" . }}-backend
    tier: {{ include "learner-insights.name" . }}-backend
    environment: {{ .Values.environment }}
spec:
  selector:
    app: {{ include "learner-insights.name" . }}-backend
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
  type: {{ .Values.service.type }}
```

#### 4. Create Helpers
📄 **File:** `helm/backend/templates/_helpers.tpl`

```tpl
{{- define "learner-insights.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "learner-insights.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end }}
{{- end }}
```

#### 5. Update Values
📄 **File:** `helm/backend/values.yaml`

```yaml
# The environment (e.g., production, staging, development)
environment: production
nameOverride: "learner-insights"
fullnameOverride: "learner-insights"

# Docker image settings
image:
  repository: securelooper/learner-insights-backend
  tag: latest # Change to a specific version/tag in production!
  pullPolicy: Always

# Number of backend pods to run
replicaCount: 2

# Pod container settings
containerPort: 3001
restartPolicy: Always

# Kubernetes Service for the backend
service: 
  type: LoadBalancer
  port: 3001
  targetPort: 3001
      
# Kubernetes Secret Reference (if you store secrets in K8s)
secretRefName: backend-secrets

  # Direct secrets (prefer using sealed-secrets or external secrets in production)
secret:
  atlasUri: bW9uZ29kYjovL3Jvb3Q6cm9vdEAxOTIuMTY4LjMxLjEwMDoyNzAxNy9sZWFybmVyLWluc2lnaHRzP2F1dGhTb3VyY2U9YWRtaW4K
  hashKey: dGhpc0lzTXlIYXNoS2V5Cg==
  jwtSecretKey: dGhpc0lzTXlKd3RTZWNyZXRLZXkK
  port: MzAwMQo=

# Liveness probe for backend health
livenessProbe:
  httpGet:
    path: /health
    port: 3001
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 2

# Readiness probe for backend readiness
readinessProbe:
  httpGet:
    path: /health
    port: 3001
  initialDelaySeconds: 40
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 2

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

🧪 Test Chart:
```
helm upgrade --install learner-insights-backend helm/backend -f helm/backend/values.yaml --namespace learner-insights --create-namespace
```

### 🔧 Frontend Helm Configuration

#### 1 Create Deployment
📄 **File:** `helm/frontend/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "learner-insights.fullname" . }}-frontend
  labels:
    app: {{ include "learner-insights.name" . }}-frontend
    tier: {{ include "learner-insights.name" . }}-frontend
    environment: {{ .Values.environment }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "learner-insights.name" . }}-frontend
  template:
    metadata:
      labels:
        app: {{ include "learner-insights.name" . }}-frontend
        tier: {{ include "learner-insights.name" . }}-frontend
        environment: {{ .Values.environment }}
    spec:
      restartPolicy: {{ .Values.restartPolicy }}
      containers:
      - name: {{ include "learner-insights.name" . }}-frontend
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
          - containerPort: {{ .Values.containerPort }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.livenessProbe }}
        livenessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.readinessProbe }}
        readinessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
```
#### 2. Create Service
📄 **File:** `helm/frontend/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "learner-insights.fullname" . }}-frontend
  labels:
    app: {{ include "learner-insights.name" . }}-frontend
    tier: {{ include "learner-insights.name" . }}-frontend
    environment: {{ .Values.environment }}
spec:
  selector:
    app: {{ include "learner-insights.name" . }}-frontend
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
  type: {{ .Values.service.type }}
```

#### 3. Create Helpers
📄 **File:** `helm/frontend/templates/_helpers.tpl`
```tpl
{{- define "learner-insights.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "learner-insights.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end }}
{{- end }}
```

#### 4. Update Values
📄 **File:** `helm/frontend/values.yaml`

```yaml
# The environment (e.g., production, staging, development)
environment: production
nameOverride: "learner-insights"
fullnameOverride: "learner-insights"

# Docker image settings
image:
  repository: securelooper/learner-insights-frontend
  tag: latest
  pullPolicy: Always

replicaCount: 2

containerPort: 80
restartPolicy: Always

# Kubernetes Service for the frontend
service: 
  type: LoadBalancer
  port: 3000
  targetPort: 80
  
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 2

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 40
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 2
```
🧪 Test Chart:
```
helm upgrade --install learner-insights-frontend helm/frontend -f helm/frontend/values.yaml --namespace learner-insights --create-namespace
```
### 🛠️ Jenkins CI/CD Pipeline

### 📦 Prerequisite Plugin Installation

Before proceeding with the pipeline setup or deployment process, ensure the following plugins are installed in your CI/CD environment (e.g., Jenkins).

#### ✅ Required Plugins

| Plugin Name        | Purpose |
|--------------------|---------|
| **SSH Agent**      | Allows you to use SSH credentials to connect to remote servers during pipeline execution. |
| **Docker Pipeline**| Provides steps to build, run, and manage Docker containers as part of your pipeline. |

### 🔧 Installation Steps (Jenkins Example)

1. **Open Jenkins Dashboard**
2. Navigate to **Manage Jenkins > Manage Plugins**
3. Go to the **Available** tab
4. Search for the following plugins:
   - `Docker Pipeline`
5. Select both plugins and click **Install without restart**
6. (Optional) Restart Jenkins if required

### 🧪 Verify Installation

1. Go to **Manage Jenkins > System Information**
2. Search for the plugin names to confirm they are listed and active.

---

### 🛠️ Shared Library Setup

To configure the global library in Jenkins:

1. Navigate to your Jenkins **Dashboard**.
2. Click on **Manage Jenkins**.
3. Select **System**.
4. Scroll down to the section labeled **Global Trusted Pipeline Libraries**.
5. Click **Add** and fill out the fields as follows:

### Library Configuration

| Field                        | Value                                                                 |
|-----------------------------|-----------------------------------------------------------------------|
| **Name**                    | `Shared`                                                              |
| **Default Version**         | `main` **(Branch name)**                                              |
| **Retrieval Method**        | `Modern SCM`                                                          |
| **Source Code Management**  | `Git`                                                                 |
| **Project Repository**      | `https://github.com/psagar-dev/jenkins-shared-libraries.git`          |
| **Credentials**             | `psagar-dev/******` *(Select your GitHub credentials if the repo is private)* |

Once the library is configured, you can load it in your `Jenkinsfile` using the following syntax:

```groovy
@Library('Shared') _
```

This will enable you to use the shared functions and utilities defined in the library across your Jenkins pipelines.

---

### 🔐 Credentials Setup (Required)

Please add the following credentials in Jenkins:

| Store  | Domain   | ID                | Name          | Functionality                      |
|--------|----------|-------------------|---------------|------------------------------------|
| System | (global) | Github            | ***** / ****** | Used for accessing GitHub repos in pipelines – [How to add this](https://github.com/psagar-dev/cheatsheet/blob/main/Jenkins/credentials-username-password.md) |
| System | (global) | docker-hub-creds  | ***** / ****** | Authenticates Docker Hub for image push/pull – [How to add this](https://github.com/psagar-dev/cheatsheet/blob/main/Jenkins/credentials-username-password.md) |

### 🔗 Setup GitHub Webhook:

- Go to your GitHub repository > Settings > Webhooks.
- Add webhook: `http://your-server-ip-or-domain/github-webhook/`
- Set content type: application/json.
- Select Push events.

### 🚀 Pipeline Steps in Jenkins
To set up the CI/CD pipelines for the MERN application, create three Jenkins pipelines: one for the backend, one for the frontend, and one for Helm deployment. Below are the steps to configure each pipeline in Jenkins.

##### 📌 1. Backend Pipeline
1.1. **Create a New Pipeline**
  - Go to `Jenkins Dashboard > New Item`.
  - Enter a name (e.g., `learner-insights-backend`).
  - Select **Pipeline** and click **OK**.

1.2. **Triggers**
  - GitHub hook trigger for GITScm polling

1.3. **Configure Pipeline**
  - In the pipeline configuration:
    - **Definition:** Select `Pipeline script from SCM`
    - **SCM:** Git.
    - **Repository URL**: URL of the backend repository (e.g., https://github.com/psagar-dev/learner-insights-backend.git).
    - **Credentials**: Add or Select Git credentials if required.
    - **Script Path**: Jenkinsfile.
    - **Branches to build**: `*/main`
1.4. **Script Path**: `Jenkinsfile`

**Save** the configuration.

![Configuration backend](images/learner-insights-backend/configuration-jenkinsfile.png)

#### 📌 2. Frontend Pipeline
2.1. **Create a New Pipeline**
  - Go to `Jenkins Dashboard > New Item`.
  - Enter a name (e.g., `learner-insights-frontend`).
  - Select **Pipeline** and click **OK**.

2.2. **Triggers**
  - GitHub hook trigger for GITScm polling

2.3. **Configure Pipeline**
  - In the pipeline configuration:
      - **Definition:** Select `Pipeline script from SCM`
      - **SCM:** Git.
      - **Repository URL**: URL of the frontend repository (e.g., https://github.com/psagar-dev/learner-insights-frontend.git).
      - **Credentials**: Add or Select Git credentials if required.
      - **Script Path**: Jenkinsfile.
      - **Branches to build**: `*/main`
3. **Script Path**: `Jenkinsfile`

**Save** the configuration.

![Configuration frontend](images/learner-insights-frontend/configuration-jenkinsfile.png)

#### 📌 3. Helm Deployment Pipeline
3.1. **Create a New Pipeline**
  - Go to `Jenkins Dashboard > New Item`.
  - Enter a name (e.g., `deploy-helm`).
  - Select **Pipeline** and click **OK**.

3.3. **Configure Pipeline**
  - In the pipeline configuration:
      - **Definition:** Select `Pipeline script from SCM`
      - **SCM:** Git.
      - **Repository URL**: URL of the frontend repository (e.g., git@github.com:psagar-dev/infra-learner-insights.git).
      - **Credentials**: Add or Select Git credentials if required.
      - **Script Path**: Jenkinsfile.
      - **Branches to build**: `*/main`
3. **Script Path**: `Jenkinsfile`

**Save** the configuration.
![Configuration deploy-helm](images/configuration-jenkinsfile.png)

##### 📄 Backend Jenkinsfile
📄 File: `Jenkinsfile` (Backend Repository)

```Jenkinsfile
@Library('Shared') _
def config = securityConfig("securelooper/learner-insights-backend:${BUILD_NUMBER}",'')

pipeline { 
    agent any
    stages {
        stage("Security Scans") {
            steps {
                script {
                    securityScan()
                }
            }
        }

        stage('Unit Test') {
            agent {
                docker {
                    image 'node:22-slim'
                }
            }
            steps {
                sh '''
                    npm install --cache .npm-cache
                    npm test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${config.DOCKER_IMAGE}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    pushDockerImage("${config.DOCKER_IMAGE}", "${config.DOCKER_CREDENTIALS_ID}")
                }
            }
        }

        stage('Trigger Deployment') {
            steps {
                echo "Triggering the deployment pipeline..."
                build job: 'deploy-helm',
                parameters: [
                    string(name: 'SERVICE_NAME', value: 'backend'), // Change to 'backend' for the backend repo
                    string(name: 'IMAGE_TAG', value: BUILD_NUMBER)
                ],
                wait: true // Proceeds without waiting for the deployment to finish
            }
        }
    }

    post {
        always {
            echo 'CI Pipeline finished.'
            // Clean up the Docker image from the Jenkins agent to save space.
            sh "docker rmi ${config.DOCKER_IMAGE}:${BUILD_NUMBER} || true"
        }
    }
}
```

#### 📄 Frontend Jenkinsfile
📄 File: `Jenkinsfile` (Frontend Repository)

```Jenkinsfile
@Library('Shared') _
def config = securityConfig("securelooper/learner-insights-frontend:${BUILD_NUMBER}",'')

pipeline { 
    agent any
    stages {
        stage("Security Scans") {
            steps {
                script {
                    securityScan()
                }
            }
        }

        stage('Unit Test') {
            agent {
                docker {
                    image 'node:22-slim'
                }
            }
            steps {
                sh '''
                    npm install --cache .npm-cache
                    npm test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(
                        "${config.DOCKER_IMAGE}",
                        "--build-arg REACT_APP_API_BASE_URL=${env.REACT_APP_API_BASE_URL} ."
                    )
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    pushDockerImage("${config.DOCKER_IMAGE}", "${config.DOCKER_CREDENTIALS_ID}")
                }
            }
        }

        stage('Trigger Deployment') {
            steps {
                echo "Triggering the deployment pipeline..."
                build job: 'deploy-helm',
                parameters: [
                    string(name: 'SERVICE_NAME', value: 'frontend'), // Change to 'frontend' for the frontend repo
                    string(name: 'IMAGE_TAG', value: BUILD_NUMBER)
                ],
                wait: true // Proceeds without waiting for the deployment to finish
            }
        }
    }

    post {
        always {
            echo 'CI Pipeline finished.'
            // Clean up the Docker image from the Jenkins agent to save space.
            sh "docker rmi ${config.DOCKER_IMAGE}:${BUILD_NUMBER} || true"
        }
    }
}
```

#### 📄 Helm Deployment Jenkinsfile
📄 **File**: `Jenkinsfile` (Infrastructure Repository)

```Jenkinsfile
pipeline { 
    agent any

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: 'frontend', description: 'The name of the service/chart to deploy')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'The Docker image tag to deploy')
    }

    stages {
        stage('Lint Helm Chart') {
            steps {
                script {
                    def serviceName = params.SERVICE_NAME
                    def chartPath = "./helm/${serviceName}"
                    echo "Linting Helm chart for ${serviceName}..."
                    sh "helm lint ${chartPath} -f ${chartPath}/values.yaml"
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                script {
                    def serviceName = params.SERVICE_NAME
                    def imageTag = params.IMAGE_TAG
                    def chartPath = "./helm/${serviceName}"

                    echo "Deploying ${serviceName} with image tag ${imageTag} using Helm..."

                    sh """
                        helm upgrade --install learner-insights-${serviceName} ${chartPath} \
                        -f ${chartPath}/values.yaml \
                        --set image.repository=securelooper/learner-insights-${serviceName} \
                        --set image.tag=${imageTag} \
                        --namespace learner-insights \
                        --create-namespace
                    """
                }
            }
        }
    }
}
```


### 🚀 Backend Pipeline Overview
![Backend Pipeline Overview](images/learner-insights-backend/pipeline-overview.png)

### 🚀 Frontend Pipeline Overview
![Frontend Pipeline Overview](images/learner-insights-frontend/pipeline-overview.png)

### 🚀 Infra Pipeline Overview
![Infra Pipeline Overview](images/pipeline-overview.png)

## Verify Helm List
- You can run this command
```
helm list -n learner-insights
```
![Helm List](images/helm-list.png)

Pod check it working
```
kubectl get pods -n learner-insights
```
![get Pods](images/get-node.png)

#### 🧪 Test with Port Forwarding
```bash
kubectl port-forward service/backend -n=learner-insights 3001:3001 --address=0.0.0.0
```

```bash
kubectl port-forward service/frontend -n=learner-insights 3000:3000 --address=0.0.0.0
```
🌍 Access the service in your browser or tool like Postman:
run it
```bash
http://localhost:3000
```

**Base URL:** `http://localhost:3000`
![learner insights frontend Container](images/learner-insights-frontend/container-run-result-01.png)

![learner insights frontend Container](images/learner-insights-frontend/container-run-result.png)


### 🛡️ Troubleshooting

- **Docker Build Fails**:
    - Ensure sufficient disk space and correct Node.js version (node:22-alpine).
    - Check logs: docker build --no-cache -t image:tag ..

- **Pod CrashLoopBackOff**:
    - Verify environment variables in secret.yaml.
    - Run: kubectl describe pod <pod-name> -n learner-insights.

- **Helm Upgrade Fails**:
    - Check syntax: helm lint helm/<chart> -f helm/<chart>/values.yaml.

- **Jenkins Pipeline Fails**:
    - Verify credentials in Jenkins.
    - Check logs in Jenkins UI for specific errors.

## 📜 Project Information

### 📄 License Details
This project is released under the MIT License, granting you the freedom to:
- 🔓 Use in commercial projects
- 🔄 Modify and redistribute
- 📚 Use as educational material

## 📞 Contact

📧 Email: [Email Me](securelooper@gmail.com
)
🔗 LinkedIn: [LinkedIn Profile](https://www.linkedin.com/in/sagar-93-patel)  
🐙 GitHub: [GitHub Profile](https://github.com/psagar-dev)  

---

<div align="center">
  <p>Built with ❤️ by Sagar Patel</p>
</div>