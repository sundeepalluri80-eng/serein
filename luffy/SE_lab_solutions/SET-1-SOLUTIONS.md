# SET-1 — Solutions

Original question paper: `SET-1.docx` (top level). Lab Internal II, Software Engineering (23CC501PC).
Repository: `https://github.com/deepthisagar7/library-management.git`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: Deployment of a Centralized Digital Library System. A university wants to deploy a Digital Library Portal for five departments using WordPress as the frontend and MySQL as the backend. The application must remain accessible during container failures, and database information must be preserved after restarting containers. Task: Design and implement a multi-container application using Docker. Expected Output: Pull MySQL and WordPress images. Create a custom Docker network and configure both containers. Establish database connectivity. Configure persistent volumes and port mapping. Demonstrate container recovery and data persistence. Access the WordPress application through a browser. Scenario based Questions: What happens if the MySQL container crashes while WordPress is running? How can database information be recovered after container deletion? Why is a custom Docker network required? How would you prevent unauthorized access to MySQL? How would you deploy five department websites using Docker?

**One-line explanation:** Run WordPress and MySQL as two Docker containers on one private network with named volumes, so the portal survives a database restart and keeps its data.

**Solution:**
1. Pull the images: `docker pull mysql:8.0` and `docker pull wordpress:latest`.
2. Create a user-defined bridge network (gives name-based DNS) and two named volumes: `docker network create library-net`, `docker volume create mysql-data`, `docker volume create wp-data`.
3. Start MySQL on `library-net` only — do **not** publish 3306. Set `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`. Add `--restart unless-stopped` so it auto-recovers.
4. Start WordPress with `-p 8000:80` (port 8080 is used by Jenkins) on the same network, with `WORDPRESS_DB_HOST=library-mysql:3306` — the container name resolves through Docker DNS.
5. Verify with `docker ps` and `docker logs -f library-wordpress`, open `http://localhost:8000` in the browser and finish the WordPress 5-minute installation.
6. Recovery/persistence demo: `docker stop library-mysql` → the site shows "Error establishing a database connection"; `docker start library-mysql` → the site recovers. `docker rm -f library-mysql` and re-run the same `docker run` command → posts and users are still there because they live in the `mysql-data` volume.

```powershell
docker pull mysql:8.0
docker pull wordpress:latest
docker network create library-net
docker volume create mysql-data
docker volume create wp-data

docker run -d --name library-mysql --network library-net --restart unless-stopped `
  -v mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=library `
  -e MYSQL_USER=libuser -e MYSQL_PASSWORD=libpass mysql:8.0

docker run -d --name library-wordpress --network library-net --restart unless-stopped `
  -p 8000:80 -v wp-data:/var/www/html `
  -e WORDPRESS_DB_HOST=library-mysql:3306 -e WORDPRESS_DB_NAME=library `
  -e WORDPRESS_DB_USER=libuser -e WORDPRESS_DB_PASSWORD=libpass wordpress:latest

docker ps
docker logs -f library-wordpress
# recovery test
docker stop library-mysql
docker start library-mysql
```

Alternative single-file version (`docker-compose.yml`):

```yaml
services:
  db:
    image: mysql:8.0
    container_name: library-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: library
      MYSQL_USER: libuser
      MYSQL_PASSWORD: libpass
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [library-net]
  wordpress:
    image: wordpress:latest
    container_name: library-wp
    restart: unless-stopped
    depends_on: [db]
    ports: ["8000:80"]
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: library
      WORDPRESS_DB_USER: libuser
      WORDPRESS_DB_PASSWORD: libpass
    volumes:
      - wp-data:/var/www/html
    networks: [library-net]
volumes:
  mysql-data:
  wp-data:
networks:
  library-net:
```

- **What happens if the MySQL container crashes while WordPress is running?** WordPress cannot query the database and shows "Error establishing a database connection". With `--restart unless-stopped` Docker restarts MySQL automatically and WordPress reconnects; data is safe in the named volume.
- **How can database information be recovered after container deletion?** Remove the container only (`docker rm -f library-mysql`), then run the same `docker run` with `-v mysql-data:/var/lib/mysql`. The volume survives container deletion, so all tables return.
- **Why is a custom Docker network required?** A user-defined bridge network provides built-in DNS, so WordPress reaches the database by the name `library-mysql` instead of a changing IP. The default bridge has no automatic name resolution and puts both containers in a less isolated network.
- **How would you prevent unauthorized access to MySQL?** Never publish `-p 3306:3306`; keep MySQL only on the internal network, use a non-root `MYSQL_USER` with a strong password, restrict the DB user to the WordPress host, and store secrets in Docker secrets/environment files rather than source code.
- **How would you deploy five department websites using Docker?** Reuse the same image and run five WordPress containers (different names, e.g. `cse-wp`, `ece-wp`) each with its own DB name/volume and a distinct host port (8001–8005), ideally behind a reverse proxy/Nginx on port 80.

---

### Question 2. Jenkins – Java Application Build and Continuous Integration [20 MARKS]

**Question:** Scenario: Automated Build and Testing of Library Management System. A software company has five developers working on a Java-based Library Management System. Developers frequently update the GitHub repository. The company wants Jenkins and Maven to automate compilation, testing and artifact generation. Task: Implement an automated Continuous Integration workflow using Jenkins Freestyle Project. Expected Output: Configure JDK, Maven and Jenkins plugins. Integrate the GitHub repository with Jenkins. Configure build triggers and Maven goals. Execute compilation and test cases. Archive the generated WAR/JAR artifact. Demonstrate successful and failed builds. Scenario based Questions: How does Jenkins identify compilation errors? What happens if compilation succeeds but test cases fail? Why is Maven used instead of manually downloading JAR files? How can Jenkins build only a specific GitHub branch? What is the difference between Continuous Integration and Continuous Deployment?

**One-line explanation:** Create a Jenkins Freestyle job that pulls the repo, runs Maven goals, and archives the built JAR/WAR, with a webhook trigger so every push is built automatically.

**Solution:**
1. Pre-check tools: Manage Jenkins → Plugins → install `git`, `maven-plugin`, `github`, `copyartifact`, `ws-cleanup`, `build-pipeline-plugin`, `email-ext`. Manage Jenkins → Tools → Maven name must be exactly `MAVEN-HOME` (`C:\apache-maven`), JDK name `JDK21`.
2. Jenkins dashboard (`http://localhost:8080`, no login) → click **New Item** → name `LibraryMS_Build` → choose **Freestyle project** → **OK**.
3. Configure → **Source Code Management** → **Git** → Repository URL `https://github.com/deepthisagar7/library-management.git`; **Branches to build** `*/main`.
4. **Build Triggers** → tick **GitHub hook trigger for GITScm polling** (or **Poll SCM** with `H/5 * * * *` if no webhook).
5. **Add build step** → **Invoke top-level Maven targets** → Maven version `MAVEN-HOME` → Goals `clean package`. Add a second step with `test` if tests are separate.
6. **Add post-build action** → **Archive the artifacts** → `target/*.jar` (use `target/*.war` for a web project). Click **Save**.
7. Click **Build Now**, then **Console Output**. Show a failed build by temporarily breaking a Java file (red ball, `COMPILATION ERROR`), then fix and rebuild to get a blue/green ball and the archived artifact.

```powershell
git clone https://github.com/deepthisagar7/library-management.git
cd library-management
mvn -f pom.xml clean package        # same goals Jenkins runs
Get-ChildItem target\*.jar
# webhook tunnel (GitHub cannot reach localhost)
ngrok http 8080
```

- **How does Jenkins identify compilation errors?** The Maven/compiler writes errors into the console log and returns a non-zero exit code; Jenkins marks the build **FAILED** (red ball) and shows lines such as `[ERROR] COMPILATION ERROR`.
- **What happens if compilation succeeds but test cases fail?** `mvn test` exits non-zero, so the build is marked failed/unstable, the artifact is not archived and downstream jobs are not triggered — the faulty code is blocked.
- **Why is Maven used instead of manually downloading JAR files?** Maven reads `pom.xml`, downloads the correct dependency versions from the repository automatically, and gives a standard lifecycle (`clean → compile → test → package`) so builds are repeatable on any machine.
- **How can Jenkins build only a specific GitHub branch?** In SCM → **Branches to build** enter `*/main` (or `*/master`); Jenkins then clones only that ref. In a Pipeline use `branches: [[name: '*/main']]`.
- **What is the difference between Continuous Integration and Continuous Deployment?** CI automatically builds and tests every commit; CD goes further and automatically releases the validated artifact to a server (staging/production).

---

### Question 3. Kubernetes – Container Deployment and Auto Scaling [20 MARKS]

**Question:** Scenario: Managing Heavy Traffic in a University Digital Library. During examinations, thousands of students access the Digital Library simultaneously. The university wants to deploy multiple application replicas using Kubernetes and ensure automatic recovery when containers fail. Task: Deploy, scale and manage the application using Kubernetes. Expected Output: Start a Kubernetes cluster using Minikube. Create a Deployment with two replicas. Expose the application using a Kubernetes Service. Scale the deployment from two to five replicas. Demonstrate automatic pod recreation after failure. Perform rolling update and rollback. Scenario based Questions: What happens when one pod crashes? How does Kubernetes maintain the desired number of replicas? What is the difference between manual scaling and auto scaling? Why is a Kubernetes Service required? How can you recover from a failed application update?

**One-line explanation:** Use Minikube + kubectl to run a 2-replica Deployment, expose it with a NodePort Service, scale to 5, and prove self-healing plus rollout/rollback.

**Solution:**
1. Start the local cluster with Docker as the driver: `minikube start --driver=docker`, then `minikube status` and `kubectl get nodes` (node must be **Ready**).
2. Create the Deployment with two replicas: `kubectl create deployment library-app --image=nginx --replicas=2`.
3. Expose it: `kubectl expose deployment library-app --type=NodePort --port=80 --target-port=80`. Get the URL with `minikube service library-app --url` and open it in a browser.
4. Scale up: `kubectl scale deployment library-app --replicas=5`, then `kubectl get pods` — five pods appear.
5. Self-healing demo: note a pod name, `kubectl delete pod <name>`, then `kubectl get pods` — a new pod is created automatically by the ReplicaSet.
6. Rolling update + rollback: `kubectl set image deployment/library-app nginx=nginx:1.27`, watch with `kubectl rollout status deployment/library-app`; if it fails, `kubectl rollout undo deployment/library-app` and `kubectl rollout history deployment/library-app`.

```powershell
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl create deployment library-app --image=nginx --replicas=2
kubectl expose deployment library-app --type=NodePort --port=80 --target-port=80
kubectl get deployments,pods,services
minikube service library-app --url
kubectl scale deployment library-app --replicas=5
kubectl get pods -o wide
kubectl delete pod <pod-name>
kubectl set image deployment/library-app nginx=nginx:1.27
kubectl rollout status deployment/library-app
kubectl rollout undo deployment/library-app
```

Equivalent YAML (`deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: library-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: library-app
  template:
    metadata:
      labels:
        app: library-app
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: library-app
spec:
  type: NodePort
  selector:
    app: library-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

- **What happens when one pod crashes?** The ReplicaSet controller notices only 4 of 5 desired pods and immediately creates a replacement pod; users may see a brief dip in capacity but the service keeps working.
- **How does Kubernetes maintain the desired number of replicas?** The Deployment controller creates a ReplicaSet whose `spec.replicas` is compared continuously with the live pod count; any difference triggers create/delete of pods (self-healing).
- **What is the difference between manual scaling and auto scaling?** Manual scaling is `kubectl scale` / changing `spec.replicas` yourself; auto scaling uses the Horizontal Pod Autoscaler (`kubectl autoscale`) to change replicas from CPU/metrics automatically.
- **Why is a Kubernetes Service required?** Pods are ephemeral and get changing IPs; a Service gives a stable virtual IP/DNS and load-balances requests across all matching pods, so clients never track individual pods.
- **How can you recover from a failed application update?** Run `kubectl rollout undo deployment/library-app` to return to the previous ReplicaSet; use `kubectl rollout history` and `kubectl rollout status` to inspect and confirm.

---

### Question 4. Nagios – Infrastructure and Application Monitoring [20 MARKS]

**Question:** Scenario: Continuous Monitoring of Library Management Infrastructure. A university experiences application downtime, slow response times and database connection failures. The administrator wants to use Nagios to monitor server performance and application availability and generate alerts when services fail. Task: Install and configure Nagios monitoring for the Library Management infrastructure. Expected Output: Pull and run the Nagios Docker image. Configure host and service monitoring. Monitor CPU, memory, disk and server uptime. Configure HTTP and database service checks. Define WARNING and CRITICAL thresholds. Demonstrate alerts and recovery through the Nagios dashboard. Scenario based Questions: How does Nagios identify high CPU utilization? What is the difference between WARNING and CRITICAL? How can Nagios identify application failure when the server is running? What happens if the Nagios monitoring server fails? How would you identify whether downtime is caused by Tomcat, MySQL or networking?

**One-line explanation:** Run Nagios Core in Docker, log into its web UI, verify host/service status, add HTTP + MySQL checks with thresholds, and watch a service go CRITICAL then recover.

**Solution:**
1. Pull and run: `docker pull jasonrivers/nagios:latest`, `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest` (8888 keeps clear of Jenkins 8080 and Tomcat 8085).
2. Open `http://localhost:8888` in the browser; log in with `nagiosadmin` / `nagios`.
3. In the UI click **Hosts** → `localhost` to see host uptime/state, then **Services** to see `load`, `disk`, `users` and `HTTP` checks with their status, last-check time and retry count.
4. Add an HTTP check with thresholds by editing the config inside the container and restarting it (example below): `check_http` on the app URL, `check_mysql` for the DB. WARNING = threshold crossed, CRITICAL = service down.
5. Demonstrate an alert: stop the WordPress container → the HTTP service turns **CRITICAL** (red); restart it → recovery notification and state returns **OK** (green). Use **Service detail → Acknowledge** in the UI to silence it while fixing.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
# open http://localhost:8888  ->  nagiosadmin / nagios
docker logs -f nagiosdemo
# add an HTTP service check with WARNING/CRITICAL response-time thresholds
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description Library_HTTP
  check_command       check_http!-H localhost -p 8000 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
# alert/recovery demo
docker stop library-wordpress
docker start library-wordpress
docker stop nagiosdemo; docker rm nagiosdemo
```

- **How does Nagios identify high CPU utilization?** The `check_load`/`check_cpu` plugin samples the 1/5/15-minute load average and compares it with configured thresholds; crossing the warning level gives WARNING, crossing critical gives CRITICAL and triggers a notification.
- **What is the difference between WARNING and CRITICAL?** WARNING means a threshold is crossed but the service still works (e.g. disk 80%); CRITICAL means the service has failed or crossed the hard limit (e.g. disk 95%, HTTP refused) and needs immediate action.
- **How can Nagios identify application failure when the server is running?** By checking the application itself — `check_http -p 8000` on the site URL or `check_mysql` on the DB port — rather than only ping; the host stays UP while the service is CRITICAL.
- **What happens if the Nagios monitoring server fails?** Monitoring and alerts stop — failures go unnoticed (a monitoring blind spot). Fix by running a second Nagios instance or an external monitoring service that watches the first one.
- **How would you identify whether downtime is caused by Tomcat, MySQL or networking?** Check the Nagios service states: `check_ping`/host UP rules out networking; HTTP check CRITICAL while MySQL check OK points to Tomcat; MySQL check CRITICAL points to the database; `docker logs <container>` confirms.

---

### Question 5. AWS – Cloud Deployment using EC2 [20 MARKS]

**Question:** Scenario: Migration of University Library Application to AWS Cloud. A university wants to migrate its Library Management System from a local server to AWS. Students and faculty should access the application remotely. The deployment must provide secure access, database connectivity and server monitoring. Task: Deploy the Library Management application on AWS using an EC2 instance. Scenario based Questions: What could cause an EC2 application to be inaccessible through a browser? Why should SSH access be restricted? What happens if Tomcat stops while EC2 is running? How can application availability be improved? Why would an Application Load Balancer and multiple EC2 instances be better than a single EC2 instance?

**One-line explanation:** Launch a free-tier Ubuntu EC2 instance, open only the needed ports, SSH in, install Docker/Git, build and run the app container, then verify it from the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**. Name: `library-app`. AMI: **Ubuntu Server (Free tier eligible)**. Architecture 64-bit. Instance type **t2.micro**.
2. **Key pair** → **Create new key pair** → `.pem` → download and store in `C:\Users\abhir\aws`.
3. **Network settings** → **Create security group**: Inbound **SSH (22) → My IP**, **HTTP (80) → Anywhere**, **Custom TCP 8080 → Anywhere** (app port). Launch instance.
4. Select the instance → **Connect** → **SSH client** tab → copy `ssh -i key.pem ubuntu@<public-ip>`. Run it from the folder containing the `.pem`; on Windows fix permissions first with `icacls`.
5. Inside the instance: `sudo apt update && sudo apt-get install -y docker.io git`, clone the repo, write a Dockerfile, `docker build -t library-app .`, `docker run -d -p 8080:8080 library-app`.
6. Open `http://<public-ip>:8080` in a browser. For monitoring: AWS Console → **CloudWatch** → **Alarms** → **Create alarm** on `CPUUtilization` and configure an SNS email action.

```powershell
# Windows: prepare the key, then SSH
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside the Ubuntu EC2 instance ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/deepthisagar7/library-management.git
cd library-management
cat > Dockerfile <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM eclipse-temurin:17-jre
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app.jar"]
EOF
sudo docker build -t library-app .
sudo docker run -d --name library -p 8080:8080 library-app
curl http://localhost:8080
```

- **What could cause an EC2 application to be inaccessible through a browser?** The Security Group does not allow the app port, the app listens only on 127.0.0.1 instead of 0.0.0.0, the container is not running/ported correctly, the OS firewall blocks it, or the public IP changed after a stop/start.
- **Why should SSH access be restricted?** Port 22 is a management entry point; allowing it from anywhere invites brute-force attacks. Restrict it to **My IP** (or a bastion host) and use key-based authentication only.
- **What happens if Tomcat stops while EC2 is running?** The instance stays reachable but the web app returns connection-refused/502. Restart it (`sudo systemctl restart tomcat` or `docker restart <container>`) and check CloudWatch/logs for the cause.
- **How can application availability be improved?** Run the app as a Docker container with `--restart unless-stopped`, put it behind an Application Load Balancer with multiple EC2 instances in different Availability Zones, add health checks, Auto Scaling, EBS-backed data and CloudWatch alarms.
- **Why would an ALB and multiple EC2 instances be better than a single EC2 instance?** A single instance is a single point of failure; an ALB spreads traffic across healthy instances, removes unhealthy ones via health checks, and Auto Scaling replaces failed instances — giving high availability and fault tolerance.
