```markdown
# 🐳 Student Registration System — Full Deployment Guide

A 3-Tier Web Application  
Frontend (React) → Backend (Spring Boot) → Database (MariaDB)

---

## 📦 Project Structure

```

EasyCRUD/
├── backend/   # Spring Boot API
├── frontend/  # React UI
└── README.md

```

---

## ✅ Prerequisites

| Tool                  | Purpose               |
|-----------------------|-----------------------|
| Ubuntu / Linux Server | Hosting               |
| Docker                | Container deployment  |
| Git                   | Clone repository      |
| Java 17               | Backend manual run    |
| Maven                 | Build backend         |
| Node.js + npm         | Build frontend        |

---

## 🐳 METHOD 1 — DOCKER DEPLOYMENT (RECOMMENDED)

### 🚀 Step 1 — Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version
```

🧱 Step 2 — Run Database (MariaDB)
Create volume

```bash
docker volume create student-db-vol
```

Start container

```bash
docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest
```

🔍 Step 3 — Get Database IP

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container
```

Save this IP — needed in backend config.

⚙️ Step 4 — Backend Setup

```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend
```

Edit file:

`backend/src/main/resources/application.properties`

Change DB IP inside:

```
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
```

Build & Push Image

```bash
docker build -t yourdockerhub/curd_app:backend-v1 .
docker push yourdockerhub/curd_app:backend-v1
```

Run Backend

```bash
docker run -d \
  --name backend-container \
  -p 8080:8080 \
  yourdockerhub/curd_app:backend-v1
```

🎨 Step 5 — Frontend Setup

```bash
cd ../frontend
vim .env
```

Edit API URL:

```
VITE_API_URL="http://<SERVER_PUBLIC_IP>:8080/api"
```

Build & Run

```bash
docker build -t yourdockerhub/curd_app:frontend-v1 .
docker push yourdockerhub/curd_app:frontend-v1

docker run -d \
  --name frontend-container \
  -p 80:80 \
  yourdockerhub/curd_app:frontend-v1
```

🌍 Access Application

```
http://YOUR_SERVER_PUBLIC_IP
```

---

🖥 METHOD 2 — MANUAL DEPLOYMENT

⚙️ Backend Manual

```bash
sudo apt install openjdk-17-jdk maven -y
cd backend
vim src/main/resources/application.properties
mvn clean package
java -jar target/spring-backend-v1.jar
```

🎨 Frontend Manual

```bash
sudo apt install nodejs npm apache2 -y
cd frontend
npm install
vim .env
npm run build
sudo cp -rf dist/* /var/www/html/
sudo systemctl restart apache2
```

---

🧹 Docker Cleanup Commands

```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rm -f $(docker ps -aq)
docker volume prune
docker network prune
```

---

🏁 Architecture

```
Browser → Frontend (80) → Backend (8080) → MariaDB
```

```