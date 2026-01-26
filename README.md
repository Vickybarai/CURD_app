

# 🐳 Student Registration System — Full Deployment Guide

Frontend (React) → Backend (Spring Boot) → Database (MariaDB)

---

# 🐳 METHOD 1 — DOCKER DEPLOYMENT

## 🚀 Install Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version


---

🧱 Run Database (MariaDB)

docker volume create student-db-vol

docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest


---

🔍 Get Database IP

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container


---

⚙️ Backend Setup

git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend

Edit file:

backend/src/main/resources/application.properties

Update:

spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
spring.datasource.username=root
spring.datasource.password=redhat


---

🐳 Backend Dockerfile

FROM maven:3.8.3-openjdk-17
WORKDIR /opt/app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]


---

Build & Push Backend

docker build -t yourdockerhub/curd_app:backend-v1 .
docker push yourdockerhub/curd_app:backend-v1


---

Run Backend

docker run -d \
  --name backend-container \
  -p 8080:8080 \
  yourdockerhub/curd_app:backend-v1


---

🎨 Frontend Setup

cd ../frontend
vim .env

Add:

VITE_API_URL="http://<SERVER_PUBLIC_IP>:8080/api"


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

Build & Run Frontend

docker build -t yourdockerhub/curd_app:frontend-v1 .
docker push yourdockerhub/curd_app:frontend-v1

docker run -d \
  --name frontend-container \
  -p 80:80 \
  yourdockerhub/curd_app:frontend-v1


---

🌍 Access App

http://YOUR_SERVER_PUBLIC_IP


---

🖥 METHOD 2 — MANUAL DEPLOYMENT

Backend

sudo apt install openjdk-17-jdk maven -y
cd backend
mvn clean package
java -jar target/spring-backend-v1.jar


---

Frontend

sudo apt install nodejs npm apache2 -y
cd frontend
npm install
npm run build
sudo cp -rf dist/* /var/www/html/
sudo systemctl restart apache2


---

🧹 Docker Cleanup

docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rm -f $(docker ps -aq)
docker volume prune
docker network prune

