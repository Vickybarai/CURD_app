
🐳 Student Registration System — Full Deployment Guide

A 3-Tier Web Application

Frontend (React) → Backend (Spring Boot) → Database (MariaDB)


---

📦 Project Structure

EasyCRUD/
 ├── backend/     → Spring Boot API
 ├── frontend/    → React UI
 └── README.md


---

✅ Prerequisites

Tool	Purpose

Ubuntu / Linux Server	Hosting
Docker	Container deployment
Git	Clone repository
Java 17	Backend manual run
Maven	Build backend
Node.js + npm	Build frontend



---

🐳 METHOD 1 — DOCKER DEPLOYMENT (RECOMMENDED)


---

🚀 Step 1 — Install Docker

sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version


---

🧱 Step 2 — Run Database (MariaDB)

Create persistent volume:

docker volume create student-db-vol

Start database container:

docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest


---

🔍 Step 3 — Get Database IP

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container

Save this IP — it will be used in backend configuration.


---

⚙️ Step 4 — Backend Setup

Clone repository:

git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend

Edit configuration file:

backend/src/main/resources/application.properties

Update values:

spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat


---

🐳 Backend Dockerfile

Create file: backend/Dockerfile

FROM maven:3.8.3-openjdk-17
WORKDIR /opt/app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]

Build and push image:

docker build -t yourdockerhub/curd_app:backend-v1 .
docker push yourdockerhub/curd_app:backend-v1

Run backend container:

docker run -d \
  --name backend-container \
  -p 8080:8080 \
  yourdockerhub/curd_app:backend-v1


---

🎨 Step 5 — Frontend Setup

Move to frontend directory:

cd ../frontend

Edit environment file:

vim .env

Add:

VITE_API_URL="http://<SERVER_PUBLIC_IP>:8080/api"


---

🐳 Frontend Dockerfile

Create file: frontend/Dockerfile

FROM node:22-alpine
WORKDIR /opt/app
COPY . .
RUN npm install && npm run build
RUN apk add --no-cache apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
CMD ["httpd","-D","FOREGROUND"]

Build and run:

docker build -t yourdockerhub/curd_app:frontend-v1 .
docker push yourdockerhub/curd_app:frontend-v1

docker run -d \
  --name frontend-container \
  -p 80:80 \
  yourdockerhub/curd_app:frontend-v1


---

🌍 Access Application

http://YOUR_SERVER_PUBLIC_IP


---

🖥 METHOD 2 — MANUAL DEPLOYMENT


---

⚙️ Backend Manual Setup

sudo apt install openjdk-17-jdk maven -y
cd backend
vim src/main/resources/application.properties

Build and run:

mvn clean package
java -jar target/spring-backend-v1.jar


---

🎨 Frontend Manual Setup

sudo apt install nodejs npm apache2 -y
cd frontend
npm install
vim .env

Add:

VITE_API_URL="http://<BACKEND_IP>:8080/api"

Build and deploy:

npm run build
sudo cp -rf dist/* /var/www/html/
sudo systemctl restart apache2


---

🧹 Docker Cleanup Commands

docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rm -f $(docker ps -aq)
docker volume prune
docker network prune


---

🧠 Files You MUST Modify

File	What to Change

backend/application.properties	Database IP
frontend/.env	Backend public IP
Docker image name	Your DockerHub username



---

🏁 Architecture

Browser → Frontend (80) → Backend (8080) → MariaDB


