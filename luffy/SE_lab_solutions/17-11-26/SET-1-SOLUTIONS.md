# 17-11-26 / SET-1 — Solutions

Original question paper: `17-11-26\SET-1.docx`. Lab Internal II, Software Engineering (23CC501PC), dated 17/11/2026.
Repository: `https://github.com/kvrajanirajesh/mid2-ecommercemaven`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: A university wants to deploy a E-Commerce Management System for multiple departments using WordPress as the frontend and MySQL as the backend. The application must remain accessible during container failures, and product, customer and order information must be preserved after restarting containers. Task: Design and implement the required solution. Expected Output: 1. Pull MySQL and WordPress images. 2. Create a custom Docker network and configure both containers. 3. Establish database connectivity. 4. Configure persistent volumes and port mapping. 5. Demonstrate container recovery and data persistence. 6. Access the WordPress application through a browser. Scenario based Questions: 1. What happens if the MySQL container crashes while the E-Commerce Management System is running? 2. How can product, customer and order information be recovered after container deletion? 3. Why is a custom Docker network required? 4. How would you prevent unauthorized access to MySQL? 5. How would you deploy multiple E-Commerce Management System instances using Docker?

**One-line explanation:** Run WordPress + MySQL on one private Docker network with named volumes so products, customers and orders survive container restarts.

**Solution:**
1. Pull images: `docker pull mysql:8.0`, `docker pull wordpress:latest`.
2. Create a user-defined bridge network and volumes: `docker network create ecom-net`, `docker volume create ecom-mysql-data`, `docker volume create ecom-wp-data`.
3. Start MySQL on `ecom-net` with `MYSQL_DATABASE=ecommerce`, `MYSQL_USER=ecomuser`, `MYSQL_PASSWORD=ecompass`, `--restart unless-stopped`; never publish 3306.
4. Start WordPress on the same network with `-p 8000:80` (8080 is Jenkins) and `WORDPRESS_DB_HOST=ecom-mysql:3306` so the container name resolves via Docker DNS.
5. Verify `docker ps` and `docker logs -f ecom-wp`; open `http://localhost:8000` and complete the WordPress install.
6. Recovery/persistence demo: `docker stop ecom-mysql` → DB connection error; `docker start ecom-mysql` → recovers. `docker rm -f ecom-mysql` and re-run with the same volume → products/orders still present.

```powershell
docker pull mysql:8.0
docker pull wordpress:latest
docker network create ecom-net
docker volume create ecom-mysql-data
docker volume create ecom-wp-data
docker run -d --name ecom-mysql --network ecom-net --restart unless-stopped `
  -v ecom-mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=ecommerce `
  -e MYSQL_USER=ecomuser -e MYSQL_PASSWORD=ecompass mysql:8.0
docker run -d --name ecom-wp --network ecom-net --restart unless-stopped `
  -p 8000:80 -v ecom-wp-data:/var/www/html `
  -e WORDPRESS_DB_HOST=ecom-mysql:3306 -e WORDPRESS_DB_NAME=ecommerce `
  -e WORDPRESS_DB_USER=ecomuser -e WORDPRESS_DB_PASSWORD=ecompass wordpress:latest
docker ps
docker logs -f ecom-wp
docker stop ecom-mysql
docker start ecom-mysql
```

- **What happens if the MySQL container crashes while the E-Commerce system is running?** The storefront shows "Error establishing a database connection" and checkout fails; with `--restart unless-stopped` MySQL restarts and WordPress reconnects automatically, with no data loss (data is in the named volume).
- **How can product, customer and order information be recovered after container deletion?** Delete only the container (`docker rm -f ecom-mysql`) and re-run it with `-v ecom-mysql-data:/var/lib/mysql`; the named volume holds the data and is reattached.
- **Why is a custom Docker network required?** It provides built-in DNS so WordPress reaches MySQL by the name `ecom-mysql`, isolates the two containers from unrelated ones, and survives container restarts better than the default bridge.
- **How would you prevent unauthorized access to MySQL?** Do not publish 3306; keep it on the internal network only, use a dedicated least-privilege user with a strong password, store credentials in secrets, and restrict access to the WordPress container.
- **How would you deploy multiple E-Commerce instances using Docker?** Run several WordPress containers (`ecom-wp-1..5`) each with its own port (8001–8005), DB/volume and container name, ideally behind an Nginx reverse proxy on port 80.

---

### Question 2. Jenkins – Java Application Build and Continuous Integration [20 MARKS]

**Question:** Scenario: A software company has five developers working on a Java-based E-Commerce Management System. Developers frequently update the GitHub repository. The company wants Jenkins and Maven to automate compilation, testing and artifact generation. Task: Design and implement the required solution. Expected Output: 1. Configure JDK, Maven and Jenkins plugins. 2. Integrate the GitHub repository with Jenkins. 3. Configure build triggers and Maven goals. 4. Execute compilation and test cases. 5. Archive the generated WAR/JAR artifact. 6. Demonstrate successful and failed builds. Scenario based Questions: 1. How does Jenkins identify compilation errors? 2. What happens if compilation succeeds but test cases fail? 3. Why is Maven used instead of manually downloading JAR files? 4. How can Jenkins build only a specific GitHub branch? 5. What is the difference between Continuous Integration and Continuous Deployment?

**One-line explanation:** Create a Jenkins Freestyle job for the e-commerce Maven repo that runs `clean package`, archives the artifact and builds on every push.

**Solution:**
1. Manage Jenkins → Plugins: install/verify `git`, `maven-plugin`, `github`, `copyartifact`, `ws-cleanup`, `email-ext`. Manage Jenkins → Tools: Maven `MAVEN-HOME` → `C:\apache-maven`, JDK `JDK21`.
2. **New Item** → `Ecom_Build` → **Freestyle project** → **OK** → **Source Code Management → Git** → URL `https://github.com/kvrajanirajesh/mid2-ecommercemaven`, branch `*/main` (or `*/master`).
3. **Build Triggers** → tick **GitHub hook trigger for GITScm polling** (or Poll SCM `H/5 * * * *`).
4. **Add build step → Invoke top-level Maven targets** → Maven `MAVEN-HOME` → Goals `clean package` (add `install`, `test` as extra steps if needed).
5. **Add post-build action → Archive the artifacts** → `target/*.jar` or `target/*.war`. Save.
6. **Build Now** → **Console Output**: a broken Java file gives a red FAILED build with `[ERROR] COMPILATION ERROR`; fixing it gives a green build and an archived artifact.

```powershell
git clone https://github.com/kvrajanirajesh/mid2-ecommercemaven.git
cd mid2-ecommercemaven
mvn -f pom.xml clean package
Get-ChildItem target\*.jar, target\*.war -ErrorAction SilentlyContinue
ngrok http 8080
```

- **How does Jenkins identify compilation errors?** Maven/`javac` writes the failing file and line to Console Output and exits with a non-zero code; Jenkins marks the build FAILED (red ball).
- **What happens if compilation succeeds but test cases fail?** `mvn test` returns non-zero, so the build is failed/unstable, the artifact is not archived and downstream deployment is blocked.
- **Why Maven instead of manual JAR downloads?** `pom.xml` declares dependencies that Maven fetches in correct versions automatically, and the standard lifecycle makes builds repeatable across machines.
- **How to build only a specific branch?** SCM → **Branches to build** = `*/main` (or `*/master`); Jenkins clones only that ref.
- **CI vs CD:** CI automatically builds/tests each commit; CD also automatically deploys the validated artifact to a server.

---

### Question 3. Kubernetes – Container Deployment and Auto Scaling [20 MARKS]

**Question:** Scenario: During a festival sale, thousands of users access the E-Commerce Management System simultaneously. The organization wants to deploy multiple application replicas using Kubernetes and ensure automatic recovery when containers fail. Task: Design and implement the required solution. Expected Output: 1. Start a Kubernetes cluster using Minikube. 2. Create a Deployment with two replicas. 3. Expose the application using a Kubernetes Service. 4. Scale the deployment from two to five replicas. 5. Demonstrate automatic pod recreation after failure. 6. Perform rolling update and rollback. Scenario based Questions: 1. What happens when one pod crashes? 2. How does Kubernetes maintain the desired number of replicas? 3. What is the difference between manual scaling and auto scaling? 4. Why is a Kubernetes Service required? 5. How can you recover from a failed application update?

**One-line explanation:** Use Minikube to run a 2-replica e-commerce Deployment, expose it with NodePort, scale to 5, prove self-healing and roll back a bad update.

**Solution:**
1. `minikube start --driver=docker`; `kubectl get nodes` → Ready. Load the app image with `minikube image load ecom-app:latest` (or use a Docker Hub image).
2. `kubectl create deployment ecom-app --image=ecom-app:latest --replicas=2`; `kubectl get pods`.
3. `kubectl expose deployment ecom-app --type=NodePort --port=8080 --target-port=8080`; open `minikube service ecom-app --url`.
4. `kubectl scale deployment ecom-app --replicas=5`; verify `kubectl get deployments` (`5/5`).
5. `kubectl delete pod <name>` → replacement pod is created automatically.
6. `kubectl set image deployment/ecom-app ecom-app=ecom-app:v2` → rolling update; `kubectl rollout status`; `kubectl rollout undo deployment/ecom-app` on failure.

```powershell
minikube start --driver=docker
kubectl get nodes
minikube image load ecom-app:latest
kubectl create deployment ecom-app --image=ecom-app:latest --replicas=2
kubectl expose deployment ecom-app --type=NodePort --port=8080 --target-port=8080
kubectl get deployments,pods,services
minikube service ecom-app --url
kubectl scale deployment ecom-app --replicas=5
kubectl delete pod <pod-name>
kubectl set image deployment/ecom-app ecom-app=ecom-app:v2
kubectl rollout status deployment/ecom-app
kubectl rollout undo deployment/ecom-app
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: ecom-app }
spec:
  replicas: 2
  selector:
    matchLabels: { app: ecom-app }
  template:
    metadata:
      labels: { app: ecom-app }
    spec:
      containers:
        - name: ecom-app
          image: ecom-app:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: ecom-app }
spec:
  type: NodePort
  selector: { app: ecom-app }
  ports: [{ port: 8080, targetPort: 8080, nodePort: 30080 }]
```

- **What happens when one pod crashes?** The ReplicaSet detects fewer than 5 ready pods and starts a replacement; the Service routes traffic only to ready pods.
- **How does Kubernetes maintain the desired replicas?** The ReplicaSet controller continuously compares `spec.replicas` with live pods and creates/deletes pods to match (self-healing).
- **Manual vs auto scaling:** `kubectl scale` sets the count yourself; a Horizontal Pod Autoscaler (`kubectl autoscale`) changes it automatically based on CPU/custom metrics.
- **Why a Service is required?** Pod IPs change when pods restart; a Service gives a stable IP/DNS and load-balances traffic across all matching pods.
- **Recover from a failed update?** `kubectl rollout undo deployment/ecom-app` restores the previous ReplicaSet; use `rollout history`/`rollout status` to inspect and confirm.

---

### Question 4. Nagios – Infrastructure and Application Monitoring [20 MARKS]

**Question:** Scenario: The organization experiences application downtime, slow response times and database connection failures. The administrator wants to use Nagios to monitor server performance and application availability and generate alerts when services fail. Task: Design and implement the required solution. Expected Output: 1. Pull and run the Nagios Docker image. 2. Configure host and service monitoring. 3. Monitor CPU, memory, disk and server uptime. 4. Configure HTTP and database service checks. 5. Define WARNING and CRITICAL thresholds. 6. Demonstrate alerts and recovery through the Nagios dashboard. Scenario based Questions: 1. How does Nagios identify high CPU utilization? 2. What is the difference between WARNING and CRITICAL? 3. How can Nagios identify application failure when the server is running? 4. What happens if the Nagios monitoring server fails? 5. How would you identify whether downtime is caused by Tomcat, MySQL or networking?

**One-line explanation:** Run Nagios in Docker, view host/service status in its dashboard, add HTTP/MySQL checks with thresholds and demonstrate an alert and recovery.

**Solution:**
1. `docker pull jasonrivers/nagios:latest`; `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`.
2. Browser → `http://localhost:8888` → login `nagiosadmin` / `nagios`.
3. **Hosts** → `localhost` for UP/DOWN and uptime; **Services** for `load`, `disk`, `users`, `HTTP` with state and "Last Check".
4. Add checks with thresholds via the container config: `check_http` on the e-commerce URL (`-w 2 -c 5`) and `check_mysql` for the DB; restart Nagios.
5. Stop `ecom-wp` → HTTP goes **CRITICAL** with an alert; restart → **OK** recovery. Use **Acknowledge** in the service page while fixing.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description ECOM_HTTP
  check_command       check_http!-H localhost -p 8000 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
docker stop ecom-wp
docker start ecom-wp
```

- **How does Nagios identify high CPU utilization?** `check_load` samples the 1/5/15-minute load and compares it with configured warning/critical thresholds, changing state and sending notifications when exceeded.
- **WARNING vs CRITICAL:** WARNING = threshold crossed but service still usable (disk 80%, slow response); CRITICAL = failed/hard limit (connection refused, disk 95%) needing immediate action.
- **Application failure while server runs:** `check_http`/`check_mysql` test the app endpoint or DB port; the host stays UP while the service turns CRITICAL.
- **If the Nagios server fails:** Monitoring stops, so outages go unnoticed; mitigate with a second Nagios instance or an external monitoring service watching it.
- **Tomcat vs MySQL vs networking?** Host DOWN → network/server; host UP + HTTP CRITICAL + MySQL OK → Tomcat; host UP + MySQL CRITICAL → database; confirm with `docker logs`.

---

### Question 5. AWS – Cloud Deployment using EC2 [20 MARKS]

**Question:** Scenario: The organization wants to migrate its E-Commerce Management System from a local server to AWS. Customers should access the application remotely. The deployment must provide secure access, database connectivity and server monitoring. Task: Design and implement the required solution. Expected Output: 1. Launch and configure an EC2 instance. 2. Configure key pairs and security groups. 3. Connect through SSH and install Java, Maven and Tomcat. 4. Clone the GitHub repository and build the application. 5. Deploy the application on Tomcat. 6. Configure database connectivity and CloudWatch monitoring. 7. Access the application using the EC2 public IP address. Scenario based Questions: 1. What could cause an EC2 application to be inaccessible through a browser? 2. Why should SSH access be restricted? 3. What happens if Tomcat stops while EC2 is running? 4. How can application availability be improved? 5. Why would an Application Load Balancer and multiple EC2 instances be better than a single EC2 instance?

**One-line explanation:** Launch free-tier Ubuntu EC2, open SSH/HTTP/8080, SSH in, install Java/Maven/Tomcat, build and deploy the WAR, set up CloudWatch and verify via public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `ecom-app`, AMI **Ubuntu Server (Free tier eligible)**, 64-bit, **t2.micro**.
2. Create/download `.pem`; Security Group inbound **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere**; launch.
3. **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (fix permissions with `icacls` on Windows).
4. Inside: `sudo apt update && sudo apt install -y openjdk-17-jdk maven tomcat9 git`; clone the repo; `mvn clean package` in the project folder; copy the WAR into `/var/lib/tomcat9/webapps/`; `sudo systemctl restart tomcat9`.
5. Configure DB connectivity in the app's properties with the RDS/DB host, user and password; if MySQL runs in Docker on the same host, use a private network and the container name.
6. CloudWatch: Console → **CloudWatch → Alarms → Create alarm** on `EC2 → CPUUtilization` (e.g. >80% for 5 min) with an SNS notification.
7. Browser → `http://<public-ip>:8080/<context>`.

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
git clone https://github.com/kvrajanirajesh/mid2-ecommercemaven.git
cd mid2-ecommercemaven
mvn clean package
sudo cp target/*.war /var/lib/tomcat9/webapps/
sudo systemctl restart tomcat9
# Docker alternative
sudo apt install -y docker.io
sudo docker build -t ecom-app .
sudo docker run -d -p 8080:8080 --restart unless-stopped ecom-app
curl http://localhost:8080
```

- **Why is the app inaccessible via browser?** Security Group missing inbound 8080, Tomcat/container down or not port-mapped, app bound to 127.0.0.1, OS firewall, wrong context path, or a changed public IP.
- **Why restrict SSH?** Port 22 is an administrative entry point; expose it only to your IP to prevent brute-force attempts, and use key-based auth.
- **If Tomcat stops while EC2 runs:** The instance stays up but the site returns connection refused; restart Tomcat (`sudo systemctl restart tomcat9` or `docker restart`) and check logs/CloudWatch.
- **How to improve availability?** `--restart unless-stopped` for containers, ALB + multiple EC2 instances across AZs, health checks, Auto Scaling, RDS/EBS for data and CloudWatch alarms.
- **ALB + multiple instances vs one:** A single instance is a single point of failure; an ALB spreads traffic, removes unhealthy instances and works with Auto Scaling for high availability and fault tolerance.
