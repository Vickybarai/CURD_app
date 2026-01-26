# -------------------------------
# 1. Update Ubuntu & Install Docker
# -------------------------------
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version

# -------------------------------
# 2. Create MariaDB Volume & Run Container
# -------------------------------
docker volume create student-db-vol

docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest

# -------------------------------
# 3. Get MariaDB Container IP
# -------------------------------
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container

# -------------------------------
# 4. Backend Setup
# -------------------------------
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend

# Backend Dockerfile
cat > Dockerfile <<EOF
FROM maven:3.8.3-openjdk-17
WORKDIR /opt/app
COPY . .
RUN mvn clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]
EOF

# Build & Run Backend
docker build -t myname/backend:v1 .
docker run -d \
  --name backend-container \
  -p 8080:8080 \
  myname/backend:v1

# Optional: Push Backend to Docker Hub
docker login
docker tag myname/backend:v1 mydockerhubuser/curd_app:backend-v1
docker push mydockerhubuser/curd_app:backend-v1

# -------------------------------
# 5. Frontend Setup
# -------------------------------
cd ../frontend

# Frontend Dockerfile
cat > Dockerfile <<EOF
FROM node:22-alpine
WORKDIR /opt/app
COPY . .
RUN npm install && npm run build
RUN apk add --no-cache apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
CMD ["httpd","-D","FOREGROUND"]
EOF

# Build & Run Frontend
docker build -t myname/frontend:v1 .
docker run -d \
  --name frontend-container \
  -p 80:80 \
  myname/frontend:v1

# Optional: Push Frontend to Docker Hub
docker tag myname/frontend:v1 mydockerhubuser/curd_app:frontend-v1
docker push mydockerhubuser/curd_app:frontend-v1

# -------------------------------
# 6. Manual Deployment (Without Docker)
# -------------------------------

# Backend
sudo apt install openjdk-17-jdk -y
sudo apt install maven -y
mvn clean package
java -jar target/spring-backend-v1.jar

# Frontend
sudo apt install nodejs npm -y
npm install
npm run build

# Deploy Frontend to Apache
sudo apt install apache2 -y
sudo cp -rf dist/* /var/www/html/
sudo systemctl restart apache2

# -------------------------------
# 7. Clean-up Commands (Start Over)
# -------------------------------
docker stop mariadb-container backend-container frontend-container
docker rm mariadb-container backend-container frontend-container
docker volume prune -f
docker network prune -f