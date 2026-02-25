

***

# 3-Tier K8s Deployment

## ✅ STEP 1 — PREPARE & BUILD (Application Layer)

### 1.1 Clone Repository
**Where:** Terminal
**Command:**
```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD
git checkout cdec-b48
```

**OR PULL your own repo :**
```bash
git remote add origin https://github.com/Vickybarai/CURD_app.git
git checkout k8s-curd
```
---

### 1.2 Create Database Service YAML
**Objective:** Define the DB Service Name.

**File:** `k8s/studentapp-db-svc.yaml`
**Content:**
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
> **Key Value:** The name is **`studentapp-db-svc`**. You must use this in the next step.

---

### 1.3 Update Backend Configuration
**Objective:** Connect Backend to the Database.
**IMPORTANT:** The database name must match   YAML exactly (`studentapp-db`).

**File:** `backend/src/main/resources/application.properties`

**Content:**
```properties
server.port=8080
# CRITICAL: Use 'studentapp-db-src' (hyphen included) to  YAML configuration
spring.datasource.url=jdbc:mariadb://studentapp-db-svc:3306/studentapp-db
spring.datasource.username=root
spring.datasource.password=redhat
spring.jpa.hibernate.ddl-auto=update
```

---

### 1.4 Update Frontend Configuration
**File:** `frontend/.env`

**Content:**
```env
VITE_API_URL="/api"
```

---

### 1.5 Create Dockerfiles

**Backend Dockerfile** (`backend/Dockerfile`):
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

**Frontend Dockerfile** (`frontend/Dockerfile`):
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

### 1.6 Build and Push Docker Images
**Where:** Terminal
**Command:**
```bash
# login to Docker Hub (if not already logged in)
docker login -u <your-dockerhub-username>
password: <your-dockerhub-password>
# Backend
cd backend
docker build -t baraivicky/k8s:backend-v1 .
docker push baraivicky/k8s:backend-v1

# Frontend
cd ../frontend
docker build -t baraivicky/k8s:frontend-v1 .
docker push baraivicky/k8s:frontend-v1

cd ..
```

---

## ✅ STEP 2 — DEPLOY TO KUBERNETES (Infrastructure Layer)

Deploying in exact dependency order, using   YAML structure with your images.

### 2.1 Deploy Database (Data Layer)
**Command:**
```bash
kubectl apply -f k8s/studentapp-db-svc.yaml
kubectl apply -f k8s/studentapp-db-sts.yaml
```

**File Reference for `studentapp-db-sts.yaml`:**
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

---

### 2.2 Deploy Backend (API Layer)
**Command:**
```bash
kubectl apply -f k8s/backend/service.yaml
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/backend/hpa.yaml
```

**File Reference for `backend/service.yaml`:**
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

**File Reference for `backend/deployment.yaml`:**
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
        image: baraivicky/k8s:backend-v1
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

**File Reference for `backend/hpa.yaml`:**
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

---

### 2.3 Deploy Frontend (UI Layer)
**Command:**
```bash
kubectl apply -f k8s/frontend/service.yaml
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/hpa.yaml
```

**File Reference for `frontend/service.yaml`:**
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

**File Reference for `frontend/deployment.yaml`:**
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
        image: baraivicky/k8s:frontend-v1
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

**File Reference for `frontend/hpa.yaml`:**
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

### 2.4 Deploy Ingress (Entry Point)
**Objective:** Expose the application.
 ```bash
 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
 
 kubectl get svc -n ingress-nginx

 ```

> **⚠️ IMPORTANT NOTE:** The `host` in this Ingress file is currently hardcoded . **You must change this** to your own Load Balancer DNS name .


> **Note on Configuration:** Since we used `VITE_API_URL="/api"` in Step 1.4, the frontend will automatically connect to the backend on whatever AWS URL you used in the Ingress Host.

**Command:**
```bash
kubectl apply -f k8s/ingress.yaml
```

**File Reference for `ingress.yaml`:**
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
  - host:  a13141857ba3548bd9fd67c82fa7f84d-357852280.ca-central-1.elb.amazonaws.com
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

### Access the App
**Command:**
```bash
kubectl get ingress studentapp-ingress
```
**Action:** Copy the **ADDRESS / ENDPOINT ** and open it in your browser.


---

## 🔎 Final Verification Checklist

1.  ✅ **Service Names:** All Services (`studentapp-db-svc`, `studentapp-svc`, `studentapp-frontend-svc`) match the `application.properties` and YAML selectors.
2.  ✅ **Database Name:** `studentapp-db` (with hyphen) is used in both YAML and `application.properties`.
3.  ✅ **Resources & HPA:** Included exactly as per code.
4.  ✅ **Docker Images:** Updated to `baraivicky/k8s:backend-v1` and `baraivicky/k8s:frontend-v1`.


---
**Reference Links:**
*   [Creating by dockerfile](https://github.com/Vickybarai/Devops/blob/main/Docker%2FDocker_4_Dockerfile_%26_Easy_CURD.MD)
*   [Create with yaml file](https://github.com/Vickybarai/project/blob/main/3-Tier_EasyCRUD_%28using_yaml%29.md)