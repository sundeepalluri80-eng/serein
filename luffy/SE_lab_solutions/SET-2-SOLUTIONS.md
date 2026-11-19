# SET-2 — Solutions

Original question paper: `SET-2.docx` (top level). Lab Internal II, Software Engineering (23CC501PC).
Repository: `https://github.com/deepthisagar7/metro-reservation.git`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: A metro railway authority wants to deploy its Metro Reservation web application using Docker. The application should run in one container, while the MySQL database should run in a separate container (if database connectivity is configured in the application). Task: Create and run multiple Docker containers for the Metro Reservation web application and MySQL database. Expected Output: Pull the required Tomcat and MySQL Docker images. Build the Metro Reservation WAR file and configure the application container. Create and configure the application and database containers. Establish communication between the containers. Access the Metro Reservation application through a web browser. Scenario-Based Questions: What happens if the MySQL container stops while passengers are accessing the Metro Reservation application? How can you preserve passenger reservation data even after deleting the MySQL container? Why should the Metro Reservation application and MySQL database run in separate containers? How can you prevent unauthorized access to passenger and reservation information? How would you troubleshoot the application if it cannot connect to the MySQL database?

**One-line explanation:** Build the WAR, then run Tomcat and MySQL as two containers on one Docker network so the metro site can reach the DB by container name.

**Solution:**
1. Pull images: `docker pull tomcat:9.0` and `docker pull mysql:8.0`.
2. Clone and build the WAR on the host: `git clone https://github.com/deepthisagar7/metro-reservation.git`, then `mvn -f metro-reservation/pom.xml clean package` → `target/metro-reservation.war`.
3. Create a network and a named volume: `docker network create metro-net`, `docker volume create metro-mysql-data`.
4. Run MySQL on `metro-net` with `MYSQL_DATABASE=metro`, `MYSQL_USER=metrouser`, `MYSQL_PASSWORD=metropass`, `MYSQL_ROOT_PASSWORD=root`; keep 3306 internal.
5. Run Tomcat with `-p 8085:8080` (8080 is Jenkins) on `metro-net` and copy the WAR into `/usr/local/tomcat/webapps/`. Tomcat 9 uses `javax.servlet`, which matches this WAR.
6. Verify `docker ps`, `docker logs -f metro-tomcat`, and open `http://localhost:8085/metro-reservation`.

```powershell
docker pull tomcat:9.0
docker pull mysql:8.0
git clone https://github.com/deepthisagar7/metro-reservation.git
mvn -f metro-reservation\pom.xml clean package
docker network create metro-net
docker volume create metro-mysql-data

docker run -d --name metro-mysql --network metro-net --restart unless-stopped `
  -v metro-mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=metro `
  -e MYSQL_USER=metrouser -e MYSQL_PASSWORD=metropass mysql:8.0

docker run -d --name metro-tomcat --network metro-net --restart unless-stopped -p 8085:8080 tomcat:9.0
docker cp metro-reservation\target\metro-reservation.war metro-tomcat:/usr/local/tomcat/webapps/
docker restart metro-tomcat
docker ps
docker logs -f metro-tomcat
```

- **What happens if the MySQL container stops while passengers are accessing the application?** Bookings fail with a database error (500). With `--restart unless-stopped` MySQL comes back automatically and Tomcat reconnects; reservations remain safe in the `metro-mysql-data` volume.
- **How can you preserve passenger reservation data even after deleting the MySQL container?** Store data in a named volume (`-v metro-mysql-data:/var/lib/mysql`). `docker rm -f metro-mysql` deletes only the container; re-running with the same `-v` restores every table.
- **Why should the Metro Reservation application and MySQL database run in separate containers?** Each tier can be scaled, updated and restarted independently; isolation limits failures and attack surface, and images/versions can be chosen separately.
- **How can you prevent unauthorized access to passenger and reservation information?** Do not publish 3306, use a non-root DB user with a strong password, keep the DB on a private Docker network, and use Docker secrets/env files plus TLS in production.
- **How would you troubleshoot the application if it cannot connect to the MySQL database?** Run `docker logs metro-mysql metro-tomcat`, check container names/network with `docker inspect`, verify `WORDPRESS/DB_HOST=metro-mysql`, and test from Tomcat: `docker exec -it metro-tomcat sh -c 'nc -zv metro-mysql 3306'` and credentials with `mysql -h metro-mysql -u metrouser -p`.

---

### Question 2. Jenkins – Freestyle Project [20 MARKS]

**Question:** Scenario: The development team frequently updates the Metro Reservation application on GitHub. The team wants Jenkins to automatically build the latest application version and generate a WAR file using Maven. Task: Create and configure a Jenkins Freestyle Project to build the Metro Reservation Java web application using Maven. Expected Output: Configure the Metro Reservation GitHub repository. Configure JDK and Maven build settings. Execute the Maven build successfully. Generate the metro-reservation.war file. Verify the build status and console output in Jenkins. Scenario-Based Questions: What happens if a developer introduces compilation errors into the Metro Reservation application? How can Jenkins detect changes made to the GitHub repository? What is the difference between successfully generating the WAR file and deploying the application on Tomcat? Why is Maven useful for managing dependencies and building the Metro Reservation application? How would you troubleshoot a Jenkins build failure caused by an incorrect pom.xml file?

**One-line explanation:** Create a Freestyle job that checks out the metro-reservation repo and runs Maven `clean package`, then archives the generated WAR.

**Solution:**
1. Manage Jenkins → Plugins → ensure `git`, `maven-plugin`, `github`, `ws-cleanup` are installed. Manage Jenkins → Tools → Maven name `MAVEN-HOME` → `C:\apache-maven`, JDK `JDK21`.
2. Dashboard → **New Item** → name `MetroWeb_Build` → **Freestyle project** → **OK**.
3. **Source Code Management** → **Git** → URL `https://github.com/deepthisagar7/metro-reservation.git`, branch `*/master` (check the repo default branch).
4. **Build Triggers** → tick **GitHub hook trigger for GITScm polling**.
5. **Add build step** → **Invoke top-level Maven targets** → Maven `MAVEN-HOME` → Goals `clean package` (or separate `clean`, then `install`).
6. **Add post-build action** → **Archive the artifacts** → `target/*.war`. Save → **Build Now** → **Console Output** → confirm `BUILD SUCCESS` and the archived `metro-reservation.war`.

```powershell
git clone https://github.com/deepthisagar7/metro-reservation.git
cd metro-reservation
mvn -f pom.xml clean package
Get-ChildItem target\*.war
ngrok http 8080     # public URL for the GitHub webhook
```

- **What happens if a developer introduces compilation errors?** `javac` via Maven returns a non-zero exit code, Jenkins shows a red FAILED build and `[ERROR] COMPILATION ERROR` in Console Output; no WAR is archived and downstream deployment does not run.
- **How can Jenkins detect changes made to the GitHub repository?** Either a **GitHub webhook** (Settings → Webhooks → `http://<jenkins-url>/github-webhook/`, "Just the push event") with **GitHub hook trigger for GITScm polling** ticked, or **Poll SCM** with a cron like `H/5 * * * *`.
- **What is the difference between generating the WAR and deploying on Tomcat?** `clean package` only produces `target/metro-reservation.war` in Jenkins; deployment means copying that WAR into Tomcat's `webapps` folder (or using **Deploy WAR/EAR to a container** with Tomcat URL `http://localhost:8085` and credentials `admin`/`1234`) so the app actually runs.
- **Why is Maven useful?** It reads `pom.xml` and downloads all dependencies in the right versions automatically, and standardises the lifecycle (`clean → compile → test → package`) so the WAR is reproducible for every developer and for Jenkins.
- **How would you troubleshoot a build failure caused by an incorrect pom.xml?** Open Console Output for the first `[ERROR]` (e.g. missing dependency, wrong plugin version, malformed XML), validate the file with `mvn validate -f pom.xml`, fix coordinates/version/tags, re-run locally, then rebuild in Jenkins.

---

### Question 3. System Monitoring using Nagios [20 MARKS]

**Question:** Scenario: The Metro Reservation application experiences slow responses and occasional downtime during peak travel hours. The administrator wants to monitor the application server and its services using Nagios. Task: Set up Nagios Monitoring to monitor the performance and availability of the Metro Reservation application server. Expected Output: Pull and run the Nagios Docker image. Configure monitoring for the Metro Reservation application server. Monitor CPU usage, memory, disk space and server uptime. Configure HTTP service monitoring to check application availability. Display monitoring status and alerts through the Nagios dashboard. Scenario-Based Questions: How can Nagios identify whether the Metro Reservation application server is running or down? What is the difference between WARNING and CRITICAL alerts? What happens if CPU usage reaches 100% during peak ticket reservations? How can Nagios identify whether the Metro Reservation application is unavailable even when the server is running? How can an administrator identify whether downtime is caused by Tomcat, MySQL or the server?

**One-line explanation:** Run Nagios in Docker, log into its dashboard, verify host uptime and system services, add HTTP/MySQL checks with WARNING/CRITICAL thresholds, and observe alerts.

**Solution:**
1. `docker pull jasonrivers/nagios:latest` then `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`.
2. Browser → `http://localhost:8888` → login `nagiosadmin` / `nagios`.
3. Click **Hosts** → `localhost` for host state and uptime; click **Services** for `load`, `disk`, `users`, `HTTP`, `PING` with current status and "Last Check".
4. Add HTTP check on the app URL and MySQL check with thresholds via the container config (`check_http -H localhost -p 8085 -w 2 -c 5`, `check_mysql` with credentials).
5. Demo: stop `metro-tomcat` → HTTP service becomes **CRITICAL** and a notification appears; restart it → **OK** recovery. Use **Acknowledge** in the service detail page while investigating.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description Metro_HTTP
  check_command       check_http!-H localhost -p 8085 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
# alert/recovery demo
docker stop metro-tomcat
docker start metro-tomcat
```

- **How can Nagios identify whether the application server is running or down?** The host check (`check_ping`/`check_host`) marks the host **UP** or **DOWN**; the dashboard's Hosts page shows state and uptime, while `check_http`/`check_tcp` verify the Tomcat port.
- **What is the difference between WARNING and CRITICAL alerts?** WARNING = threshold crossed but still usable (e.g. response 2–5 s, disk 80%); CRITICAL = hard failure (connection refused, >5 s, disk 95%) requiring immediate action.
- **What happens if CPU usage reaches 100% during peak ticket reservations?** `check_load` returns **CRITICAL**, an alert/notification is sent, reservations slow down or time out; the administrator scales the app (more replicas) or investigates the runaway process.
- **How can Nagios identify application failure even when the server is running?** By service checks against the application itself — `check_http` on `/metro-reservation` or `check_mysql` — the host stays UP but the service goes CRITICAL when the app/DB is down.
- **How can an administrator identify whether downtime is caused by Tomcat, MySQL or the server?** Compare service states: host DOWN → server/network; host UP + HTTP CRITICAL + MySQL OK → Tomcat; host UP + MySQL CRITICAL → database. Confirm with `docker logs metro-tomcat` / `metro-mysql`.

---

### Question 4. Container Scaling using Kubernetes [20 MARKS]

**Question:** Scenario: During festival seasons and holidays, thousands of passengers access the Metro Reservation application simultaneously. The railway authority wants to increase the number of application instances to handle the additional requests. Task: Demonstrate the scaling of the Metro Reservation application using Kubernetes. Expected Output: Create a Docker image for the Metro Reservation application. Create a Kubernetes Deployment for the application. Check the running application pods. Scale the application deployment by increasing the number of replicas. Verify the scaling using Kubernetes commands. Demonstrate automatic pod recreation after deleting a pod. Scenario-Based Questions: What happens if one Metro Reservation application pod crashes during ticket booking? How does Kubernetes maintain the desired number of application replicas? What is the difference between scaling up and scaling down? How can increasing the number of application replicas help during peak reservation hours? How can Kubernetes distribute incoming requests among multiple application pods?

**One-line explanation:** Build the app image, deploy it to Minikube, scale the Deployment to more replicas, and verify self-healing and load distribution.

**Solution:**
1. Build and tag the image locally: `docker build -t metro-reservation:latest .` in the repo folder (WAR on Tomcat 9 base).
2. Start Minikube: `minikube start --driver=docker`; make the image available inside the cluster with `minikube image load metro-reservation:latest` (or push to Docker Hub).
3. Create the Deployment: `kubectl create deployment metro-app --image=metro-reservation:latest --replicas=2`; check `kubectl get pods`.
4. Expose it: `kubectl expose deployment metro-app --type=NodePort --port=8080 --target-port=8080`; open `minikube service metro-app --url`.
5. Scale up: `kubectl scale deployment metro-app --replicas=5`; verify with `kubectl get deployments` (`5/5`) and `kubectl get pods`.
6. Delete one pod: `kubectl delete pod <name>` → ReplicaSet creates a new one automatically.

```powershell
cd metro-reservation
docker build -t metro-reservation:latest .
minikube start --driver=docker
minikube image load metro-reservation:latest
kubectl create deployment metro-app --image=metro-reservation:latest --replicas=2
kubectl expose deployment metro-app --type=NodePort --port=8080 --target-port=8080
kubectl get deployments,pods,services
minikube service metro-app --url
kubectl scale deployment metro-app --replicas=5
kubectl get pods -o wide
kubectl delete pod <pod-name>
kubectl get pods -o wide
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metro-app
spec:
  replicas: 2
  selector:
    matchLabels: { app: metro-app }
  template:
    metadata:
      labels: { app: metro-app }
    spec:
      containers:
        - name: metro-app
          image: metro-reservation:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: metro-app
spec:
  type: NodePort
  selector: { app: metro-app }
  ports:
    - port: 8080
      targetPort: 8080
```

- **What happens if one pod crashes during ticket booking?** Its requests fail briefly, but the ReplicaSet detects the missing pod and starts a replacement; the Service stops sending traffic to the dead pod once it fails readiness.
- **How does Kubernetes maintain the desired number of replicas?** The Deployment's ReplicaSet continuously compares `spec.replicas` with the number of healthy pods and creates or deletes pods to match (self-healing).
- **What is the difference between scaling up and scaling down?** Scaling up increases `replicas` (e.g. 2 → 5) for more capacity; scaling down reduces it to save resources. Both are done with `kubectl scale --replicas=N`.
- **How can more replicas help during peak reservation hours?** Each pod handles part of the load, so response time stays low and a single pod failure affects only a fraction of users instead of all bookings.
- **How does Kubernetes distribute incoming requests among multiple pods?** The Service has a stable ClusterIP/NodePort and uses kube-proxy/iptables to load-balance connections across all ready pods matched by its selector.

---

### Question 5. AWS Deployment using EC2 [20 MARKS]

**Question:** Scenario: The Metro Railway authority wants passengers to access the Metro Reservation application from different locations. The authority decides to deploy the Java web application on an AWS EC2 instance. Task: Demonstrate the deployment of the Metro Reservation web application in AWS using an EC2 instance. Expected Output: Launch and configure an AWS EC2 instance. Access the Metro Reservation application through the EC2 public IP address. Scenario-Based Questions: What could be the reasons if passengers cannot access the Metro Reservation application using the EC2 public IP address? Why should HTTP and application server ports be configured in the EC2 security group? What happens if Tomcat stops while the EC2 instance is still running? How can you make the Metro Reservation application available if the EC2 instance fails? What are the advantages of deploying the Metro Reservation application on AWS instead of a local server?

**One-line explanation:** Launch a free-tier Ubuntu EC2 box, open SSH/HTTP/Tomcat ports, SSH in, install Docker + Git, run the WAR on Tomcat and open it from the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**; name `metro-app`; AMI **Ubuntu Server (Free tier eligible)**; type **t2.micro**.
2. **Create new key pair** (`.pem`), store in `C:\Users\abhir\aws`. Security Group inbound: **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere**. Launch.
3. Select instance → **Connect** → **SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (`icacls` first on Windows if needed).
4. Inside: `sudo apt update && sudo apt-get install -y docker.io git`, clone the repo, build the WAR locally with Maven (or use multi-stage Dockerfile), then run Tomcat in Docker with the WAR mounted on port 8080.
5. Open `http://<public-ip>:8080/metro-reservation` in a browser.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/deepthisagar7/metro-reservation.git
cd metro-reservation
cat > Dockerfile <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -t metro-app .
sudo docker run -d --name metro -p 8080:8080 --restart unless-stopped metro-app
curl http://localhost:8080/
```

- **What could be the reasons passengers cannot access the app using the public IP?** Security Group missing inbound 8080, Tomcat not running or container not port-mapped, app bound to 127.0.0.1 instead of 0.0.0.0, OS firewall blocking, wrong context path, or a changed public IP after stop/start.
- **Why should HTTP and application server ports be configured in the EC2 security group?** A Security Group is a stateful virtual firewall; inbound traffic is dropped by default, so 80/8080 must be explicitly allowed for browsers to reach the app while all other ports stay closed.
- **What happens if Tomcat stops while the EC2 instance is still running?** The server remains reachable but the site shows connection refused/502; restart Tomcat (`sudo docker restart metro` or `sudo systemctl restart tomcat`) and check logs/CloudWatch.
- **How can you make the application available if the EC2 instance fails?** Use an Application Load Balancer with Auto Scaling and multiple EC2 instances across Availability Zones, keep data on EBS/RDS, and use an Elastic IP or Route 53 for a stable address.
- **What are the advantages of deploying on AWS instead of a local server?** On-demand scaling, no upfront hardware, high availability across AZs, pay-as-you-go cost, managed backups/monitoring (CloudWatch), and global access through public IPs/DNS.
