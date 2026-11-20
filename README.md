# Lab 11: End-to-End Microservices Deployment on K3s with ArgoCD & NGINX Ingress

This lab demonstrates an end-to-end microservices migration and GitOps continuous delivery workflow. It migrates the **Mental Health Booking App** from Docker Compose to Kubernetes using **K3s**, **NGINX Ingress Controller**, and **ArgoCD**.

The project maintains a production-grade yet straightforward microservices architecture demonstrating essential Kubernetes and GitOps concepts:

- Docker multi-stage builds (`node:22-alpine`, `nginx:1.28-alpine`)
- K3s lightweight Kubernetes cluster
- Declarative GitOps continuous delivery with ArgoCD
- Automated synchronization, self-healing, and resource pruning
- Kubernetes Namespace isolation (`smallsteps`)
- Configuration management with ConfigMaps and Secrets
- Stateless workload Deployments (`backend`, `frontend`)
- Stateful workload StatefulSet with PersistentVolumeClaim (`mongodb`)
- K3s `local-path` StorageClass persistence
- Headless Service for stateful DNS resolution (`mongodb-headless`)
- ClusterIP Services for internal service discovery (`backend-service`, `frontend-service`)
- Automated PostSync Smoke Test Job (`argocd.argoproj.io/hook: PostSync`)
- Scheduled batch processing with Kubernetes CronJob (`mongodb-backup`)
- High availability with Liveness, Readiness, and Startup Probes
- Zero-downtime Rolling Updates (`maxUnavailable: 0`, `maxSurge: 1`)
- Horizontal Pod Autoscaling (HPA) with ArgoCD `ignoreDifferences`
- NGINX Ingress Controller routing on custom local domain (`smallsteps.local`)

---

## 1. System Architecture

### 1.1 Architecture Diagram

```text
                         Browser
                            |
                            |
                   smallsteps.local
                            |
                            v
              +--------------------------+
              | NGINX Ingress Controller |
              |        Port 80            |
              +------------+-------------+
                           |
                 +---------+---------+
                 |                   |
               /api                  /
                 |                   |
                 v                   v
       +------------------+   +------------------+
       | backend-service  |   | frontend-service |
       | ClusterIP :3000  |   | ClusterIP :80    |
       +--------+---------+   +--------+---------+
                |                      |
                v                      v
       +------------------+   +------------------+
       | Backend Pods     |   | NGINX Pods       |
       | Deployment       |   | Deployment       |
       | 1 replica        |   | 1 replica        |
       +--------+---------+   +------------------+
                |
                v
       +----------------------+
       | mongodb-headless     |
       | Headless Service     |
       +----------+-----------+
                  |
                  v
       +----------------------+
       | MongoDB StatefulSet  |
       | mongodb-0            |
       +----------+-----------+
                  |
                  v
       +----------------------+
       | MongoDB PVC          |
       | 2Gi / local-path     |
       +----------------------+
```

### 1.2 Request Flow

```text
Browser
   |
   v
NGINX Ingress
   |
   +---- / ------------> frontend-service:80 (NGINX Static + Reverse Proxy)
   |
   +---- /api ---------> backend-service:3000 (Node.js Express API)
                              |
                              v
                       mongodb-headless:27017
                              |
                              v
                           MongoDB (mongodb-0)
```

---

## 2. Application Overview

The application is a full-stack mental health booking portal composed of:

```text
Frontend (Static HTML/CSS/JS)
   |
   v
NGINX Web Server / Reverse Proxy
   |
   v
Node.js / Express REST API
   |
   v
MongoDB Document Database
```

### 2.1 Application Ports

| Component | Container Port | Service Type | Service Port | Exposure |
|---|---|---|---|---|
| NGINX Ingress Controller | 80, 443 | LoadBalancer / HostPort | 80, 443 | External (`smallsteps.local`) |
| Frontend (NGINX) | 80 | ClusterIP (`frontend-service`) | 80 | Internal Cluster |
| Backend (Node.js / Express) | 3000 | ClusterIP (`backend-service`) | 3000 | Internal Cluster |
| MongoDB | 27017 | Headless (`mongodb-headless`) | 27017 | Internal Cluster |

### 2.2 API Endpoints

```text
/api/auth/*     - User authentication (registration, login)
/api/doctors    - Doctor directory and listings
/api/bookings   - Appointment bookings and management
/api/health     - Microservice health probe endpoint
```

---

## 3. Project Directory Structure

```text
mental-health-booking-app/
├── .dockerignore
├── .env.example
├── .gitignore
├── docker-compose.yml
├── package.json
├── package-lock.json
├── README.md
├── argocd/
│   └── application.yaml              # Declarative ArgoCD Application manifest
├── docker/
│   ├── Dockerfile                    # Multi-stage build (deps, backend, frontend)
│   └── nginx.conf                    # Frontend NGINX configuration
├── k8s/
│   ├── 00-namespace.yaml             # Dedicated 'smallsteps' namespace
│   ├── 01-configmap-secret.yaml      # Environment variables & DB credentials
│   ├── 02-mongodb-statefulset.yaml   # Headless service & StatefulSet (2Gi PVC)
│   ├── 03-backend-deployment.yaml    # Node.js backend deployment & ClusterIP
│   ├── 04-frontend-deployment.yaml   # NGINX frontend deployment & ClusterIP
│   ├── 05-ingress.yaml               # Ingress routing rules for smallsteps.local
│   ├── 06-smoke-test-job.yaml        # PostSync smoke test job hook
│   ├── 07-mongodb-backup-cronjob.yaml# Daily scheduled mongodump CronJob
│   └── 08-backend-hpa.yaml           # HorizontalPodAutoscaler for backend
├── public/
│   ├── index.html                    # Landing page
│   ├── login.html                    # Authentication page
│   ├── admin-dashboard.html          # Admin portal
│   ├── patient-dashboard.html        # Patient portal
│   ├── css/
│   │   └── style.css                 # Application styles
│   └── js/
│       ├── admin.js                  # Admin logic
│       ├── app.js                    # Core frontend client
│       ├── auth.js                   # JWT authentication handler
│       └── booking.js                # Booking interaction logic
└── src/
    └── server.js                     # Express server & API routes
```

---

## 4. Kubernetes Components Matrix

| Component | Kind | Port | Exposure | Storage | Health Probes |
|---|---|---|---|---|---|
| NGINX Ingress | Ingress | 80/443 | External (`smallsteps.local`) | None | Controller built-in |
| Frontend | Deployment | 80 | ClusterIP: 80 | None | Liveness, Readiness (`/nginx-health`) |
| Backend | Deployment | 3000 | ClusterIP: 3000 | None | Startup, Readiness, Liveness (`/api/health`) |
| MongoDB | StatefulSet | 27017 | Headless Service | 2Gi (`local-path`) | Startup, Readiness, Liveness (`mongosh ping`) |
| MongoDB DNS | Headless Service | 27017 | Cluster internal | None | Service DNS endpoints |
| Config | ConfigMap | - | Cluster internal | None | Mounted as env vars / config files |
| Credentials | Secret | - | Cluster internal | None | Mounted as env vars |
| Smoke Test | Job (PostSync) | - | Batch workload | None | Pod restartPolicy: Never |
| Backup | CronJob | - | Scheduled batch | Ephemeral `/tmp` | Daily at 02:00 AM (`0 2 * * *`) |
| Autoscaling | HPA | - | Autoscaling controller | None | Target 70% CPU (2 to 5 replicas) |

---

## 5. Docker Packaging & NGINX Configuration

### 5.1 Multi-Stage Dockerfile

File: `docker/Dockerfile`

The project utilizes a multi-stage Docker build to keep images minimal and secure:

```dockerfile
# syntax=docker/dockerfile:1

# ==================================================
# Backend Dependencies
# ==================================================
FROM node:22-alpine AS backend-deps

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev \
    && npm cache clean --force

# ==================================================
# Backend
# ==================================================
FROM node:22-alpine AS backend

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000

COPY --from=backend-deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node src ./src
COPY --chown=node:node public ./public

USER node
EXPOSE 3000

CMD ["node", "src/server.js"]

# ==================================================
# Frontend
# ==================================================
FROM nginx:1.28-alpine AS frontend

RUN rm -f /etc/nginx/conf.d/default.conf
COPY public/ /usr/share/nginx/html/
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 5.2 NGINX Web Server Configuration

File: `docker/nginx.conf`

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    server_tokens off;

    resolver 10.43.0.10 valid=10s ipv6=off;

    location = /nginx-health {
        access_log off;
        default_type text/plain;
        return 200 "OK\n";
    }

    location /api/ {
        set $backend_upstream http://backend-service.smallsteps.svc.cluster.local:3000;
        proxy_pass $backend_upstream;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## 6. Kubernetes Declarative Manifests

All manifests reside in the `k8s/` directory and are completely declarative and GitOps-ready.

### 6.1 Namespace (`k8s/00-namespace.yaml`)

Defines the isolated environment for the microservices stack:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: smallsteps
```

### 6.2 ConfigMap & Secret (`k8s/01-configmap-secret.yaml`)

Separates non-sensitive configurations from database credentials and cryptographic secrets:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: smallsteps-config
  namespace: smallsteps
data:
  NODE_ENV: "production"
  PORT: "3000"
  MONGO_DATABASE: "smallsteps"
---
apiVersion: v1
kind: Secret
metadata:
  name: smallsteps-secret
  namespace: smallsteps
type: Opaque
stringData:
  MONGO_ROOT_USERNAME: "smallsteps"
  MONGO_ROOT_PASSWORD: "SmallSteps@12345"
  JWT_SECRET: "CHANGE_THIS_TO_A_LONG_RANDOM_SECRET"
  MONGODB_URI: "mongodb://smallsteps:SmallSteps%4012345@mongodb-0.mongodb-headless.smallsteps.svc.cluster.local:27017/smallsteps?authSource=admin"
```

### 6.3 MongoDB StatefulSet & Headless Service (`k8s/02-mongodb-statefulset.yaml`)

Ensures ordered deployment, stable network identities, and persistent storage retention using K3s `local-path`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  namespace: smallsteps
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: smallsteps
spec:
  serviceName: mongodb-headless
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: mongodb
          image: mongo:7
          imagePullPolicy: IfNotPresent
          ports:
            - name: mongodb
              containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: smallsteps-secret
                  key: MONGO_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: smallsteps-secret
                  key: MONGO_ROOT_PASSWORD
            - name: MONGO_INITDB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: smallsteps-config
                  key: MONGO_DATABASE
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
          startupProbe:
            exec:
              command:
                - mongosh
                - --quiet
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 15
            failureThreshold: 30
          readinessProbe:
            exec:
              command:
                - mongosh
                - --quiet
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 15
            failureThreshold: 3
          livenessProbe:
            exec:
              command:
                - mongosh
                - --quiet
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 15
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: local-path
        resources:
          requests:
            storage: 2Gi
```

### 6.4 Backend Deployment & Service (`k8s/03-backend-deployment.yaml`)

Deploys the Node.js API with zero-downtime rolling updates and three-tier health probes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: smallsteps
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: smallsteps-backend:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: NODE_ENV
              valueFrom:
                configMapKeyRef:
                  name: smallsteps-config
                  key: NODE_ENV
            - name: PORT
              valueFrom:
                configMapKeyRef:
                  name: smallsteps-config
                  key: PORT
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: smallsteps-secret
                  key: JWT_SECRET
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: smallsteps-secret
                  key: MONGODB_URI
          startupProbe:
            httpGet:
              path: /api/health
              port: http
            failureThreshold: 30
            periodSeconds: 5
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /api/health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            httpGet:
              path: /api/health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: smallsteps
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 3000
      targetPort: http
```

### 6.5 Frontend Deployment & Service (`k8s/04-frontend-deployment.yaml`)

Serves static assets through NGINX and proxies API calls to `backend-service`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: smallsteps
data:
  default.conf: |
    server {
        listen 80;
        server_name _;

        root /usr/share/nginx/html;
        index index.html;

        server_tokens off;

        resolver 10.43.0.10 valid=10s ipv6=off;

        location = /nginx-health {
            access_log off;
            default_type text/plain;
            return 200 "OK\n";
        }

        location /api/ {
            set $backend_upstream http://backend-service.smallsteps.svc.cluster.local:3000;
            proxy_pass $backend_upstream;
            proxy_http_version 1.1;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;
            proxy_send_timeout 60s;
        }

        location / {
            try_files $uri $uri/ /index.html;
        }

        location ~ /\. {
            deny all;
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: smallsteps
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: smallsteps-frontend:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
              readOnly: true
          readinessProbe:
            httpGet:
              path: /nginx-health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            httpGet:
              path: /nginx-health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 20
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: smallsteps
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```

### 6.6 NGINX Ingress Routing (`k8s/05-ingress.yaml`)

Exposes services externally using virtual host routing on `smallsteps.local`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: smallsteps-ingress
  namespace: smallsteps
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
    - host: smallsteps.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### 6.7 PostSync Smoke Test Job (`k8s/06-smoke-test-job.yaml`)

Executes post-deployment automated health validation via ArgoCD hooks:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smallsteps-smoke-test
  namespace: smallsteps
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 5
  template:
    metadata:
      labels:
        app: smallsteps-smoke-test
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: curlimages/curl:8.10.1
          command:
            - sh
            - -c
          args:
            - |
              set -e
              echo "Testing backend..."
              curl -f http://backend-service:3000/api/health
              echo ""
              echo "Backend test passed."
              echo "Testing frontend..."
              curl -f http://frontend-service/nginx-health
              echo ""
              echo "Frontend test passed."
              echo ""
              echo "All smoke tests passed."
```

### 6.8 MongoDB Backup CronJob (`k8s/07-mongodb-backup-cronjob.yaml`)

Performs daily database backups at 02:00 AM using `mongodump`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: smallsteps
spec:
  schedule: "0 2 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: mongodb-backup
              image: mongo:7
              command:
                - /bin/sh
                - -c
              args:
                - |
                  set -e
                  DATE=$(date +%Y-%m-%d-%H%M)
                  echo "Starting MongoDB backup: $DATE"
                  mongodump \
                    --host mongodb-0.mongodb-headless.smallsteps.svc.cluster.local \
                    --port 27017 \
                    --username "$MONGO_USERNAME" \
                    --password "$MONGO_PASSWORD" \
                    --authenticationDatabase admin \
                    --db smallsteps \
                    --archive="/tmp/smallsteps-$DATE.archive"
                  echo "MongoDB backup completed."
              env:
                - name: MONGO_USERNAME
                  valueFrom:
                    secretKeyRef:
                      name: smallsteps-secret
                      key: MONGO_ROOT_USERNAME
                - name: MONGO_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: smallsteps-secret
                      key: MONGO_ROOT_PASSWORD
```

### 6.9 Backend Horizontal Pod Autoscaler (`k8s/08-backend-hpa.yaml`)

Dynamically scales the Node.js backend pods between 2 and 5 replicas based on CPU load:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: smallsteps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 1
  maxReplicas: 2
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 7. Cluster Preparation & Ingress Setup

### 7.1 Verify K3s Installation & StorageClass

Check K3s version and node status:

```bash
k3s --version
kubectl get nodes -o wide
```

Check the K3s default StorageClass:

```bash
kubectl get storageclass
```

Expected output:

```text
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  ...
```

The MongoDB PVC will bind dynamically to `local-path`.

### 7.2 Disable Traefik Ingress Controller

K3s installs Traefik by default. Disable it to use the standard NGINX Ingress Controller:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo nano /etc/rancher/k3s/config.yaml
```

Add the following configuration:

```yaml
disable:
  - traefik
```

Restart K3s to apply changes:

```bash
sudo systemctl restart k3s
```

Verify Traefik pods are terminating or removed:

```bash
kubectl get pods -n kube-system
```

### 7.3 Install NGINX Ingress Controller via Helm

Add the official ingress-nginx Helm repository:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install the controller:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

Verify that the Ingress Controller is running and the IngressClass exists:

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

Expected IngressClass output:

```text
NAME    CONTROLLER                      PARAMETERS   AGE
nginx   k8s.io/ingress-nginx            <none>       ...
```

---

## 8. Building & Importing Container Images

### 8.1 Build Docker Images Locally

From the root of the project directory (`mental-health-booking-app`):

```bash
# Build the Node.js backend image
docker build \
  -f docker/Dockerfile \
  --target backend \
  -t smallsteps-backend:1.0 \
  .

# Build the NGINX frontend image
docker build \
  -f docker/Dockerfile \
  --target frontend \
  -t smallsteps-frontend:1.0 \
  .
```

Verify the built images:

```bash
docker images | grep smallsteps
```

### 8.2 Import Images into K3s Containerd

Because K3s uses an internal `containerd` runtime, local Docker images must be saved and imported into the `k8s.io` namespace:

```bash
# Save images to tar archives
docker save smallsteps-backend:1.0 -o /tmp/smallsteps-backend.tar
docker save smallsteps-frontend:1.0 -o /tmp/smallsteps-frontend.tar

# Import images into K3s containerd
sudo k3s ctr -n k8s.io images import /tmp/smallsteps-backend.tar
sudo k3s ctr -n k8s.io images import /tmp/smallsteps-frontend.tar

# Verify containerd image cache
sudo k3s ctr -n k8s.io images list | grep smallsteps
```

---

## 9. GitOps Deployment with ArgoCD

GitOps uses Git as the single source of truth for declarative infrastructure and applications. ArgoCD continuously monitors the Git repository and automatically synchronizes the live cluster state with the desired state in Git.

### 9.1 GitOps Architecture & Continuous Delivery Flow

```text
+-----------------------------------------------------------+
| Git Repository (Single Source of Truth)                   |
| https://github.com/MohanadHesham98/k8s-labs.git           |
| Path: "Lab 11: End-to-End Microservices Deployment /k8s" |
+-----------------------------+-----------------------------+
                              |
                              | Git Poll / Webhook (Continuous Monitoring)
                              v
                +----------------------------+
                |     ArgoCD Controller      |
                |   (Namespace: argocd)      |
                +--------------+-------------+
                               |
               Automated Sync  |  Self-Healing (Drift Correction)
                               v
+-----------------------------------------------------------+
| K3s Cluster - Namespace: smallsteps                       |
|                                                           |
| [ConfigMap] [Secret] [Ingress]                            |
|       |                                                   |
|       v                                                   |
| [Deployments: backend (1-2 pods) / frontend (1 pod)]      |
|       |                                                   |
|       v                                                   |
| [StatefulSet: mongodb-0 + 2Gi PVC]                        |
|       |                                                   |
|       v (PostSync Hook)                                   |
| [Smoke Test Job: curl backend & frontend health]          |
+-----------------------------------------------------------+
```

### 9.2 ArgoCD Prerequisites

1. ArgoCD must be installed and running on your K3s cluster:

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. Confirm all ArgoCD components are running:

   ```bash
   kubectl get pods -n argocd
   ```

### 9.3 Declarative ArgoCD Application Manifest

File: `argocd/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: smallsteps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/MohanadHesham98/End-to-End-Microservices-Deployment-on-K3s-with-ArgoCD-NGINX-Ingress.git
    targetRevision: main
    path: k8s

  destination:
    server: https://kubernetes.default.svc
    namespace: smallsteps

  # The HPA owns the backend replica count. Without this rule, self-healing
  # would reset replicas and fight the autoscaler.
  ignoreDifferences:
    - group: apps
      kind: Deployment
      name: backend
      jsonPointers:
        - /spec/replicas

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Key Application Configurations

- **`metadata.finalizers`**: Protects against orphan resources. Deleting the ArgoCD Application cascades deletion to all managed Kubernetes workloads.
- **`source.repoURL`**: Points directly to the GitOps repository (`https://github.com/MohanadHesham98/k8s-labs.git`).
- **`source.path`**: Points to `"Lab 11: End-to-End Microservices Deployment /k8s"`.
- **`destination.namespace`**: Targets the `smallsteps` namespace.
- **`syncPolicy.automated.prune: true`**: Automatically deletes cluster resources that are removed from the Git repository.
- **`syncPolicy.automated.selfHeal: true`**: Detects manual configuration drift (e.g., changes made manually via `kubectl`) and immediately reverts the live cluster state to match Git.
- **`syncOptions: CreateNamespace=true`**: Automatically creates the `smallsteps` namespace if it does not already exist.
- **`ignoreDifferences` & `RespectIgnoreDifferences=true`**: Excludes `/spec/replicas` on `backend` Deployment. Because `backend-hpa` dynamically manages replica count based on CPU load, this prevents ArgoCD self-healing from resetting scaled replicas back to the base manifest count of 2.

### 9.4 Applying the ArgoCD Application

Deploy the application into ArgoCD:

```bash
kubectl apply -f argocd/application.yaml
```

Monitor the application synchronization status:

```bash
kubectl get application smallsteps -n argocd
```

Expected output:

```text
NAME         SYNC STATUS   HEALTH STATUS
smallsteps   Synced        Healthy
```
<img width="1256" height="193" alt="image" src="https://github.com/user-attachments/assets/d9476352-e157-496a-8c43-1393b647d032" />
<img width="1419" height="191" alt="image" src="https://github.com/user-attachments/assets/c632f28f-94bc-431c-8c10-aceea0a01d4c" />
<img width="1284" height="766" alt="image" src="https://github.com/user-attachments/assets/48da4ed9-ec91-412f-88b9-01735fe70f04" />
<img width="1834" height="788" alt="image" src="https://github.com/user-attachments/assets/0497f11b-6741-4980-b298-33131ac95307" />
<img width="1840" height="770" alt="image" src="https://github.com/user-attachments/assets/2a522512-9ae0-47a9-a485-86de954a8317" />

### 9.5 Auto-Sync, Self-Healing & Drift Detection in Action

1. **Self-Healing Verification**:
   If an administrator or unauthorized user manually alters or deletes a resource:

   ```bash
   # Manually scale frontend deployment to 0 replicas
   kubectl scale deployment frontend -n smallsteps --replicas=0
   ```

   ArgoCD detects the drift between live state (0 replicas) and Git desired state (1 replica) and automatically restores it to the desired state.

2. **Automated Resource Pruning**:
   If a manifest is deleted or renamed in the Git repository, ArgoCD's `prune: true` policy automatically deletes the orphaned resource from the cluster without manual intervention.

3. **PostSync Hook Execution**:
   Once all core deployments, statefulsets, and ingress resources reach a healthy state, ArgoCD automatically triggers `06-smoke-test-job.yaml` via `argocd.argoproj.io/hook: PostSync`. The `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation` guarantees that previous smoke test pods are cleaned up prior to running a fresh test.

---

## 10. Manual Deployment Alternative (Direct `kubectl`)

If deploying manually without ArgoCD, apply the manifests in sequence:

```bash
# 1. Create Namespace
kubectl apply -f k8s/00-namespace.yaml

# 2. Apply Configuration & Secrets
kubectl apply -f k8s/01-configmap-secret.yaml

# 3. Deploy Stateful Database
kubectl apply -f k8s/02-mongodb-statefulset.yaml

# 4. Deploy Backend Microservice
kubectl apply -f k8s/03-backend-deployment.yaml

# 5. Deploy Frontend Web Server
kubectl apply -f k8s/04-frontend-deployment.yaml

# 6. Configure Ingress Rules
kubectl apply -f k8s/05-ingress.yaml

# 7. Execute Post-Deployment Smoke Test
kubectl apply -f k8s/06-smoke-test-job.yaml

# 8. Schedule Database Backup CronJob
kubectl apply -f k8s/07-mongodb-backup-cronjob.yaml

# 9. Enable Horizontal Pod Autoscaling
kubectl apply -f k8s/08-backend-hpa.yaml
```

---

## 11. Local DNS Configuration

To access the application using the configured Ingress domain (`smallsteps.local`), resolve the domain to the K3s VM's IP address.

### 11.1 Identify the VM IP Address

On the K3s Linux host:

```bash
hostname -I
```

Example IP: `192.168.107.200`

### 11.2 Update the Hosts File

On the Windows host machine, edit the hosts file:

```text
C:\Windows\System32\drivers\etc\hosts
```

Open Notepad as **Administrator** and append:

```text
192.168.107.200 smallsteps.local
```

*(Replace `192.168.107.200` with your actual VM IP address).*

Verify domain resolution from Windows Command Prompt:

```cmd
ping smallsteps.local
```

---

## 12. Verification & Testing

### 12.1 Verify Cluster Resources

Inspect all workloads in the `smallsteps` namespace:

```bash
kubectl get all -n smallsteps
```

Check individual resource types:

```bash
# Check Pods
kubectl get pods -n smallsteps

# Check Services
kubectl get svc -n smallsteps

# Check StatefulSet and PVC
kubectl get statefulset -n smallsteps
kubectl get pvc -n smallsteps

# Check Ingress
kubectl get ingress -n smallsteps

# Check HPA
kubectl get hpa -n smallsteps

# Check Jobs and CronJobs
kubectl get jobs -n smallsteps
kubectl get cronjobs -n smallsteps
```

### 12.2 Test Application Endpoints

Open your browser or use curl to test the web frontend and API:

- **Frontend Web UI**: [http://smallsteps.local](http://smallsteps.local)
- **Backend Health Check**: [http://smallsteps.local/api/health](http://smallsteps.local/api/health)

Expected response from `/api/health`:

```json
{"status":"healthy","uptime":...}
```

### 12.3 Verify the Smoke Test Job

Check the execution logs of the automated smoke test job:

```bash
kubectl logs job/smallsteps-smoke-test -n smallsteps
```

Expected output:

```text
Testing backend...
{"status":"healthy","uptime":...}
Backend test passed.
Testing frontend...
OK
Frontend test passed.

All smoke tests passed.
```

---

## 13. StatefulSet Persistence & Resilience Testing

To demonstrate that MongoDB data persists across pod failures and recreations:

1. Identify the running MongoDB pod:

   ```bash
   kubectl get pod mongodb-0 -n smallsteps
   ```

2. Forcefully delete the pod:

   ```bash
   kubectl delete pod mongodb-0 -n smallsteps
   ```

3. Watch the StatefulSet self-heal and re-attach the volume:

   ```bash
   kubectl get pods -n smallsteps -w
   ```

4. Verify that `mongodb-0` is recreated and the PVC remains `Bound`:

   ```bash
   kubectl get pvc -n smallsteps
   ```

   The claim `mongodb-data-mongodb-0` retains all data, proving stateful persistence on K3s `local-path`.

---

## 14. Logs & Observability

Retrieve logs from individual microservices:

```bash
# Backend Node.js logs
kubectl logs deployment/backend -n smallsteps

# Frontend NGINX logs
kubectl logs deployment/frontend -n smallsteps

# MongoDB database logs
kubectl logs mongodb-0 -n smallsteps

# NGINX Ingress Controller routing logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

## 15. Troubleshooting Guide

### Pod in `ImagePullBackOff` or `ErrImagePull`

Cause: Container image is missing from the K3s containerd cache.

Resolution:

```bash
sudo k3s ctr -n k8s.io images list | grep smallsteps
```

If missing, re-import the tar archives:

```bash
sudo k3s ctr -n k8s.io images import /tmp/smallsteps-backend.tar
sudo k3s ctr -n k8s.io images import /tmp/smallsteps-frontend.tar
```

### Pod in `CrashLoopBackOff` or Failing Probes

Inspect events and describe the pod:

```bash
kubectl describe pod <pod-name> -n smallsteps
kubectl get events -n smallsteps --sort-by=.lastTimestamp
```

### Ingress Returns 404 or 503

1. Verify Ingress Controller pod status:

   ```bash
   kubectl get pods -n ingress-nginx
   ```

2. Verify Ingress resource backend endpoints:

   ```bash
   kubectl describe ingress smallsteps-ingress -n smallsteps
   ```

3. Ensure backend and frontend services have active endpoints:

   ```bash
   kubectl get endpoints -n smallsteps
   ```

---

## 16. Cleanup & Restoration

### 16.1 Remove Application via ArgoCD

If deployed via ArgoCD, delete the application (finalizer removes all cluster resources cleanly):

```bash
kubectl delete application smallsteps -n argocd
```

### 16.2 Manual Cleanup

If deployed manually:

```bash
kubectl delete namespace smallsteps
```

To remove NGINX Ingress Controller:

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

### 16.3 Restore Default K3s Traefik (Optional)

To revert K3s back to Traefik:

```bash
sudo nano /etc/rancher/k3s/config.yaml
```

Remove or comment out:

```yaml
# disable:
#   - traefik
```

Restart K3s:

```bash
sudo systemctl restart k3s
kubectl get pods -n kube-system | grep traefik
```

---

## 17. Summary & Topics Covered

| Category | Kubernetes & GitOps Resource | Functional Purpose |
|---|---|---|
| Continuous Delivery | ArgoCD `Application` | Declarative GitOps deployment, auto-sync & self-healing |
| Workload Isolation | `Namespace` (`smallsteps`) | Logical isolation for application microservices |
| Configuration | `ConfigMap` & `Secret` | Externalized environment variables & database credentials |
| Stateless Workloads | `Deployment` (`backend`, `frontend`) | Replicated pods with zero-downtime rolling updates |
| Stateful Workloads | `StatefulSet` (`mongodb`) | Ordered, predictable pod lifecycle with dedicated volume |
| Storage | `PVC` (`mongodb-data`) | Persistent data retention via K3s `local-path` StorageClass |
| Service Discovery | `Service` (`mongodb-headless`) | Direct Pod DNS resolution for stateful MongoDB |
| Traffic Balancing | `Service` (ClusterIP) | Internal routing to backend (:3000) and frontend (:80) |
| Ingress Routing | `Ingress` (NGINX) | External HTTP routing on domain `smallsteps.local` |
| Health Checks | Liveness, Readiness, Startup Probes | Automated failure recovery and traffic gatekeeping |
| Autoscaling | `HorizontalPodAutoscaler` | Dynamic backend pod scaling between 1 and 2 pods |
| Testing Hook | `Job` (`PostSync`) | Automated HTTP smoke test executed after ArgoCD sync |
| Scheduled Task | `CronJob` (`mongodb-backup`) | Daily automated database archive backup at 02:00 AM |
| Container Runtime | `containerd` | Direct image import into K3s container cache |
