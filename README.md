
# 🐳 Student Registration System 
---

## 1. What we will do
1. Start a database (MariaDB)  
2. Start a Java server (Spring Boot)  
3. Start a web page (React)  
4. Open the page in your browser – done!

All three pieces run inside **Docker** boxes so nothing breaks.



[Details steps with dockerfile implementation]
(https://github.com/Vickybarai/Devops/blob/main/Docker%2FDocker_4_Dockerfile_%26_Easy_CURD.MD)
---

## 2. One-time setup
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

---

3. Start the database box

```bash
docker volume create student-db-vol
docker run -d --name db \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest
```

Get its IP:

```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' db
```

Copy the IP (looks like 172.17.0.2).

---

4. Build & start the Java server box

```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend
```

Open `src/main/resources/application.properties` and change:

```
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
```

to the IP you copied (e.g. 172.17.0.2).

Build and run:

```bash
docker build -t backend .
docker run -d --name api -p 8080:8080 backend
```

Test:

`http://your-server-ip:8080/api/students` → should show `[]`

---

5. Build & start the web page box

```bash
cd ../frontend
```

Create file `.env`:

```
VITE_API_URL="http://your-server-ip:8080/api"
```

Build and run:

```bash
docker build -t frontend .
docker run -d --name web -p 80:80 frontend
```

---

6. Use it
Open browser:

`http://your-server-ip`

Add a student – finished!

---

7. Stop / remove everything

```bash
docker stop db api web
docker rm db api web
docker volume prune
```
[EasyCRUD Repository](https://github.com/shubhamkalsait/EasyCRUD.git)