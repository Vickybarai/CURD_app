
## 🏁 STAGE 1: Clone & Setup

**Where:** Terminal
**Command:**
```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD
git checkout cdec-b48
```
In end push to your own repo:
```bash
git remote add origin https://github.com/Vickybarai/CURD_app.git
git checkout -b k8s-curd

```

---

## ⚙️ STAGE 2: Local Code Configuration
**Why?** Kubernetes uses internal DNS for services. We must update your code so it knows how to find the database and the API.

### Step 1: Configure Backend (Database Connection)
**Where:** `backend/src/main/resources/application.properties`

**Action:** Open this file and edit the `spring.datasource.url`.

**Change this line:**
```properties
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
```

**To this (Replace `<DB_IP>` with the K8s Service Name):**
```properties
spring.datasource.url=jdbc:mariadb://studentapp-db-svc:3306/studentdb
```

> **Beginner Note:** In Kubernetes, we do not use IP addresses like `192.168.1.5`. We use the **Service Name** (`studentapp-db-svc`). Kubernetes handles the DNS resolution automatically.

**Final `application.properties` should look like this:**
```properties
server.port=8080
spring.datasource.url=jdbc:mariadb://studentapp-db-svc:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat
spring.jpa.hibernate.ddl-auto=update
```

---

### Step 2: Configure Frontend (API Connection)
**Where:** `frontend/.env` (Create this file if it doesn't exist).

**Action:** We need to tell the Frontend where the Backend is. Since the Frontend runs in the user's browser, it needs a public URL.

**Edit this line:**
```env
VITE_API_URL="http://<BACKEND_PUBLIC_IP>:8080/api"
```

**To this (Replace `<BACKEND_PUBLIC_IP>` with your Ingress URL):**
```env
VITE_API_URL="http://<YOUR-INGRESS-HOST-URL>/api"
```

> **Beginner Note:** We will get this `<YOUR-INGRESS-HOST-URL>` from AWS in **Stage 5**. If you don't have it yet, keep this file open and come back to it after the Ingress is created.

---

## 🐳 STAGE 3: Create Dockerfiles

### Step 1: Backend Dockerfile
**Where:** `backend/Dockerfile`

**Content:**
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

### Step 2: Frontend Dockerfile
**Where:** `frontend/Dockerfile`

**Content:**
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

## 📦 STAGE 4: Create Kubernetes YAMLs

### Step 1: Database Service
**Where:** `k8s/studentapp-db-svc.yaml`

**Content:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentapp-db-svc
spec:
  selector:
    app: studentapp-db
  ports:
    - port: 3306
      targetPort: 3306
```

### Step 2: Database StatefulSet
**Where:** `k8s/studentapp-db-sts.yaml`

**Content:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: studentapp-db
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
            value: "studentdb"
```

### Step 3: Backend Service
**Where:** `k8s/backend/service.yaml`

**Content:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentapp-svc
spec:
  selector:
    app: studentapp
  ports:
  - port: 8080
    targetPort: 8080
```

### Step 4: Backend Deployment
**Where:** `k8s/backend/deployment.yaml`
**Note:** Change `image: baraivicky/k8s:backend-v1` to your Docker ID.

**Content:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentapp
spec:
  replicas: 1
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
        ports:
        - containerPort: 8080
```

### Step 5: Frontend Service
**Where:** `k8s/frontend/service.yaml`

**Content:**
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

### Step 6: Frontend Deployment
**Where:** `k8s/frontend/deployment.yaml`
**Note:** Change `image: baraivicky/k8s:frontend-v1` to your Docker ID.

**Content:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentapp-frontend
spec:
  replicas: 1
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
        ports:
        - containerPort: 80
```

---

## 🚪 STAGE 5: Ingress Configuration (Entry Point)

**Objective:** Define the external access rules. **CHANGE THE HOST** to your AWS ELB URL.

### How to find your ELB URL?
If you already have an Nginx Ingress Controller running, get the URL by running:
`kubectl get svc -n ingress-nginx`
*(Copy the EXTERNAL-IP address, e.g., `a1b2c...elb.amazonaws.com`)*

### Create Ingress File
**Where:** `k8s/ingress.yaml`

**Action:** Replace `<YOUR-AWS-ELB-URL>` below with the address you found above.

**Content:**
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
  - host: <YOUR-AWS-ELB-URL> 
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

> **Important:** Once you create this file, go back to **Stage 2, Step 2** and update your `frontend/.env` file with this exact `<YOUR-AWS-ELB-URL>`.

---

## 🚀 STAGE 6: Build, Push & Deploy

### 1. Build and Push Images
**Where:** Terminal
**Command:**
```bash
# Backend
cd backend
docker build -t baraivicky/k8s:backend-v1 .
docker push baraivicky/k8s:backend-v1

# Frontend
cd ../frontend
docker build -t baraivicky/k8s:frontend-v1 .
docker push baraivicky/k8s:frontend-v1
```

### 2. Deploy to Kubernetes
**Where:** Terminal
**Command:**
```bash
# 1. Database
kubectl apply -f k8s/studentapp-db-svc.yaml
kubectl apply -f k8s/studentapp-db-sts.yaml

# 2. Backend & Frontend Services
kubectl apply -f k8s/backend/service.yaml
kubectl apply -f k8s/frontend/service.yaml

# 3. Deploy Apps
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/frontend/deployment.yaml

# 4. Ingress
kubectl apply -f k8s/ingress.yaml
```

### 3. Access Application
**Where:** Browser
**Action:** Open `http://<YOUR-AWS-ELB-URL>`


[Creating by dockerfile creating](https://github.com/Vickybarai/Devops/blob/main/Docker%2FDocker_4_Dockerfile_%26_Easy_CURD.MD)
[Create with yaml file](https://github.com/Vickybarai/project/blob/main/3-Tier_EasyCRUD_%28using_yaml%29.md)