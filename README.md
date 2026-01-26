
# 🐳 Student Registration System — Full Deployment Guide  
*(Written for absolute IT beginners)*

---

## 0. What you are about to build
Imagine three friends working in a restaurant:

| Friend | Job | Analogy |
|--------|-----|---------|
| **MariaDB** | Storekeeper | Keeps all student records on shelves (tables) |
| **Spring Boot** | Waiter | Takes orders (HTTP requests) from customers, brings data from the storekeeper |
| **React** | Waiter | Hands the food (web pages) to the customer |

We will pack each friend inside a **Docker container** so they can live on any Linux computer without fighting.

---

## 1. Prepare your Linux computer
*(You need a place to run everything. An Ubuntu 22.04 VM, AWS EC2, or even an old laptop is fine.)*

### 1.1 Update the package list
```bash
sudo apt update        # “Hey Ubuntu, refresh the list of installable programs.”
```

1.2 Install Docker

```bash
sudo apt install docker.io -y   # “Install Docker please, and say yes to everything.”
sudo systemctl start docker     # “Start the Docker service right now.”
sudo systemctl enable docker    # “Start Docker every time the computer reboots.”
docker --version                # “Prove to me Docker is there.”
```

What is Docker?

A tool that puts an application + everything it needs into a box (container) so it runs the same everywhere.

---

2. Give the storekeeper (MariaDB) a place to live

2.1 Create a permanent storage folder

```bash
docker volume create student-db-vol
```

Think of this as buying a hard-disk that survives even if the container dies.

2.2 Start MariaDB container

```bash
docker run -d \
  --name mariadb-container \
  -e MARIADB_ROOT_PASSWORD=redhat \
  -e MARIADB_DATABASE=studentdb \
  -v student-db-vol:/var/lib/mysql \
  mariadb:latest
```

Flag	Meaning	
`-d`	Run in background (detached)	
`--name mariadb-container`	Give the box a name so we can talk to it later	
`-e MARIADB_ROOT_PASSWORD=redhat`	Set the master password for the database	
`-e MARIADB_DATABASE=studentdb`	Create a ready-made database called “studentdb”	
`-v student-db-vol:/var/lib/mysql`	Attach the permanent disk inside the container at the folder MariaDB uses	
`mariadb:latest`	Use the newest official MariaDB image	

---

3. Find the storekeeper’s address

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb-container
```

This prints an IP like `172.17.0.2`.

Write it down – the waiter (Spring Boot) needs this address to fetch data.

---

4. Bring the waiter (Spring Boot) to life

4.1 Download the project

```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD/backend
```

4.2 Tell the waiter where the storekeeper lives
Open the file `src/main/resources/application.properties` with any text editor (nano, vim, VS Code).

Replace the line:

```
spring.datasource.url=jdbc:mariadb://<DB_IP>:3306/studentdb
```

with your real IP, e.g.:

```
spring.datasource.url=jdbc:mariadb://172.17.0.2:3306/studentdb
```

4.3 Build the waiter’s container image
Create a file named `Dockerfile` (exact spelling, no extension) inside the `backend` folder with these lines:

```
FROM maven:3.8.3-openjdk-17   # Start from a picture that already has Java & Maven
WORKDIR /opt/app              # Create and enter a working folder
COPY . .                      # Copy every file from current laptop folder into the image
RUN mvn clean package -DskipTests   # Compile the code and make a JAR file
EXPOSE 8080                   # Tell Docker we plan to use port 8080
ENTRYPOINT ["java","-jar","target/student-registration-backend-0.0.1-SNAPSHOT.jar"]
```

Now build the image (this may take 3-5 minutes the first time):

```bash
docker build -t myname/backend:v1 .
```

`-t` gives the image a tag (name). The dot means “use the Dockerfile in this folder”.

4.4 Push the image to Docker Hub (optional but handy)

```bash
docker login
docker tag myname/backend:v1 mydockerhubuser/curd_app:backend-v1
docker push mydockerhubuser/curd_app:backend-v1
```

4.5 Run the container

```bash
docker run -d \
  --name backend-container \
  -p 8080:8080 \
  myname/backend:v1
```

Port part	Meaning	
`-p 8080:8080`	Map laptop’s port 8080 → container’s port 8080 so you can talk to the waiter	

Quick test:

Open a browser and go to `http://<your-laptop-ip>:8080/api/students`

You should see an empty JSON list `[]` – that means the waiter is alive and can reach the storekeeper.

---

5. Bring the chef who decorates the plate (React frontend)

5.1 Move to the frontend folder

```bash
cd ../frontend
```

5.2 Tell React where the waiter lives
Create or edit the file `.env` (note the leading dot) and write:

```
VITE_API_URL="http://<BACKEND_PUBLIC_IP>:8080/api"
```

Important:  
- If you are testing inside the same laptop, you can use `localhost`.  
- If the browser is on another computer, use the public IP of the laptop running the backend.

5.3 Build the frontend container image
Create `Dockerfile` inside `frontend` folder:

```
FROM node:22-alpine
WORKDIR /opt/app
COPY . .
RUN npm install && npm run build
RUN apk add --no-cache apache2
RUN cp -rf dist/* /var/www/localhost/htdocs/
EXPOSE 80
CMD ["httpd","-D","FOREGROUND"]
```

Build:

```bash
docker build -t myname/frontend:v1 .
```

Push (optional):

```bash
docker tag myname/frontend:v1 mydockerhubuser/curd_app:frontend-v1
docker push mydockerhubuser/curd_app:frontend-v1
```

Run:

```bash
docker run -d \
  --name frontend-container \
  -p 80:80 \
  myname/frontend:v1
```

5.4 Enjoy the food
Open any browser and type:

```
http://<your-laptop-ip>
```

You should see the Student Registration page. Add a student – the data travels:

Browser → React (port 80) → Spring Boot (port 8080) → MariaDB

---

6. Manual way (without Docker) – quick sketch
If Docker feels scary, you can run each part directly on your laptop:

Part	Commands	
MariaDB	`sudo apt install mariadb-server` then create database	
Backend	Install Java 17 + Maven, `mvn clean package`, `java -jar target/....jar`	
Frontend	Install Node.js, `npm install`, `npm run build`, copy `dist/` folder into Apache’s `/var/www/html`	

The files you must still edit are the same: `application.properties` and `.env`.

---

7. Common mistakes and how to fix them

Problem	Typical cause	Quick fix	
Browser shows “Cannot connect”	Wrong IP in `.env`	Use the public IP of the backend laptop	
Backend exits immediately	Wrong DB IP	`docker logs backend-container` will show “Connection refused” – fix IP	
Port 8080 already in use	Another program uses it	Change to `-p 9090:8080` and update `.env`	
“table doesn’t exist”	First run	Add `spring.jpa.hibernate.ddl-auto=update` so Hibernate creates tables	

---

8. Clean-up commands (when you want to start over)

```bash
docker stop mariadb-container backend-container frontend-container
docker rm mariadb-container backend-container frontend-container
docker volume prune   # Remove orphaned disks
```

---

9. Next steps for curious minds
- Put all three containers in one docker-compose.yml file so you can start everything with a single command.  
- Add HTTPS with Nginx and free SSL certificates.  
- Replace IP addresses with container names so Docker’s internal DNS does the wiring.

---
