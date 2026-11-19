# SET-6 — Solutions

Original question paper: `SET-6.docx` (top level). Lab Internal II, Software Engineering (23CC505PC).
Repository: `https://github.com/Kumbhambhargavi75/CEMS`

---

### Question 1. Docker Compose – WordPress and MySQL [25 MARKS]

**Question:** a) A company wants to create a blogging website using WordPress and MySQL. Write a Docker Compose file to run both containers. b) Users cannot access the WordPress website. Configure port mapping from host port 8080 to container port 80. c) WordPress starts before MySQL and causes database errors. Use `depends_on` to configure the service dependency. d) A company wants to protect its blog data from loss when the MySQL container stops. Configure a named volume for persistent storage. e) Create a complete Docker Compose file to run WordPress and MySQL with port mapping, service dependency and persistent storage.

**One-line explanation:** Build one `docker-compose.yml` with MySQL and WordPress services, `ports: "8080:80"`, `depends_on`, and named volumes.

**Solution:**
1. Create `docker-compose.yml` with two services: `db` (`mysql:8.0`) and `wordpress` (`wordpress:latest`).
2. Map host 8080 to container 80 with `ports: - "8080:80"` as the question asks. Note: Jenkins already uses 8080 on this laptop, so stop Jenkins or use `8000:80` locally to avoid a clash.
3. Add `depends_on: [db]` so Compose starts MySQL first.
4. Add named volumes `db_data:/var/lib/mysql` and `wp_data:/var/www/html` so data survives `down`/recreate.
5. Run `docker compose up -d`, `docker compose ps`, `docker compose logs -f`, open `http://localhost:8080` and finish WordPress setup.

```yaml
services:
  db:
    image: mysql:8.0
    container_name: cems-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db_data:/var/lib/mysql          # (d) persistent DB
  wordpress:
    image: wordpress:latest
    container_name: cems-wp
    restart: unless-stopped
    depends_on: [db]                    # (c) start DB first
    ports:
      - "8080:80"                       # (b) host 8080 -> container 80
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
    volumes:
      - wp_data:/var/www/html
volumes:
  db_data:
  wp_data:
```
```powershell
docker compose up -d
docker compose ps
docker compose logs -f
# open http://localhost:8080  (or use 8000:80 if Jenkins occupies 8080)
docker compose down
docker compose down -v     # deletes volumes -> DATA LOSS, avoid
```

- **(a) Two containers:** The `db` (MySQL) and `wordpress` services in one Compose file produce two containers on one default network; WordPress uses the service name `db` as DB host.
- **(b) Port mapping:** `ports: - "8080:80"` publishes container port 80 on host port 8080, so `http://localhost:8080` reaches WordPress (use 8000 if Jenkins owns 8080).
- **(c) `depends_on`:** `depends_on: [db]` makes Compose create/start the MySQL container before WordPress, avoiding initial "database not ready" errors (add a healthcheck + `condition: service_healthy` for strict ordering).
- **(d) Named volume:** `db_data:/var/lib/mysql` stores the database outside the container's writable layer, so stopping/recreating MySQL does not lose blog posts.
- **(e) Complete file:** The YAML above combines both images, `8080:80`, `depends_on`, environment variables and both named volumes in one runnable file.

---

### Question 2. Jenkins Scripted Pipeline – Build, Test & Publish [25 MARKS]

**Question:** A college wants to automate the build process of its Campus Event Management System. Create a Jenkins Pipeline that clones the CEMS source code from GitHub and executes Maven clean, install, test and package commands. The college development team has configured Maven in Jenkins and wants to use it in the CEMS pipeline. Create a Jenkins Pipeline that references the configured Maven installation and builds the application successfully. The college wants to verify the correctness of its event registration application before releasing it to students. Create a Jenkins Pipeline with separate stages for GitHub cloning, application installation and Maven testing. The college administrator wants the CEMS application pipeline to run automatically every day at 6 PM. Configure a Jenkins scheduled build trigger to execute the pipeline at the required time. The college wants a final deployable application package for its Campus Event Management System. Create a Jenkins Pipeline that clones the GitHub repository, performs Maven clean and install, executes test cases and generates a WAR package.

**One-line explanation:** Write a pipeline that clones CEMS, uses the configured `MAVEN-HOME`, runs clean/install/test/package in separate stages, archives the WAR, and uses a cron trigger at 6 PM.

**Solution:**
1. Jenkins → **New Item** → `CEMS_Pipeline` → **Pipeline** → Definition **Pipeline script** → paste the script → **Save**.
2. `tools { maven 'MAVEN-HOME' }` references the exact Maven tool name configured in Manage Jenkins → Tools (`C:\apache-maven`); JDK tool is `JDK21`.
3. Stages: `Checkout` (git clone), `Clean` (`mvn clean`), `Install` (`mvn install`), `Test` (`mvn test`), `Package` (`mvn package`) with `archiveArtifacts target/*.war`.
4. Schedule: Job → **Configure → Build Triggers → tick "Build periodically"** → Schedule `H 18 * * *` (18:00 = 6 PM daily). In a declarative Jenkinsfile use `triggers { cron('H 18 * * *') }`.
5. **Build Now** → Console Output → confirm each stage green and the WAR archived.

```groovy
pipeline {
    agent any
    tools { maven 'MAVEN-HOME'; jdk 'JDK21' }
    triggers { cron('H 18 * * *') }        // daily at 6 PM
    stages {
        stage('Checkout') { steps { bat 'git clone https://github.com/Kumbhambhargavi75/CEMS.git' } }
        stage('Clean')    { steps { bat 'mvn clean -f CEMS' } }
        stage('Install')  { steps { bat 'mvn install -f CEMS' } }
        stage('Test')     { steps { bat 'mvn test -f CEMS' } }
        stage('Package')  {
            steps {
                bat 'mvn package -f CEMS'
                archiveArtifacts artifacts: 'CEMS/target/*.war, CEMS/target/*.jar', allowEmptyArchive: true
            }
        }
    }
}
```

```powershell
git clone https://github.com/Kumbhambhargavi75/CEMS.git
cd CEMS
mvn clean install test package
Get-ChildItem target\*.war, target\*.jar -ErrorAction SilentlyContinue
```

- **Clone + Maven commands:** The `Checkout`, `Clean`, `Install`, `Test` and `Package` stages run `git clone`, `mvn clean`, `mvn install`, `mvn test`, `mvn package`.
- **Use configured Maven:** `tools { maven 'MAVEN-HOME' }` injects the Maven installation configured in Jenkins by that exact name, so `mvn` resolves correctly.
- **Separate stages for clone/install/test:** Each `stage(...)` appears separately in the Stage View; a failure in `Test` stops `Package`.
- **Daily 6 PM trigger:** **Build periodically** with `H 18 * * *`, or `triggers { cron('H 18 * * *') }` in the Jenkinsfile.
- **Final WAR package:** `mvn package` in the last stage creates `target/*.war`, and `archiveArtifacts` stores it in Jenkins for download/deployment.

---

### Question 3. Kubernetes – Deploy and Scale Application [25 MARKS]

**Question:** a) A college wants to deploy CEMS using Minikube. Create a Kubernetes Deployment using its Docker image. b) Thousands of students may register for a college fest. Scale the CEMS Deployment to 5 replicas. c) Students cannot access CEMS from their browsers. Expose the Deployment using a Node Port service. d) The college wants to distribute student requests across 5 application instances. Configure a Kubernetes Service to access the replicas. e) Deploy CEMS with 5 replicas, expose it using NodePort and verify the application in a browser using `minikube service`.

**One-line explanation:** Use Minikube to run a 5-replica CEMS Deployment and expose it with a NodePort Service, then verify through `minikube service`.

**Solution:**
1. `minikube start --driver=docker`; `kubectl get nodes` must show Ready.
2. Build and load the image: `docker build -t cems-app:latest .`, `minikube image load cems-app:latest`.
3. Create the Deployment with 5 replicas: `kubectl create deployment cems-app --image=cems-app:latest --replicas=5`; verify `kubectl get deployments`.
4. Expose with NodePort: `kubectl expose deployment cems-app --type=NodePort --port=8080 --target-port=8080`.
5. Verify service, endpoints and browser access: `kubectl get svc cems-app`, `kubectl get endpoints cems-app`, `minikube service cems-app --url` → open in browser; `kubectl get pods -o wide` shows 5 pods.

```powershell
minikube start --driver=docker
kubectl get nodes
docker build -t cems-app:latest .
minikube image load cems-app:latest
kubectl create deployment cems-app --image=cems-app:latest --replicas=5
kubectl get deployments,pods -o wide
kubectl expose deployment cems-app --type=NodePort --port=8080 --target-port=8080
kubectl get svc cems-app
kubectl get endpoints cems-app
minikube service cems-app --url
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: cems-app }
spec:
  replicas: 5
  selector:
    matchLabels: { app: cems-app }
  template:
    metadata:
      labels: { app: cems-app }
    spec:
      containers:
        - name: cems-app
          image: cems-app:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: cems-app }
spec:
  type: NodePort
  selector: { app: cems-app }
  ports:
    - port: 8080
      targetPort: 8080
```

- **(a) Deployment:** `kubectl create deployment cems-app --image=cems-app:latest` creates a ReplicaSet and pod; `kubectl get pods` verifies it is Running.
- **(b) Scale to 5:** `kubectl scale deployment cems-app --replicas=5` (or `replicas: 5` in YAML) gives five pods; `kubectl get deployments` shows `5/5`.
- **(c) NodePort service:** `kubectl expose --type=NodePort` opens a port 30000–32767 on the Minikube node so browsers outside the cluster can reach the app.
- **(d) Distribute requests:** The Service's selector `app=cems-app` matches all five pods; kube-proxy load-balances connections across their IPs, visible in `kubectl get endpoints`.
- **(e) Browser verification:** `minikube service cems-app --url` prints the NodePort URL; open it to see the CEMS homepage and confirm all pods serve it.

---

### Question 4. Nagios + Minikube Monitoring [25 MARKS]

**Question:** a) A college wants to deploy CEMS using Kubernetes. Start Minikube and create a Deployment using the application Docker image. b) Scale the CEMS Deployment to 4 replicas to handle multiple student requests. c) Expose CEMS using a NodePort service and access it through the Minikube service URL. d) The college wants to monitor its application server. Run Nagios using Docker and check the host and service status through its web dashboard. e) A new CEMS version is causing errors. Use Kubernetes commands to roll back to the previous working Deployment.

**One-line explanation:** Deploy and scale CEMS to 4 pods, expose via NodePort, monitor with Nagios, and recover from a bad release with `kubectl rollout undo`.

**Solution:**
1. `minikube start --driver=docker`, `minikube image load cems-app:latest`, `kubectl create deployment cems-app --image=cems-app:latest`.
2. Scale: `kubectl scale deployment cems-app --replicas=4`; confirm `kubectl get deployments`.
3. Expose: `kubectl expose deployment cems-app --type=NodePort --port=8080 --target-port=8080`; open `minikube service cems-app --url`.
4. Nagios: `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest` → `http://localhost:8888` (`nagiosadmin`/`nagios`) → **Hosts** (UP/DOWN, uptime) and **Services** (HTTP, load, disk — OK/WARNING/CRITICAL).
5. Rollback: `kubectl set image deployment/cems-app cems-app=cems-app:v2` (bad version) → `kubectl rollout status deployment/cems-app` shows failure → `kubectl rollout history deployment/cems-app` → `kubectl rollout undo deployment/cems-app` → `kubectl rollout status` returns success.

```powershell
minikube start --driver=docker
minikube image load cems-app:latest
kubectl create deployment cems-app --image=cems-app:latest
kubectl scale deployment cems-app --replicas=4
kubectl expose deployment cems-app --type=NodePort --port=8080 --target-port=8080
minikube service cems-app --url
# Nagios
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
# rollback
kubectl rollout history deployment/cems-app
kubectl set image deployment/cems-app cems-app=cems-app:v2
kubectl rollout status deployment/cems-app
kubectl rollout undo deployment/cems-app
kubectl rollout status deployment/cems-app
```

- **(a) Deploy on Minikube:** Start the cluster with Docker driver, load the image, and `kubectl create deployment` — `kubectl get pods` confirms Running.
- **(b) Scale to 4:** `kubectl scale deployment cems-app --replicas=4` and `kubectl get pods` shows four replicas.
- **(c) NodePort access:** `kubectl expose ... --type=NodePort` plus `minikube service cems-app --url` gives the browser URL.
- **(d) Nagios dashboard:** Run the Nagios container on 8888, log in with `nagiosadmin`/`nagios`, and read host/service states and alerts (OK/WARNING/CRITICAL).
- **(e) Rollback:** `kubectl rollout history` lists revisions; `kubectl rollout undo deployment/cems-app` restores the previous ReplicaSet/image and `kubectl rollout status` confirms success.

---

### Question 5. AWS EC2 – Maven Web Application Using Docker [25 MARKS]

**Question:** a) A college wants to host its CEMS homepage online. Launch an Ubuntu EC2 instance and deploy index.html using Docker and Nginx. b) The CEMS source code is available on GitHub. Connect to EC2 using SSH, clone the repository and create a Docker image. c) Deploy the CEMS Maven Web Project on EC2 by cloning the GitHub repository, building a Docker image and running the container. d) The college cannot connect to its EC2 instance. Configure SSH access using the downloaded `.pem` key file and troubleshoot connection issues. e) The CEMS website is not accessible through the browser. Configure EC2 Security Group inbound rules and verify access using the public IP address.

**One-line explanation:** Launch free-tier Ubuntu EC2, SSH with the `.pem`, serve CEMS via Nginx/Tomcat Docker, fix Security Group rules, and verify from the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `cems-web`, AMI **Ubuntu Server (Free tier eligible)**, **t2.micro**, create/download `.pem`, security group **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere** → Launch.
2. SSH: **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder. On Windows if the key is rejected: `icacls key.pem /inheritance:r` then `icacls key.pem /grant:r "$($env:USERNAME):(R)"`.
3. Inside: `sudo apt update && sudo apt-get install -y docker.io git`; clone `https://github.com/Kumbhambhargavi75/CEMS.git`.
4. Static homepage: `FROM nginx:alpine` + `COPY index.html`, `docker build -t cems-site .`, `docker run -d -p 80:80 cems-site`.
5. Maven Web Project: multi-stage `maven` → `tomcat:9.0` Dockerfile, `docker build -t cems-web .`, `docker run -d -p 8080:8080 --restart unless-stopped cems-web`.
6. If SSH fails: verify correct key name/region, instance is Running, port 22 open from your IP, key permissions restricted, and use the exact username `ubuntu`. If HTTP fails: **Security Groups → Edit inbound rules** → HTTP 80 and Custom TCP 8080 from `0.0.0.0/0` → Save; open `http://<public-ip>`.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/Kumbhambhargavi75/CEMS.git
cd CEMS
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
sudo docker build -t cems-site .
sudo docker run -d --name cems-site -p 80:80 cems-site
# Maven Web Project on Tomcat
cat > Dockerfile.web <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -f Dockerfile.web -t cems-web .
sudo docker run -d --name cems-web -p 8080:8080 --restart unless-stopped cems-web
curl http://localhost/ ; curl http://localhost:8080/
```

- **(a) index.html on Nginx:** Launch Ubuntu t2.micro, install Docker, write `FROM nginx:alpine`, build and run `-p 80:80`, open the public IP.
- **(b) SSH + clone + Dockerfile:** Use the SSH command from the console with the `.pem` from the correct folder; `git clone <repo>`, then create the Nginx Dockerfile shown and `docker build -t <name> .`.
- **(c) Maven Web Project:** Multi-stage Dockerfile builds the WAR with Maven and copies it into `tomcat:9.0/webapps/ROOT.war`; run with `docker run -d -p 8080:8080 cems-web`.
- **(d) SSH troubleshooting:** Check `.pem` file matches the instance and region, instance is Running, Security Group allows 22 from your IP, key permissions are restricted (`icacls`), and username is `ubuntu`; check console output if needed.
- **(e) Website not accessible:** Edit the Security Group inbound rules to allow **HTTP 80** and **Custom TCP 8080** from `0.0.0.0/0`, confirm the container is running and port-mapped, then browse `http://<public-ip>` / `:8080`.
