# 📘 Production-Grade K8s Deployment
**Order:** Clone → Service → Deployment → Docker → Ingress → HPA  
### 1. Change Docker Hub Username
**Why:** The Kubernetes manifests currently point to `baraivicky/k8s`. You must change this to your own Docker Hub ID so K8s can pull *your* images.

*   **File 1:** `k8s/backend/deployment.yaml`
     **Change To:** `image: <YOUR-DOCKERHUB-ID>/k8s:backend-v1`

*   **File 2:** `k8s/frontend/deployment.yaml`
     **Change To:** `image: <YOUR-DOCKERHUB-ID>/k8s:frontend-v1`

*   **Also Change in Phase 5 (Build Commands):**
    *   Change `baraivicky/k8s:backend-v1` to `<YOUR-DOCKERHUB-ID>/k8s:backend-v1`
    *   Change `baraivicky/k8s:frontend-v1` to `<YOUR-DOCKERHUB-ID>/k8s:frontend-v1`

### 2. Change Ingress Host (Load Balancer URL)
**Why:** The Ingress file has a hardcoded AWS Load Balancer address from Sir's demo. You must replace this with your cluster's Load Balancer address (or a real domain name).

*   **File:** `k8s/ingress.yaml`
    *   **Find:** `host: af049ffa178f7428aa19c4548441ee9f-1118725864.ap-southeast-1.elb.amazonaws.com`
    *   **Change To:** `host: <YOUR-AWS-ELB-URL>` or `host: <YOUR-DOMAIN.COM>`

### 3. Change AWS Cluster Details
**Why:** To connect your terminal to the correct cluster.

*   **Phase 8 Commands:**
    *   Change `<cluster-name>` to your actual EKS cluster name.
    *   Change `<region>` to your region (e.g., `ap-southeast-1`).

---

## 🏗️ PHASE 1: CLONE & SETUP
*Objective:* Initialize the local repository and configure remotes.

**Where:** Terminal  
**Command:**
```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD
git checkout cdec-b48
git remote rename origin upstream
git remote add origin https://github.com/Vickybarai/CURD_app.git
git checkout -b k8s-curd
mkdir -p k8s/backend k8s/frontend
```

---

## 🌐 PHASE 2: SERVICE YAMLs (Define Contracts)
*Objective:* Create the ClusterIP services first.

### 2.1 Database Service
**File:** `k8s/studentapp-db-svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentapp-db-svc
  labels:
    app: studentapp-db
spec:
  selector:
    app: studentapp-db
  ports:
    - port: 3306
      targetPort: 3306
```

### 2.2 Backend Service
**File:** `k8s/backend/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentapp-svc
  labels:
    app: studentapp
spec:
  selector:
    app: studentapp
  ports:
  - port: 8080
    targetPort: 8080
```

### 2.3 Frontend Service
**File:** `k8s/frontend/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentapp-frontend-svc
spec:
  selector:
    app: studentapp-frontend
  ports:
  - port: 80
    targetPort: 80
```

---

## 💻 PHASE 3: DEPLOYMENT YAMLs (Define Workloads)
*Objective:* Define the Pods and StatefulSets. **Note the <YOUR-DOCKERHUB-ID> placeholders here.**

### 3.1 Database StatefulSet
**File:** `k8s/studentapp-db-sts.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: studentapp-db
  labels:
    app: studentapp-db
spec:
  selector:
    matchLabels:
      app: studentapp-db
  replicas: 1
  template:
    metadata:
      labels:
        app: studentapp-db
    spec:
      containers:
      - name: studentapp
        image: mariadb:latest
        ports:
        - containerPort: 3306
        env:
          - name: "MARIADB_ROOT_PASSWORD"
            value: "redhat"
          - name: "MARIADB_DATABASE"
            value: "studentapp-db"
```

### 3.2 Backend Deployment
**File:** `k8s/backend/deployment.yaml`
*Note: **CHANGE `baraivicky` TO YOUR USERNAME** below.*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentapp
  labels:
    app: studentapp
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: studentapp
  template:
    metadata:
      labels:
        app: studentapp
    spec:
      containers:
      - name: studentapp
        image: baraivicky/k8s:backend-v1  # <--- CHANGE THIS
        resources:
          limits:
            memory: "256Mi"
            cpu: "500m"
          requests:
            memory: "128Mi"
            cpu: "400m"
        ports:
        - containerPort: 8080
```

### 3.3 Frontend Deployment
**File:** `k8s/frontend/deployment.yaml`
*Note: **CHANGE `baraivicky` TO YOUR USERNAME** below.*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentapp-frontend
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: studentapp-frontend
  template:
    metadata:
      labels:
        app: studentapp-frontend
    spec:
      containers:
      - name: studentapp-frontend
        image: baraivicky/k8s:frontend-v1  # <--- CHANGE THIS
        resources:
          limits:
            memory: "256Mi"
            cpu: "500m"
          requests:
            memory: "128Mi"
            cpu: "400m"
        ports:
        - containerPort: 80
```

---

## 🐳 PHASE 4: DOCKERFILES (Define Containers)
*Objective:* Create the build recipes for the images referenced in Phase 3.

### 4.1 Backend Dockerfile
**File:** `backend/Dockerfile`
```dockerfile
FROM maven:3.8.3-openjdk-17 AS build
WORKDIR /opt
COPY . .
RUN mvn clean package -DskipTests

FROM openjdk:17.0.2-jdk
COPY --from=build /opt/target/student-registration-backend-0.0.1-SNAPSHOT.jar /opt/studentapp.jar
EXPOSE 8080
CMD ["java", "-jar", "/opt/studentapp.jar"]
```

### 4.2 Frontend Dockerfile
**File:** `frontend/Dockerfile`
```dockerfile
FROM node:22-alpine AS build
WORKDIR /opt
COPY . .
RUN npm install && npm run build

FROM httpd:latest
COPY --from=build /opt/dist/ htdocs/
EXPOSE 80
CMD ["httpd", "-D", "FOREGROUND"]
```

---

## 🛠️ PHASE 5: BUILD & PUSH IMAGES
*Objective:* Use the Dockerfiles to create the images. **CHANGE `baraivicky` TO YOUR USERNAME** in the build tags.

**Where:** Terminal  
**Command:**
```bash
# Build Backend
cd backend
docker build -t baraivicky/k8s:backend-v1 .  # <--- CHANGE THIS
docker push baraivicky/k8s:backend-v1         # <--- CHANGE THIS

# Build Frontend
cd ../frontend
docker build -t baraivicky/k8s:frontend-v1 . # <--- CHANGE THIS
docker push baraivicky/k8s:frontend-v1        # <--- CHANGE THIS

cd ..
```

---

## 🚪 PHASE 6: INGRESS (Define Entry Point)
*Objective:* Define the external access rules. **CHANGE THE HOST** to your AWS ELB URL.

**File:** `k8s/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: studentapp-ingress
  labels:
    app: studentapp
spec:
  ingressClassName: nginx
  rules:
  - host: af049ffa178f7428aa19c4548441ee9f-1118725864.ap-southeast-1.elb.amazonaws.com # <--- CHANGE THIS TO YOUR URL
    http:
      paths:
      - pathType: Prefix
        path: "/api"
        backend:
          service:
            name: studentapp-svc
            port: 
              number: 8080
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: studentapp-frontend-svc
            port: 
              number: 80
```

---

## 📈 PHASE 7: HPA (Define Scaling)
*Objective:* Define scaling policies.

### 7.1 Backend HPA
**File:** `k8s/backend/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: studentapp-hpa
  labels:
    app: studentapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: studentapp
  minReplicas: 1
  maxReplicas: 2
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 7.2 Frontend HPA
**File:** `k8s/frontend/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: studentapp-frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: studentapp-frontend
  minReplicas: 1
  maxReplicas: 2
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 🚀 PHASE 8: EXECUTION & DEPLOYMENT
*Objective:* Push code to Git and Apply to Kubernetes.

### Step 1: Commit & Push to Git
**Where:** Terminal
```bash
git add .
git commit -m "Production grade K8s setup with custom configs"
git push origin k8s-curd
```

### Step 2: Connect to EKS
**Where:** Terminal
**What Replace:** `<cluster-name>` and `<region>`
```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl get nodes
```

### Step 3: Apply in Dependency Order
**Where:** Terminal
```bash
# 1. Services (Network Layer)
kubectl apply -f k8s/studentapp-db-svc.yaml
kubectl apply -f k8s/backend/service.yaml
kubectl apply -f k8s/frontend/service.yaml

# 2. Database (Data Layer)
kubectl apply -f k8s/studentapp-db-sts.yaml

# 3. Verify DB Ready
kubectl get pods -w
# Wait for DB to be Running, then Ctrl+C

# 4. Deployments (Application Layer)
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/frontend/deployment.yaml

# 5. Ingress (Access Layer)
kubectl apply -f k8s/ingress.yaml

# 6. HPA (Scaling Layer)
kubectl apply -f k8s/backend/hpa.yaml
kubectl apply -f k8s/frontend/hpa.yaml
```

### Step 4: Verify Access
**Command:**
```bash
kubectl get ingress studentapp-ingress
```
Open the **ADDRESS** URL in your browser.