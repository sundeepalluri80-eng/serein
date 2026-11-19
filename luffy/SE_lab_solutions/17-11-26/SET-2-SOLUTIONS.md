# 17-11-26 / SET-2 — Solutions

Original question paper: `17-11-26\SET-2.docx`. Lab Internal III, Software Engineering (23CC501PC), dated 17/11/2026.
Repository: `https://github.com/kvrajanirajesh/mid2-hospitalmaven`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: A university wants to deploy a Hospital Management System for multiple departments using WordPress as the frontend and MySQL as the backend. The application must remain accessible during container failures, and patient, doctor and appointment information must be preserved after restarting containers. Task: Design and implement the required solution. Expected Output: 1. Pull MySQL and WordPress images. 2. Create a custom Docker network and configure both containers. 3. Establish database connectivity. 4. Configure persistent volumes and port mapping. 5. Demonstrate container recovery and data persistence. 6. Access the WordPress application through a browser. Scenario based Questions: 1. What happens if the MySQL container crashes while the Hospital Management System is running? 2. How can patient, doctor and appointment information be recovered after container deletion? 3. Why is a custom Docker network required? 4. How would you prevent unauthorized access to MySQL? 5. How would you deploy multiple Hospital Management System instances using Docker?

**One-line explanation:** Run WordPress + MySQL on one private Docker network with named volumes so patient, doctor and appointment data survives restarts.

**Solution:**
1. `docker pull mysql:8.0` and `docker pull wordpress:latest`.
2. `docker network create hospital-net`, `docker volume create hospital-mysql-data`, `docker volume create hospital-wp-data`.
3. Start MySQL on `hospital-net` with `MYSQL_DATABASE=hospital`, `MYSQL_USER=hospitaluser`, `MYSQL_PASSWORD=hospitalpass`, `--restart unless-stopped`; do not publish 3306.
4. Start WordPress on the network with `-p 8000:80` (Jenkins owns 8080) and `WORDPRESS_DB_HOST=hospital-mysql:3306`.
5. `docker ps`, `docker logs -f hospital-wp`, open `http://localhost:8000` and finish the WordPress install.
6. Recovery demo: `docker stop hospital-mysql` → site errors; `docker start hospital-mysql` → recovers; `docker rm -f hospital-mysql` + re-run with the same volume → all records intact.

```powershell
docker pull mysql:8.0
docker pull wordpress:latest
docker network create hospital-net
docker volume create hospital-mysql-data
docker volume create hospital-wp-data
docker run -d --name hospital-mysql --network hospital-net --restart unless-stopped `
  -v hospital-mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=hospital `
  -e MYSQL_USER=hospitaluser -e MYSQL_PASSWORD=hospitalpass mysql:8.0
docker run -d --name hospital-wp --network hospital-net --restart unless-stopped `
  -p 8000:80 -v hospital-wp-data:/var/www/html `
  -e WORDPRESS_DB_HOST=hospital-mysql:3306 -e WORDPRESS_DB_NAME=hospital `
  -e WORDPRESS_DB_USER=hospitaluser -e WORDPRESS_DB_PASSWORD=hospitalpass wordpress:latest
docker ps
docker logs -f hospital-wp
docker stop hospital-mysql
docker start hospital-mysql
```

- **What happens if the MySQL container crashes while the Hospital Management System is running?** Pages show "Error establishing a database connection" and appointments cannot be saved; `--restart unless-stopped` brings MySQL back and WordPress reconnects, with data safe in the volume.
- **How can patient, doctor and appointment information be recovered after container deletion?** Delete only the container and re-run with `-v hospital-mysql-data:/var/lib/mysql`; the named volume retains the database.
- **Why is a custom Docker network required?** It gives name-based DNS (`hospital-mysql`) and isolates the containers, unlike the default bridge which has no automatic name resolution.
- **How would you prevent unauthorized access to MySQL?** Keep 3306 unpublished/internal, use a least-privilege DB user with a strong password, store secrets safely, and restrict DB access to the WordPress container.
- **How would you deploy multiple Hospital Management System instances using Docker?** Run several WordPress containers (`hospital-wp-1..n`) on different host ports with separate DB names/volumes, behind an Nginx reverse proxy.

---

### Question 2. Jenkins – Java Application Build and Continuous Integration [20 MARKS]

**Question:** Scenario: A software company has five developers working on a Java-based Hospital Management System. Developers frequently update the GitHub repository. The company wants Jenkins and Maven to automate compilation, testing and artifact generation. Task: Design and implement the required solution. Expected Output: 1. Configure JDK, Maven and Jenkins plugins. 2. Integrate the GitHub repository with Jenkins. 3. Configure build triggers and Maven goals. 4. Execute compilation and test cases. 5. Archive the generated WAR/JAR artifact. 6. Demonstrate successful and failed builds. Scenario based Questions: 1. How does Jenkins identify compilation errors? 2. What happens if compilation succeeds but test cases fail? 3. Why is Maven used instead of manually downloading JAR files? 4. How can Jenkins build only a specific GitHub branch? 5. What is the difference between Continuous Integration and Continuous Deployment?

**One-line explanation:** Configure a Jenkins Freestyle job for the hospital Maven repo that compiles, tests, archives the artifact and runs on each push.

**Solution:**
1. Manage Jenkins → Plugins: verify `git`, `maven-plugin`, `github`, `copyartifact`, `ws-cleanup`, `email-ext`. Manage Jenkins → Tools: Maven `MAVEN-HOME` → `C:\apache-maven`, JDK `JDK21`.
2. **New Item** → `HospitalMS_Build` → **Freestyle project** → **OK** → **Source Code Management → Git** → URL `https://github.com/kvrajanirajesh/mid2-hospitalmaven`, branch `*/main` or `*/master`.
3. **Build Triggers** → **GitHub hook trigger for GITScm polling** (or Poll SCM `H/5 * * * *`).
4. **Add build step → Invoke top-level Maven targets** → `MAVEN-HOME` → Goals `clean package` (plus `install`/`test` steps as needed).
5. **Add post-build action → Archive the artifacts** → `target/*.jar` or `target/*.war`. Save.
6. **Build Now** → **Console Output**: broken code gives a red failed build with `[ERROR] COMPILATION ERROR`; fixed code gives a green build with the archived artifact.

```powershell
git clone https://github.com/kvrajanirajesh/mid2-hospitalmaven.git
cd mid2-hospitalmaven
mvn -f pom.xml clean package
Get-ChildItem target\*.jar, target\*.war -ErrorAction SilentlyContinue
ngrok http 8080
```

- **How does Jenkins identify compilation errors?** Maven/`javac` prints the error with file and line and exits non-zero; Jenkins marks the build FAILED and highlights it in Console Output.
- **If compilation succeeds but tests fail?** `mvn test` exits non-zero → build failed/unstable, no artifact archived and no downstream deployment.
- **Why Maven instead of manual JAR downloads?** `pom.xml` drives automatic dependency resolution in correct versions and a repeatable standard lifecycle.
- **Build only a specific branch?** SCM → **Branches to build** = `*/main` (or `*/master`).
- **CI vs CD:** CI builds and tests each commit automatically; CD also deploys the validated artifact automatically.

---

### Question 3. Kubernetes – Container Deployment and Auto Scaling [20 MARKS]

**Question:** Scenario: During appointment booking hours, thousands of users access the Hospital Management System simultaneously. The organization wants to deploy multiple application replicas using Kubernetes and ensure automatic recovery when containers fail. Task: Design and implement the required solution. Expected Output: 1. Start a Kubernetes cluster using Minikube. 2. Create a Deployment with two replicas. 3. Expose the application using a Kubernetes Service. 4. Scale the deployment from two to five replicas. 5. Demonstrate automatic pod recreation after failure. 6. Perform rolling update and rollback. Scenario based Questions: 1. What happens when one pod crashes? 2. How does Kubernetes maintain the desired number of replicas? 3. What is the difference between manual scaling and auto scaling? 4. Why is a Kubernetes Service required? 5. How can you recover from a failed application update?

**One-line explanation:** Deploy the hospital app on Minikube with 2 replicas, expose it via NodePort, scale to 5, and verify self-healing plus rollback.

**Solution:**
1. `minikube start --driver=docker`; `kubectl get nodes` → Ready; `minikube image load hospital-app:latest`.
2. `kubectl create deployment hospital-app --image=hospital-app:latest --replicas=2`; `kubectl get pods`.
3. `kubectl expose deployment hospital-app --type=NodePort --port=8080 --target-port=8080`; open `minikube service hospital-app --url`.
4. `kubectl scale deployment hospital-app --replicas=5`; confirm `5/5` with `kubectl get deployments`.
5. `kubectl delete pod <name>` → new pod created automatically.
6. `kubectl set image deployment/hospital-app hospital-app=hospital-app:v2`, watch `kubectl rollout status`, then `kubectl rollout undo deployment/hospital-app` to recover from a bad release.

```powershell
minikube start --driver=docker
kubectl get nodes
minikube image load hospital-app:latest
kubectl create deployment hospital-app --image=hospital-app:latest --replicas=2
kubectl expose deployment hospital-app --type=NodePort --port=8080 --target-port=8080
kubectl get deployments,pods,services
minikube service hospital-app --url
kubectl scale deployment hospital-app --replicas=5
kubectl delete pod <pod-name>
kubectl set image deployment/hospital-app hospital-app=hospital-app:v2
kubectl rollout status deployment/hospital-app
kubectl rollout undo deployment/hospital-app
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: hospital-app }
spec:
  replicas: 2
  selector:
    matchLabels: { app: hospital-app }
  template:
    metadata:
      labels: { app: hospital-app }
    spec:
      containers:
        - name: hospital-app
          image: hospital-app:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: hospital-app }
spec:
  type: NodePort
  selector: { app: hospital-app }
  ports: [{ port: 8080, targetPort: 8080, nodePort: 30080 }]
```

- **What happens when one pod crashes?** The ReplicaSet replaces it so the desired replica count is restored; the Service sends traffic only to ready pods.
- **How are replicas maintained?** The ReplicaSet controller compares `spec.replicas` to live pods and creates/deletes as needed (self-healing).
- **Manual vs auto scaling:** `kubectl scale` sets replicas manually; HPA (`kubectl autoscale`) adjusts them from CPU/metrics automatically.
- **Why a Service?** Pod IPs are temporary; a Service gives a stable address and load-balances across the replicas.
- **Recover from a failed update?** `kubectl rollout undo deployment/hospital-app` returns to the previous revision; verify with `rollout history`/`rollout status`.

---

### Question 4. Nagios – Infrastructure and Application Monitoring [20 MARKS]

**Question:** Scenario: The organization experiences application downtime, slow response times and database connection failures. The administrator wants to use Nagios to monitor server performance and application availability and generate alerts when services fail. Task: Design and implement the required solution. Expected Output: 1. Pull and run the Nagios Docker image. 2. Configure host and service monitoring. 3. Monitor CPU, memory, disk and server uptime. 4. Configure HTTP and database service checks. 5. Define WARNING and CRITICAL thresholds. 6. Demonstrate alerts and recovery through the Nagios dashboard. Scenario based Questions: 1. How does Nagios identify high CPU utilization? 2. What is the difference between WARNING and CRITICAL? 3. How can Nagios identify application failure when the server is running? 4. What happens if the Nagios monitoring server fails? 5. How would you identify whether downtime is caused by Tomcat, MySQL or networking?

**One-line explanation:** Run Nagios in Docker, inspect host/service status, add HTTP and DB checks with thresholds, and demonstrate alerts and recovery.

**Solution:**
1. `docker pull jasonrivers/nagios:latest`; `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`.
2. `http://localhost:8888` → login `nagiosadmin` / `nagios`.
3. **Hosts** → `localhost` (UP/DOWN, uptime); **Services** → `load`, `disk`, `users`, `HTTP` with states and thresholds.
4. Add `check_http` on the hospital URL (`-w 2 -c 5`) and `check_mysql` for the DB by appending to the localhost config inside the container, then `docker restart nagiosdemo`.
5. Stop `hospital-wp` → HTTP CRITICAL + alert; restart → OK. Use **Acknowledge** while investigating.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description Hospital_HTTP
  check_command       check_http!-H localhost -p 8000 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
docker stop hospital-wp
docker start hospital-wp
```

- **How does Nagios identify high CPU utilization?** `check_load` compares load averages against configured warning/critical limits and changes state/notifies when exceeded.
- **WARNING vs CRITICAL:** WARNING = threshold crossed but usable; CRITICAL = failed/hard limit and needs immediate action.
- **Application failure when server runs:** HTTP/MySQL service checks fail while the host remains UP, isolating the fault to the application.
- **If Nagios fails:** Monitoring stops and failures go unnoticed — use a second instance or external monitoring service.
- **Tomcat vs MySQL vs networking?** Host DOWN → network; host UP + HTTP CRITICAL + MySQL OK → Tomcat; host UP + MySQL CRITICAL → database; confirm via `docker logs`.

---

### Question 5. AWS – Cloud Deployment using EC2 [20 MARKS]

**Question:** Scenario: The organization wants to migrate its Hospital Management System from a local server to AWS. Doctors, staff and patients should access the application remotely. The deployment must provide secure access, database connectivity and server monitoring. Task: Design and implement the required solution. Expected Output: 1. Launch and configure an EC2 instance. 2. Configure key pairs and security groups. 3. Connect through SSH and install Java, Maven and Tomcat. 4. Clone the GitHub repository and build the application. 5. Deploy the application on Tomcat. 6. Configure database connectivity and CloudWatch monitoring. 7. Access the application using the EC2 public IP address. Scenario based Questions: 1. What could cause an EC2 application to be inaccessible through a browser? 2. Why should SSH access be restricted? 3. What happens if Tomcat stops while EC2 is running? 4. How can application availability be improved? 5. Why would an Application Load Balancer and multiple EC2 instances be better than a single EC2 instance?

**One-line explanation:** Launch free-tier Ubuntu EC2, open the right ports, SSH in, install Java/Maven/Tomcat, build and deploy the WAR, configure CloudWatch and verify from the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `hospital-app`, AMI **Ubuntu Server (Free tier eligible)**, **t2.micro**.
2. Create `.pem`; Security Group inbound **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere**; launch.
3. **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (`icacls` on Windows if needed).
4. Inside: `sudo apt update && sudo apt install -y openjdk-17-jdk maven tomcat9 git`; `git clone https://github.com/kvrajanirajesh/mid2-hospitalmaven.git`; `mvn clean package`; `sudo cp target/*.war /var/lib/tomcat9/webapps/`; `sudo systemctl restart tomcat9`.
5. Set DB connection properties (host/user/password) in the app configuration; for a containerised DB use the internal network name.
6. CloudWatch → **Alarms → Create alarm** on `CPUUtilization` with an SNS email action.
7. Open `http://<public-ip>:8080/<context>` in a browser.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update
sudo apt install -y openjdk-17-jdk maven tomcat9 git
git clone https://github.com/kvrajanirajesh/mid2-hospitalmaven.git
cd mid2-hospitalmaven
mvn clean package
sudo cp target/*.war /var/lib/tomcat9/webapps/
sudo systemctl restart tomcat9
sudo apt install -y docker.io
sudo docker build -t hospital-app .
sudo docker run -d -p 8080:8080 --restart unless-stopped hospital-app
curl http://localhost:8080
```

- **Inaccessible via browser:** Security Group missing 8080, Tomcat/container not running or port-mapped, app bound to 127.0.0.1, OS firewall, wrong context path, or public IP changed.
- **Restrict SSH:** Port 22 is administrative; allow only your IP/key to avoid brute-force attacks.
- **Tomcat stops while EC2 runs:** Server stays up but site is down; restart Tomcat/container and check logs.
- **Improve availability:** Restart policies, ALB + multiple instances across AZs, Auto Scaling, health checks, RDS/EBS and CloudWatch alarms.
- **ALB + multiple instances vs single:** Removes the single point of failure, balances load, replaces unhealthy instances and scales on demand.
