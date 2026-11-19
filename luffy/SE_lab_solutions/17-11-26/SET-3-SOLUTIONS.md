# 17-11-26 / SET-3 — Solutions

Original question paper: `17-11-26\SET-3.docx`. Lab Internal II, Software Engineering (23CC501PC), dated 17/11/2026.
Repository: `https://github.com/kvrajanirajesh/mid1-e-learningmaven`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: A university wants to deploy a E-Learning Management System for multiple departments using WordPress as the frontend and MySQL as the backend. The application must remain accessible during container failures, and course, student and enrollment information must be preserved after restarting containers. Task: Design and implement the required solution. Expected Output: 1. Pull MySQL and WordPress images. 2. Create a custom Docker network and configure both containers. 3. Establish database connectivity. 4. Configure persistent volumes and port mapping. 5. Demonstrate container recovery and data persistence. 6. Access the WordPress application through a browser. Scenario based Questions: 1. What happens if the MySQL container crashes while the E-Learning Management System is running? 2. How can course, student and enrollment information be recovered after container deletion? 3. Why is a custom Docker network required? 4. How would you prevent unauthorized access to MySQL? 5. How would you deploy multiple E-Learning Management System instances using Docker?

**One-line explanation:** Run WordPress + MySQL on a private Docker network with named volumes so course, student and enrolment data survives container restarts.

**Solution:**
1. `docker pull mysql:8.0` and `docker pull wordpress:latest`.
2. `docker network create elearn-net`, `docker volume create elearn-mysql-data`, `docker volume create elearn-wp-data`.
3. Start MySQL on `elearn-net` with `MYSQL_DATABASE=elearning`, `MYSQL_USER=elearnuser`, `MYSQL_PASSWORD=elearnpass`, `--restart unless-stopped`; keep 3306 internal.
4. Start WordPress on the same network with `-p 8000:80` and `WORDPRESS_DB_HOST=elearn-mysql:3306` (name resolved by Docker DNS).
5. `docker ps`, `docker logs -f elearn-wp`, open `http://localhost:8000`, finish the WordPress setup.
6. `docker stop elearn-mysql` → site errors; `docker start elearn-mysql` → recovers; `docker rm -f elearn-mysql` and re-run with the same volume → all courses/students intact.

```powershell
docker pull mysql:8.0
docker pull wordpress:latest
docker network create elearn-net
docker volume create elearn-mysql-data
docker volume create elearn-wp-data
docker run -d --name elearn-mysql --network elearn-net --restart unless-stopped `
  -v elearn-mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=elearning `
  -e MYSQL_USER=elearnuser -e MYSQL_PASSWORD=elearnpass mysql:8.0
docker run -d --name elearn-wp --network elearn-net --restart unless-stopped `
  -p 8000:80 -v elearn-wp-data:/var/www/html `
  -e WORDPRESS_DB_HOST=elearn-mysql:3306 -e WORDPRESS_DB_NAME=elearning `
  -e WORDPRESS_DB_USER=elearnuser -e WORDPRESS_DB_PASSWORD=elearnpass wordpress:latest
docker ps
docker logs -f elearn-wp
docker stop elearn-mysql
docker start elearn-mysql
```

- **What happens if the MySQL container crashes?** The portal shows "Error establishing a database connection" and enrolments fail; `--restart unless-stopped` restarts MySQL and the site recovers, with data in the named volume.
- **How can course, student and enrollment information be recovered after container deletion?** Delete only the container and re-run with `-v elearn-mysql-data:/var/lib/mysql`; the volume keeps the data.
- **Why is a custom Docker network required?** It provides container-name DNS (`elearn-mysql`) and isolation, which the default bridge does not.
- **How would you prevent unauthorized access to MySQL?** Never publish 3306, use a least-privilege user with a strong password, keep the DB on the internal network, and store secrets outside source code.
- **How would you deploy multiple E-Learning instances?** Run several WordPress containers with distinct names, ports (8001–8005) and DBs/volumes, behind an Nginx reverse proxy.

---

### Question 2. Jenkins – Java Application Build and Continuous Integration [20 MARKS]

**Question:** Scenario: A software company has five developers working on a Java-based E-Learning Management System. Developers frequently update the GitHub repository. The company wants Jenkins and Maven to automate compilation, testing and artifact generation. Task: Design and implement the required solution. Expected Output: 1. Configure JDK, Maven and Jenkins plugins. 2. Integrate the GitHub repository with Jenkins. 3. Configure build triggers and Maven goals. 4. Execute compilation and test cases. 5. Archive the generated WAR/JAR artifact. 6. Demonstrate successful and failed builds. Scenario based Questions: 1. How does Jenkins identify compilation errors? 2. What happens if compilation succeeds but test cases fail? 3. Why is Maven used instead of manually downloading JAR files? 4. How can Jenkins build only a specific GitHub branch? 5. What is the difference between Continuous Integration and Continuous Deployment?

**One-line explanation:** Set up a Jenkins Freestyle job for the e-learning repo that runs Maven `clean package`, archives the artifact and builds on every push.

**Solution:**
1. Manage Jenkins → Plugins: verify `git`, `maven-plugin`, `github`, `copyartifact`, `ws-cleanup`, `email-ext`. Manage Jenkins → Tools: Maven `MAVEN-HOME` → `C:\apache-maven`, JDK `JDK21`.
2. **New Item** → `Elearning_Build` → **Freestyle project** → **OK** → **Source Code Management → Git** → URL `https://github.com/kvrajanirajesh/mid1-e-learningmaven`, branch `*/main` or `*/master`.
3. **Build Triggers** → **GitHub hook trigger for GITScm polling** (or Poll SCM `H/5 * * * *`).
4. **Add build step → Invoke top-level Maven targets** → `MAVEN-HOME` → Goals `clean package` (plus `install`, `test` if separate).
5. **Add post-build action → Archive the artifacts** → `target/*.jar` or `target/*.war`. Save.
6. **Build Now** → **Console Output**: broken code → red FAILED with `[ERROR] COMPILATION ERROR`; fixed code → green with archived artifact.

```powershell
git clone https://github.com/kvrajanirajesh/mid1-e-learningmaven.git
cd mid1-e-learningmaven
mvn -f pom.xml clean package
Get-ChildItem target\*.jar, target\*.war -ErrorAction SilentlyContinue
ngrok http 8080
```

- **How does Jenkins identify compilation errors?** Maven/`javac` logs the error with file/line and exits non-zero; Jenkins shows a red FAILED build in Console Output.
- **If compilation succeeds but tests fail?** `mvn test` exits non-zero → build failed/unstable, no archived artifact and no deployment.
- **Why Maven over manual JAR downloads?** `pom.xml` resolves all dependencies in correct versions and standardises the build lifecycle.
- **Build only a specific branch?** SCM → **Branches to build** = `*/main` (or `*/master`).
- **CI vs CD:** CI builds/tests each commit; CD additionally deploys the validated artifact automatically.

---

### Question 3. Kubernetes – Container Deployment and Auto Scaling [20 MARKS]

**Question:** Scenario: During online examinations, thousands of users access the E-Learning Management System simultaneously. The organization wants to deploy multiple application replicas using Kubernetes and ensure automatic recovery when containers fail. Task: Design and implement the required solution. Expected Output: 1. Start a Kubernetes cluster using Minikube. 2. Create a Deployment with two replicas. 3. Expose the application using a Kubernetes Service. 4. Scale the deployment from two to five replicas. 5. Demonstrate automatic pod recreation after failure. 6. Perform rolling update and rollback. Scenario based Questions: 1. What happens when one pod crashes? 2. How does Kubernetes maintain the desired number of replicas? 3. What is the difference between manual scaling and auto scaling? 4. Why is a Kubernetes Service required? 5. How can you recover from a failed application update?

**One-line explanation:** Deploy the e-learning app on Minikube with 2 replicas, expose it via NodePort, scale to 5, and verify self-healing and rollback.

**Solution:**
1. `minikube start --driver=docker`; `kubectl get nodes` → Ready; `minikube image load elearn-app:latest`.
2. `kubectl create deployment elearn-app --image=elearn-app:latest --replicas=2`; `kubectl get pods`.
3. `kubectl expose deployment elearn-app --type=NodePort --port=8080 --target-port=8080`; open `minikube service elearn-app --url`.
4. `kubectl scale deployment elearn-app --replicas=5`; verify `kubectl get deployments` shows `5/5`.
5. `kubectl delete pod <name>` → new pod created automatically.
6. `kubectl set image deployment/elearn-app elearn-app=elearn-app:v2`, `kubectl rollout status`, then `kubectl rollout undo deployment/elearn-app` for a bad release.

```powershell
minikube start --driver=docker
kubectl get nodes
minikube image load elearn-app:latest
kubectl create deployment elearn-app --image=elearn-app:latest --replicas=2
kubectl expose deployment elearn-app --type=NodePort --port=8080 --target-port=8080
kubectl get deployments,pods,services
minikube service elearn-app --url
kubectl scale deployment elearn-app --replicas=5
kubectl delete pod <pod-name>
kubectl set image deployment/elearn-app elearn-app=elearn-app:v2
kubectl rollout status deployment/elearn-app
kubectl rollout undo deployment/elearn-app
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: elearn-app }
spec:
  replicas: 2
  selector:
    matchLabels: { app: elearn-app }
  template:
    metadata:
      labels: { app: elearn-app }
    spec:
      containers:
        - name: elearn-app
          image: elearn-app:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: elearn-app }
spec:
  type: NodePort
  selector: { app: elearn-app }
  ports: [{ port: 8080, targetPort: 8080, nodePort: 30080 }]
```

- **What happens when one pod crashes?** The ReplicaSet starts a replacement so capacity is restored; the Service avoids the failed pod.
- **How are replicas maintained?** The ReplicaSet controller compares `spec.replicas` with the live pod count and reconciles (self-healing).
- **Manual vs auto scaling:** `kubectl scale` is manual; HPA changes replicas automatically from CPU/metrics.
- **Why a Service?** It gives a stable virtual IP/DNS and load-balances across changing pod IPs.
- **Recover from failed update?** `kubectl rollout undo deployment/elearn-app` restores the previous revision; check with `rollout history`/`status`.

---

### Question 4. Nagios – Infrastructure and Application Monitoring [20 MARKS]

**Question:** Scenario: The organization experiences application downtime, slow response times and database connection failures. The administrator wants to use Nagios to monitor server performance and application availability and generate alerts when services fail. Task: Design and implement the required solution. Expected Output: 1. Pull and run the Nagios Docker image. 2. Configure host and service monitoring. 3. Monitor CPU, memory, disk and server uptime. 4. Configure HTTP and database service checks. 5. Define WARNING and CRITICAL thresholds. 6. Demonstrate alerts and recovery through the Nagios dashboard. Scenario based Questions: 1. How does Nagios identify high CPU utilization? 2. What is the difference between WARNING and CRITICAL? 3. How can Nagios identify application failure when the server is running? 4. What happens if the Nagios monitoring server fails? 5. How would you identify whether downtime is caused by Tomcat, MySQL or networking?

**One-line explanation:** Run Nagios in Docker, read the dashboard, add HTTP/DB checks with thresholds and show alert plus recovery.

**Solution:**
1. `docker pull jasonrivers/nagios:latest`; `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`.
2. `http://localhost:8888` → login `nagiosadmin` / `nagios`.
3. **Hosts** → `localhost` for UP/DOWN + uptime; **Services** for `load`, `disk`, `users`, `HTTP` with thresholds.
4. Add `check_http` on the e-learning URL (`-w 2 -c 5`) and `check_mysql` for the DB via the container config, then `docker restart nagiosdemo`.
5. Stop `elearn-wp` → HTTP CRITICAL + alert; restart → OK. Use **Acknowledge** while investigating.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description ELearn_HTTP
  check_command       check_http!-H localhost -p 8000 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
docker stop elearn-wp
docker start elearn-wp
```

- **How does Nagios identify high CPU utilization?** `check_load` compares load averages with configured warning/critical thresholds and notifications fire when crossed.
- **WARNING vs CRITICAL:** WARNING = threshold crossed but usable; CRITICAL = hard failure needing immediate action.
- **Application failure when server runs:** `check_http`/`check_mysql` fail while the host stays UP, pinpointing the application.
- **If Nagios fails:** Monitoring stops and outages go unnoticed; use a second instance or external monitoring.
- **Tomcat vs MySQL vs networking?** Host DOWN → network; host UP + HTTP CRITICAL + MySQL OK → Tomcat; host UP + MySQL CRITICAL → database; confirm in `docker logs`.

---

### Question 5. AWS – Cloud Deployment using EC2 [20 MARKS]

**Question:** Scenario: The organization wants to migrate its E-Learning Management System from a local server to AWS. Students and faculty should access the application remotely. The deployment must provide secure access, database connectivity and server monitoring. Task: Design and implement the required solution. Expected Output: 1. Launch and configure an EC2 instance. 2. Configure key pairs and security groups. 3. Connect through SSH and install Java, Maven and Tomcat. 4. Clone the GitHub repository and build the application. 5. Deploy the application on Tomcat. 6. Configure database connectivity and CloudWatch monitoring. 7. Access the application using the EC2 public IP address. Scenario based Questions: 1. What could cause an EC2 application to be inaccessible through a browser? 2. Why should SSH access be restricted? 3. What happens if Tomcat stops while EC2 is running? 4. How can application availability be improved? 5. Why would an Application Load Balancer and multiple EC2 instances be better than a single EC2 instance?

**One-line explanation:** Launch free-tier Ubuntu EC2, open ports, SSH in, install Java/Maven/Tomcat, deploy the WAR, set up CloudWatch and open the app from the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `elearn-app`, AMI **Ubuntu Server (Free tier eligible)**, **t2.micro**.
2. Create `.pem`; Security Group inbound **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere**; launch.
3. **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (`icacls` on Windows if needed).
4. Inside: `sudo apt update && sudo apt install -y openjdk-17-jdk maven tomcat9 git`; `git clone https://github.com/kvrajanirajesh/mid1-e-learningmaven.git`; `mvn clean package`; `sudo cp target/*.war /var/lib/tomcat9/webapps/`; `sudo systemctl restart tomcat9`.
5. Configure DB properties (host/user/password) in the app config; if MySQL is containerised, use the internal network name.
6. CloudWatch → **Alarms → Create alarm** on `CPUUtilization` with an SNS notification.
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
git clone https://github.com/kvrajanirajesh/mid1-e-learningmaven.git
cd mid1-e-learningmaven
mvn clean package
sudo cp target/*.war /var/lib/tomcat9/webapps/
sudo systemctl restart tomcat9
sudo apt install -y docker.io
sudo docker build -t elearn-app .
sudo docker run -d -p 8080:8080 --restart unless-stopped elearn-app
curl http://localhost:8080
```

- **Inaccessible via browser:** Security Group missing 8080, Tomcat/container down or not port-mapped, app bound to localhost, OS firewall, wrong context path, or changed public IP.
- **Restrict SSH:** Port 22 is administrative — allow only your IP and key-based login to prevent attacks.
- **Tomcat stops while EC2 runs:** Server stays up but site is down; restart Tomcat/container and check logs.
- **Improve availability:** Restart policies, ALB + multiple instances across AZs, Auto Scaling, health checks, RDS/EBS, CloudWatch alarms.
- **ALB + multiple instances vs single:** Removes the single point of failure, balances traffic and replaces unhealthy instances automatically.
