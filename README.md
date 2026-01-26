# 🐳 Student Registration System — Full Deployment Guide

This project is a **3-tier web application**:

Frontend (React) → Backend (Spring Boot) → Database (MariaDB)

You can deploy it in two ways:
1) Docker Deployment (Recommended)
2) Manual Deployment (Without Docker)

------------------------------------------------------------
📦 PROJECT STRUCTURE
------------------------------------------------------------

EasyCRUD/
 ├── backend/      → Spring Boot API
 ├── frontend/     → React UI
 └── README.md

------------------------------------------------------------
✅ PREREQUISITES
------------------------------------------------------------

Linux Server (Ubuntu / EC2)
Docker
Git
Java 17
Maven
Node.js + npm

============================================================
🐳 METHOD 1 — DOCKER DEPLOYMENT
============================================================

STEP 1 — Install Docker
-----------------------
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version

STEP 2 — Start Database (MariaDB)
---------------------------------
docker volume create student-db-vol

docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest

STEP 3 — Get Database IP
------------------------
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container

👉 Save this IP. You must use it in backend config.

STEP 4 — Backend Setup
----------------------
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend

EDIT FILE:
backend/src/main/resources/application.properties

CHANGE TO:

server.port=8080
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat
spring.jpa.hibernate.ddl-auto=update

Replace <DB_IP> with database IP from Step 3.

Create Dockerfile in backend folder:

FROM maven:3.8.3-openjdk-17
WORKDIR /opt/app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]

Build Image:
docker build -t yourdockerhub/curd_app:backend-v1 .
docker push yourdockerhub/curd_app:backend-v1

Run Backend:
docker run -d \
  --name backend-container \
  -p 8080:8080 \
  yourdockerhub/curd_app:backend-v1

STEP 5 — Frontend Setup
-----------------------
cd ../frontend

EDIT FILE:
frontend/.env

CHANGE:
VITE_API_URL="http://<BACKEND_PUBLIC_IP>:8080/api"

Create Dockerfile:

FROM node:22-alpine
WORKDIR /opt/app
COPY . .
RUN npm install && npm run build
RUN apk add --no-cache apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
CMD ["httpd","-D","FOREGROUND"]

Build & Run:
docker build -t yourdockerhub/curd_app:frontend-v1 .
docker push yourdockerhub/curd_app:frontend-v1

docker run -d \
  --name frontend-container \
  -p 80:80 \
  yourdockerhub/curd_app:frontend-v1

ACCESS APPLICATION:
http://YOUR_SERVER_IP

============================================================
🖥 METHOD 2 — MANUAL DEPLOYMENT
============================================================

Backend:
--------
apt install openjdk-17-jdk -y
apt install maven -y

Edit:
backend/src/main/resources/application.properties

spring.datasource.url=jdbc:mariadb://<DB_HOST>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat

Build:
mvn clean package

Run:
java -jar target/spring-backend-v1.jar

Frontend:
---------
apt install nodejs npm -y
npm install

Edit frontend/.env:
VITE_API_URL="http://<BACKEND_IP>:8080/api"

Build:
npm run build

Deploy to Apache:
apt install apache2 -y
cp -rf dist/* /var/www/html/
systemctl restart apache2

============================================================
❌ COMMON ERRORS
============================================================

Backend can't connect DB → Wrong DB IP
React API error → Wrong .env URL
Port already used → Change port
Container stops → docker logs <name>

============================================================
🔧 FILES YOU MUST EDIT
============================================================

backend/application.properties → DB IP, user, password
frontend/.env → Backend public IP
Docker image names → Your Docker Hub username

============================================================
🏁 DEPLOYMENT COMPLETE
============================================================

Browser → Frontend (80) → Backend (8080) → MariaDB