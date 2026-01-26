Perfect. I’ll give you complete, beginner-proof documentation — including what to edit, where to edit, and why. This will be a GitHub-level README.md that even a fresher can follow without confusion.

You can paste this directly into your project README.


---

🐳 Student Registration System — Full Deployment Guide

 a 3-tier web application deploy by docker file
:

Frontend (React) → Backend (Spring Boot) → Database (MariaDB)

You can deploy it in two ways:

1️⃣ Docker Deployment (Recommended)
2️⃣ Manual Deployment (Without Docker)


---

📦 PROJECT STRUCTURE

EasyCRUD/
 ├── backend/      → Spring Boot API
 ├── frontend/     → React UI
 └── README.md


---

✅ PREREQUISITES

Tool	Purpose

Linux Server (Ubuntu / EC2)	Hosting
Docker	Container deployment
Git	Clone project
Java 17	Run backend manually
Maven	Build backend
Node.js + npm	Build frontend



---

🐳 METHOD 1 — DOCKER DEPLOYMENT (BEST METHOD)


---

🚀 STEP 1 — Install Docker

sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

Check:

docker --version


---

🧱 STEP 2 — Start Database (MariaDB)

Create volume (data safety)

docker volume create student-db-vol

Run DB container

docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest


---

🔍 STEP 3 — Get Database IP

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container

Example output:

172.17.0.2

📌 You will use this IP in backend configuration.


---

⚙️ STEP 4 — Backend Setup

git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend

✏️ EDIT THIS FILE:

backend/src/main/resources/application.properties

🔧 CHANGE THESE VALUES:

server.port=8080
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat
spring.jpa.hibernate.ddl-auto=update

Replace:

<DB_IP>  → IP from Step 3


---

🐳 Backend Dockerfile

Create backend/Dockerfile:

FROM maven:3.8.3-openjdk-17
WORKDIR /opt/app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]


---

Build Image

docker build -t yourdockerhub/curd_app:backend-v1 .
docker push yourdockerhub/curd_app:backend-v1


---

Run Backend

docker run -d \
  --name backend-container \
  -p 8080:8080 \
  yourdockerhub/curd_app:backend-v1


---

🎨 STEP 5 — Frontend Setup

cd ../frontend

✏️ EDIT FILE:

frontend/.env

🔧 CHANGE:

VITE_API_URL="http://<BACKEND_PUBLIC_IP>:8080/api"

Replace:

<BACKEND_PUBLIC_IP> → Your server public IP


---

🐳 Frontend Dockerfile

FROM node:22-alpine
WORKDIR /opt/app
COPY . .
RUN npm install && npm run build
RUN apk add --no-cache apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
CMD ["httpd","-D","FOREGROUND"]


---

Build & Run

docker build -t yourdockerhub/curd_app:frontend-v1 .
docker push yourdockerhub/curd_app:frontend-v1

docker run -d \
  --name frontend-container \
  -p 80:80 \
  yourdockerhub/curd_app:frontend-v1


---

🌍 ACCESS APP

http://YOUR_SERVER_IP


---

🖥 METHOD 2 — MANUAL DEPLOYMENT (NO DOCKER)


---

⚙️ Backend Manual

Install Java

apt install openjdk-17-jdk -y

Install Maven

apt install maven -y

Configure DB

Edit:

backend/src/main/resources/application.properties

spring.datasource.url=jdbc:mariadb://<DB_HOST>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat


---

Build Backend

mvn clean package

Run

java -jar target/spring-backend-v1.jar


---

🎨 Frontend Manual

Install Node

apt install nodejs npm -y

Install dependencies

npm install

Edit .env

VITE_API_URL="http://<BACKEND_IP>:8080/api"

Build

npm run build

Deploy to Apache

apt install apache2 -y
cp -rf dist/* /var/www/html/
systemctl restart apache2


---

❌ COMMON ERRORS

Issue	Solution

Backend can't connect DB	Wrong DB IP
React shows API error	Wrong .env URL
Port already used	Change port
CORS error	Enable CORS in backend
Container stops	docker logs <name>



---

🧠 IMPORTANT THINGS YOU MUST CHANGE

File	What to change

backend/application.properties	DB IP, user, password
frontend/.env	Backend public IP
Docker image names	Your Docker Hub username



---

🏁 YOU ARE DONE

Architecture now:

Browser → Frontend (80) → Backend (8080) → MariaDB


--
