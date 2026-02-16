
---

3-Tier K8s Deployment 
---

✅ STEP 1 — PREPARE & BUILD (Application Layer)

1.1 Clone Repository

Where: Terminal
Command:

git clone https://github.com/shubhamkalsait/EasyCRUD.git  
cd EasyCRUD  
git checkout cdec-b48

git checkout -b k8s-curd


---

1.2 Create Database Service YAML (Define Service Name)

Objective: Create the DB service file now so you know the Service Name to use in the Backend configuration.

File: k8s/studentapp-db-svc.yaml
Content:

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

> Key Value: The name is studentapp-db-svc. You must use this in the next step.




---

1.3 Update Backend Configuration

Objective: Connect Backend to the Database using the Service Name

File: backend/src/main/resources/application.properties

Edit the URL line:

spring.datasource.url=jdbc:mariadb://studentapp-db-svc:3306/studentdb  
spring.datasource.username=root  
spring.datasource.password=redhat  
spring.jpa.hibernate.ddl-auto=update

> Dependency Check: studentapp-db-svc must match the name in 1.2.




---

1.4 Update Frontend Configuration

Objective: Tell Frontend how to call the API. We use relative paths for K8s Ingress.

File: frontend/.env (Create this file)

Content:

VITE_API_URL="/api"

> Why: Using /api ensures it works regardless of the IP (Localhost vs AWS).




---

1.5 Create Dockerfiles

Backend Dockerfile (backend/Dockerfile):

FROM maven:3.8.3-openjdk-17 AS build  
WORKDIR /opt  
COPY . .  
RUN mvn clean package -DskipTests  
  
FROM openjdk:17.0.2-jdk  
COPY --from=build /opt/target/student-registration-backend-0.0.1-SNAPSHOT.jar /opt/studentapp.jar  
EXPOSE 8080  
CMD ["java", "-jar", "/opt/studentapp.jar"]

Frontend Dockerfile (frontend/Dockerfile):

FROM node:22-alpine AS build  
WORKDIR /opt  
COPY . .  
RUN npm install && npm run build  
  
FROM httpd:latest  
COPY --from=build /opt/dist/ htdocs/  
EXPOSE 80  
CMD ["httpd", "-D", "FOREGROUND"]


---

1.6 Build and Push Docker Images

Objective: Build the images with the correct configuration and push to Docker Hub.

Where: Terminal
Action: Replace baraivicky with your Docker ID.

Command:

# Backend  
cd backend  
docker build -t baraivicky/k8s:backend-v1 .  
docker push baraivicky/k8s:backend-v1  
  
# Frontend  
cd ../frontend  
docker build -t baraivicky/k8s:frontend-v1 .  
docker push baraivicky/k8s:frontend-v1  
  
cd ..


---

✅ STEP 2 — DEPLOY TO KUBERNETES (Infrastructure Layer)

Now deploy everything to the cluster in the correct dependency order.

2.1 Deploy Database (Data Layer)

Action: Apply Service and StatefulSet. Wait for the Pod to be Running.

Command:

kubectl apply -f k8s/studentapp-db-svc.yaml  
kubectl apply -f k8s/studentapp-db-sts.yaml

Verify:

kubectl get pods -w

(Press Ctrl+C once the DB pod is Running)

File Reference for studentapp-db-sts.yaml:

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


---

2.2 Deploy Backend (API Layer)

Action: Create Service and Deployment.

Command:

kubectl apply -f k8s/backend/service.yaml  
kubectl apply -f k8s/backend/deployment.yaml

File Reference for backend/service.yaml:

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

File Reference for backend/deployment.yaml:

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


---

2.3 Deploy Frontend (UI Layer)

Action: Create Service and Deployment.

Command:

kubectl apply -f k8s/frontend/service.yaml  
kubectl apply -f k8s/frontend/deployment.yaml

File Reference for frontend/service.yaml:

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

File Reference for frontend/deployment.yaml:

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


---

2.4 Deploy Ingress (Entry Point)

Action: Expose the application to the outside world.

Command:

kubectl apply -f k8s/ingress.yaml

File Reference for ingress.yaml:

apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: studentapp-ingress  
  labels:  
    app: studentapp  
spec:  
  ingressClassName: nginx  
  rules:  
  - http:  
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


---

🔎 Final Logical View

1. STEP 1 = Configure + Build: We ensured the code knows how to talk to K8s (studentapp-db-svc) and we baked that logic into the images.


2. STEP 2 = Deploy in Order: We launched the DB first (so it's ready), then Backend, then Frontend, then Ingress to connect them all.



Access the App:

kubectl get ingress studentapp-ingress

Copy the ADDRESS  define 1.4 at Update Frontend Configuration

and open it in your browser.

[Creating by dockerfile:](https://github.com/Vickybarai/Devops/blob/main/Docker%2FDocker_4_Dockerfile_%26_Easy_CURD.MD)
[Create with yaml file :](https://github.com/Vickybarai/project/blob/main/3-Tier_EasyCRUD_%28using_yaml%29.md)