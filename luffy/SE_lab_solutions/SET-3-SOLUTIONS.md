# SET-3 — Solutions

Original question paper: `SET-3.docx` (top level). Lab Internal II, Software Engineering (23CC501PC).
Repository: `https://github.com/deepthisagar7/AI-OLMS.git`

---

### Question 1. Docker – Multi-Container Application [20 MARKS]

**Question:** Scenario: A college wants to deploy its AI Online Learning Management System (AI-OLMS) using Docker. Students and instructors should be able to access the learning portal, while course and enrolment information is managed through a database. Task: Create and run multiple Docker containers for the AI-OLMS Java web application and MySQL database. Expected Output: Pull the required Tomcat and MySQL Docker images. Build the AI-OLMS Maven project and generate the WAR file. Create and configure application and database containers. Establish communication between the containers. Access the AI-OLMS application through a web browser. Scenario-Based Questions: What happens if the MySQL container stops while students are accessing the AI-OLMS portal? How can you preserve student and course information even after deleting the MySQL container? Why should the AI-OLMS application and MySQL database run in separate containers? How can you protect student login credentials and course information from unauthorized access? How would you troubleshoot the application if it cannot connect to the MySQL database?

**One-line explanation:** Build the AI-OLMS WAR and run it on Tomcat in one container with MySQL in another, connected by a private Docker network and a persistent volume.

**Solution:**
1. Pull `tomcat:9.0` and `mysql:8.0`.
2. Clone `https://github.com/deepthisagar7/AI-OLMS.git` and build with Maven: `mvn -f AI-OLMS/pom.xml clean package` → `target/*.war`.
3. Create `docker network create olms-net` and `docker volume create olms-mysql-data`.
4. Run MySQL on `olms-net` (no host port 3306) with DB `olms`, user `olmsuser`, password `olmspass`.
5. Run Tomcat with `-p 8085:8080` on `olms-net`, copy the WAR into `webapps/`, and restart it.
6. Verify `docker ps`, `docker logs -f olms-tomcat`, and open `http://localhost:8085/<war-name>` in the browser.

```powershell
docker pull tomcat:9.0
docker pull mysql:8.0
git clone https://github.com/deepthisagar7/AI-OLMS.git
mvn -f AI-OLMS\pom.xml clean package
docker network create olms-net
docker volume create olms-mysql-data

docker run -d --name olms-mysql --network olms-net --restart unless-stopped `
  -v olms-mysql-data:/var/lib/mysql `
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=olms `
  -e MYSQL_USER=olmsuser -e MYSQL_PASSWORD=olmspass mysql:8.0

docker run -d --name olms-tomcat --network olms-net --restart unless-stopped -p 8085:8080 tomcat:9.0
docker cp AI-OLMS\target\*.war olms-tomcat:/usr/local/tomcat/webapps/
docker restart olms-tomcat
docker ps
docker logs -f olms-tomcat
```

- **What happens if the MySQL container stops while students are accessing the portal?** Login and course queries fail (HTTP 500); with `--restart unless-stopped` MySQL restarts automatically and the portal recovers. Student/course data is untouched because it is in the named volume.
- **How can you preserve student and course information even after deleting the MySQL container?** Keep the data in `-v olms-mysql-data:/var/lib/mysql`; deleting the container does not delete volumes, so re-running with the same volume restores all records.
- **Why should the application and MySQL database run in separate containers?** Independent scaling and updates, better isolation/failure containment, and the ability to use tuned images for each tier.
- **How can you protect student login credentials and course information?** Never publish 3306, use a least-privilege DB user, keep the DB on the private network, hash passwords in the app, store secrets outside source code, and enable HTTPS.
- **How would you troubleshoot the application if it cannot connect to the MySQL database?** Check `docker logs olms-mysql olms-tomcat`, confirm both are on `olms-net`, verify the JDBC URL uses `olms-mysql:3306`, and test connectivity with `docker exec -it olms-tomcat sh -c 'nc -zv olms-mysql 3306'`.

---

### Question 2. Jenkins – Freestyle Project [20 MARKS]

**Question:** Scenario: The AI-OLMS development team frequently updates the application on GitHub. The college wants to automate the build process using Jenkins and Maven to identify errors and generate the application WAR file. Task: Create and configure a Jenkins Freestyle Project to build the AI-OLMS Java web application using Maven. Expected Output: Configure the AI-OLMS GitHub repository. Configure JDK and Maven build settings. Execute the Maven build successfully. Generate the AI-OLMS WAR file. Verify the build status and console output in Jenkins. Scenario-Based Questions: What happens if a developer introduces compilation errors in the AI-OLMS application? How can Jenkins automatically detect changes in the GitHub repository? What is the difference between successfully generating the WAR file and deploying the application on Tomcat? Why is Maven useful for managing dependencies in the AI-OLMS project? How would you identify and resolve a failed Jenkins build?

**One-line explanation:** Configure a Freestyle job that checks out AI-OLMS, runs Maven `clean package`, archives the WAR and is triggered by a GitHub webhook.

**Solution:**
1. Manage Jenkins → Plugins: confirm `git`, `maven-plugin`, `github`, `email-ext`. Manage Jenkins → Tools: Maven `MAVEN-HOME` (`C:\apache-maven`), JDK `JDK21`.
2. **New Item** → name `AIOLMS_Build` → **Freestyle project** → **OK**.
3. **Source Code Management** → **Git** → URL `https://github.com/deepthisagar7/AI-OLMS.git`, branch `*/main` or `*/master`.
4. **Build Triggers** → tick **GitHub hook trigger for GITScm polling** (GitHub: Settings → Webhooks → Add webhook → Payload URL `https://<ngrok>/github-webhook/`, JSON, push only).
5. **Add build step** → **Invoke top-level Maven targets** → `MAVEN-HOME` → Goals `clean package`.
6. **Add post-build action** → **Archive the artifacts** → `target/*.war`. Save → **Build Now** → **Console Output** → check `BUILD SUCCESS` and the WAR.

```powershell
git clone https://github.com/deepthisagar7/AI-OLMS.git
cd AI-OLMS
mvn -f pom.xml clean package
Get-ChildItem target\*.war
ngrok http 8080
```

- **What happens if a developer introduces compilation errors?** `javac` fails, Maven exits non-zero, Jenkins marks the build FAILED (red) with `[ERROR] COMPILATION ERROR` in Console Output and does not archive the WAR.
- **How can Jenkins automatically detect changes in the GitHub repository?** Via a GitHub webhook hitting `/github-webhook/` with the **GitHub hook trigger for GITScm polling** option enabled, or by **Poll SCM** (`H/5 * * * *`) as a fallback.
- **What is the difference between generating the WAR and deploying on Tomcat?** Generating is only building `target/*.war` in the workspace; deploying means putting that WAR in Tomcat's `webapps` (or Jenkins **Deploy WAR/EAR to a container**, Tomcat URL `http://localhost:8085`, user `admin`/`1234`) so it runs.
- **Why is Maven useful for dependency management?** It resolves all libraries declared in `pom.xml` with correct versions from Maven Central and provides a standard build lifecycle, avoiding manual JAR downloading and version conflicts.
- **How would you identify and resolve a failed Jenkins build?** Open the build's **Console Output**, read the first `[ERROR]`, fix the source or `pom.xml` locally, verify with `mvn clean package`, push, and rebuild until green.

---

### Question 3. System Monitoring using Nagios [20 MARKS]

**Question:** Scenario: During online examinations, many students access the AI-OLMS portal simultaneously. The administrator experiences slow responses and occasional application downtime. The college wants to monitor server performance using Nagios. Task: Set up Nagios Monitoring to monitor the performance and availability of the AI-OLMS application server. Expected Output: Pull and run the Nagios Docker image. Configure monitoring services for the application server. Monitor CPU usage, memory and service uptime. Configure HTTP service monitoring. Display monitoring status and alerts through the Nagios dashboard. Scenario-Based Questions: How can Nagios identify whether the AI-OLMS application server is running or down? What is the difference between WARNING and CRITICAL alerts? What happens if CPU usage reaches 100% when many students access the portal? How can Nagios detect application failure even when the server is running? How can an administrator identify whether a problem is caused by Tomcat, MySQL or the server?

**One-line explanation:** Start Nagios in Docker, log into the dashboard, check host/system services, add an HTTP check with thresholds, and demonstrate an alert and recovery.

**Solution:**
1. `docker pull jasonrivers/nagios:latest`; `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`.
2. Open `http://localhost:8888` and log in as `nagiosadmin` / `nagios`.
3. **Hosts** → `localhost` shows UP/DOWN and uptime; **Services** shows `load`, `disk`, `users`, `HTTP` with status, check time and thresholds.
4. Add HTTP service check (`check_http -H localhost -p 8085 -w 2 -c 5`) via the container config and restart; WARNING = slow, CRITICAL = not responding.
5. Stop `olms-tomcat` → HTTP turns CRITICAL with an alert; restart → recovery to OK.

```powershell
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo sh -c "cat >> /usr/local/nagios/etc/objects/localhost.cfg <<'EOF'
define service {
  use                 generic-service
  host_name           localhost
  service_description OLMS_HTTP
  check_command       check_http!-H localhost -p 8085 -w 2.0 -c 5.0
  check_interval      1
  retry_interval      1
}
EOF"
docker restart nagiosdemo
docker stop olms-tomcat
docker start olms-tomcat
```

- **How can Nagios identify whether the application server is running or down?** The host check (`check_ping`) reports UP/DOWN on the Hosts page along with uptime and last-check time.
- **What is the difference between WARNING and CRITICAL alerts?** WARNING = a warning threshold crossed but still functioning (e.g. load above 2, disk 80%); CRITICAL = hard failure (connection refused, load above 5, disk 95%) that pages the administrator.
- **What happens if CPU usage reaches 100% during online examinations?** `check_load` reports CRITICAL, an alert is sent, and the portal slows/times out; the fix is to add replicas or investigate the heavy process.
- **How can Nagios detect application failure even when the server is running?** By checking the service endpoint (`check_http` on the WAR URL) or DB port (`check_mysql`) — the host stays UP while only the service goes CRITICAL.
- **How can an administrator identify whether the problem is Tomcat, MySQL or the server?** Host DOWN → server/network; host UP + HTTP CRITICAL + MySQL OK → Tomcat; host UP + MySQL CRITICAL → MySQL. Use `docker logs` for confirmation.

---

### Question 4. Container Scaling using Kubernetes [20 MARKS]

**Question:** Scenario: During online examinations and assignment submissions, hundreds of students access the AI-OLMS application simultaneously. The college wants to increase the number of application instances to handle the additional workload. Task: Demonstrate the scaling of the AI-OLMS application using Kubernetes. Expected Output: Create a Docker image for the AI-OLMS application. Create a Kubernetes Deployment for the application. Check the running application pods. Scale the deployment by increasing the number of replicas. Verify scaling using Kubernetes commands. Demonstrate automatic pod recreation after deleting a pod. Scenario-Based Questions: What happens if one AI-OLMS application pod crashes during an online examination? How does Kubernetes maintain the desired number of application replicas? What is the difference between scaling up and scaling down? How can increasing application replicas improve performance during online examinations? How does a Kubernetes Service distribute incoming requests among multiple application pods?

**One-line explanation:** Build the AI-OLMS image, deploy it to Minikube, scale from 2 to more replicas, and verify self-healing and Service load balancing.

**Solution:**
1. `docker build -t olms-app:latest .` from the repo after `mvn clean package` (base image `tomcat:9.0`).
2. `minikube start --driver=docker` and `minikube image load olms-app:latest` (or push to Docker Hub).
3. `kubectl create deployment olms-app --image=olms-app:latest --replicas=2`; `kubectl get pods`.
4. `kubectl expose deployment olms-app --type=NodePort --port=8080 --target-port=8080`; access with `minikube service olms-app --url`.
5. `kubectl scale deployment olms-app --replicas=5`; verify `kubectl get deployments` and `kubectl get pods`.
6. `kubectl delete pod <name>` → a new pod is recreated automatically.

```powershell
cd AI-OLMS
mvn -f pom.xml clean package
docker build -t olms-app:latest .
minikube start --driver=docker
minikube image load olms-app:latest
kubectl create deployment olms-app --image=olms-app:latest --replicas=2
kubectl expose deployment olms-app --type=NodePort --port=8080 --target-port=8080
kubectl get deployments,pods,services
minikube service olms-app --url
kubectl scale deployment olms-app --replicas=5
kubectl get pods -o wide
kubectl delete pod <pod-name>
kubectl get pods -o wide
```

- **What happens if one pod crashes during an online examination?** That pod's students reconnect to another replica; the ReplicaSet launches a replacement pod within seconds so total capacity is restored.
- **How does Kubernetes maintain the desired number of replicas?** The ReplicaSet controller watches pod labels and compares live replicas with `spec.replicas`, creating/deleting pods to converge to the desired count.
- **What is the difference between scaling up and scaling down?** Scaling up raises the replica count for more throughput; scaling down lowers it to save CPU/memory. Both use `kubectl scale --replicas=N`.
- **How can increasing replicas improve performance during online examinations?** Requests are split across pods, so each pod handles fewer connections, lowering response time and avoiding overload; it also removes a single point of failure.
- **How does a Kubernetes Service distribute incoming requests?** The Service has a stable IP/DNS and kube-proxy load-balances traffic across all ready pods selected by its `selector`, with session affinity available if needed.

---

### Question 5. AWS Deployment using EC2 [20 MARKS]

**Question:** Scenario: A college wants students and instructors to access the AI-OLMS portal from anywhere. The college decides to deploy the Java web application on an AWS EC2 instance. Expected Output: Launch and configure an AWS EC2 instance. Access the AI-OLMS application through the EC2 public IP address. Scenario-Based Questions: What could be the reasons if students cannot access the AI-OLMS application using the EC2 public IP address? Why should HTTP and Tomcat ports be configured in the EC2 security group? What happens if Tomcat stops while the EC2 instance is still running? How can you make the AI-OLMS application available if the EC2 instance fails? What are the advantages of deploying AI-OLMS on AWS instead of a local server?

**One-line explanation:** Launch Ubuntu on EC2, allow SSH/HTTP/8080, SSH in, install Docker + Git, build and run the WAR in Tomcat, then open it using the public IP.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**; name `olms-app`; AMI **Ubuntu Server (Free tier eligible)**; **t2.micro**.
2. Create/download a `.pem` key pair; Security Group inbound **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 8080 anywhere**. Launch.
3. **Connect** → **SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (fix key permissions with `icacls` on Windows).
4. `sudo apt update && sudo apt-get install -y docker.io git`, clone AI-OLMS, write a Dockerfile using `tomcat:9.0`, `docker build -t olms-app .`, `docker run -d -p 8080:8080 --restart unless-stopped olms-app`.
5. Browser → `http://<public-ip>:8080/<context>`.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/deepthisagar7/AI-OLMS.git
cd AI-OLMS
cat > Dockerfile <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -t olms-app .
sudo docker run -d --name olms -p 8080:8080 --restart unless-stopped olms-app
curl http://localhost:8080/
```

- **What could be the reasons students cannot access the portal using the public IP?** Security Group lacks inbound 8080, Tomcat/container not running or not port-mapped, app bound to localhost only, OS firewall, wrong context path, or public IP changed after stop/start.
- **Why should HTTP and Tomcat ports be configured in the security group?** Inbound traffic is denied by default, so 80 and 8080 must be explicitly opened for browsers while SSH stays restricted to administrators.
- **What happens if Tomcat stops while EC2 is still running?** The instance is reachable but the portal is down (connection refused); restart Tomcat/container and check logs.
- **How can you make the portal available if the EC2 instance fails?** Use an ALB with Auto Scaling across multiple AZs, keep the DB in RDS, store code/data on EBS, and use Route 53/Elastic IP for a stable address.
- **What are the advantages of deploying AI-OLMS on AWS instead of a local server?** Global accessibility, elastic scaling, high availability, managed monitoring/backups, no hardware cost, and pay-as-you-go pricing.
