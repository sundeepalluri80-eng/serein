# Software Engineering Lab — Master Pattern Handbook

Consolidated guide to recurring practical patterns in SET-1 to SET-9 and dated SET-1 to SET-8.

## 1. Pattern map

| Pattern | Main workflow | Typical scenario |
|---|---|---|
| Docker multi-container | App + DB, network, volume, ports, recovery | Tomcat/WordPress + MySQL |
| Docker Compose | Multiple services defined in YAML | Flask + Nginx; WordPress + MySQL; app + Nagios |
| Jenkins Freestyle | Git → Maven → test → archive | CI build of WAR/JAR |
| Jenkins Scripted Pipeline | Checkout → build → test → package | Hospital/food/management app |
| Jenkins CI/CD | Build → deploy → Docker build/push | Tomcat, Docker Hub, Stage View |
| Jenkins email | SMTP + success/failure notification | Failure-only mail |
| Kubernetes / Minikube | Deployment → Service → scale → verify | Nginx/custom image, NodePort |
| Kubernetes troubleshooting | Events, describe, logs, endpoints | Pending pods, ImagePullBackOff |
| Nagios | Host/service checks + thresholds + alerts | HTTP, CPU/load, memory, disk, DB |
| AWS EC2 | Launch → SSH → install → deploy → expose | Nginx, Maven app, Docker |

**Priority:** Docker/Compose, Jenkins, Kubernetes/Minikube, AWS EC2 and Nagios. The scenario names change, but the workflow remains similar.

## 2. Docker multi-container application

**Recognise it:** The question asks for the web application and MySQL in separate containers, communication, persistent records, or recovery after container deletion.

**Concepts**
- Image = packaged template; container = running instance.
- Network = container-to-container communication by name.
- Named volume = database persistence independent of container lifecycle.
- Port mapping = host access to a container port.
- Restart policy = automatic restart after certain failures.

**Workflow**
1. Pull required images (Tomcat/WordPress and MySQL).
2. Create a user-defined bridge network.
3. Create a named volume for database files.
4. Start MySQL on the network with database/user/password.
5. Start the application on the same network; configure DB host as the DB container name and port 3306.
6. Publish only the web application port.
7. Verify containers/logs and open the browser.
8. If asked, stop/start MySQL; optionally remove only the DB container and recreate it with the same volume.

**Generic PowerShell commands**
~~~powershell
docker pull mysql:8.0
docker pull tomcat:9
docker network create app-net
docker volume create app-db-data
docker run -d --name app-db --network app-net --restart unless-stopped -v app-db-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=change-me -e MYSQL_DATABASE=appdb -e MYSQL_USER=appuser -e MYSQL_PASSWORD=change-me mysql:8.0
docker ps
docker logs app-db
~~~

Start the application container with its actual image, port and required DB environment variables. For JDBC, the DB URL commonly resembles jdbc:mysql://app-db:3306/appdb. WordPress uses WORDPRESS_DB_HOST=app-db:3306. Use the variables expected by the actual application.

**Viva**
- Why separate containers? Independent scaling, updates and fault isolation.
- Why custom network? Name-based discovery and isolation.
- Why named volume? It survives container deletion and preserves DB files.
- What if MySQL stops? DB-dependent requests fail until it returns; restart policy may restart the DB container.
- How secure MySQL? Do not publish 3306 publicly; use least-privilege user, strong secret and private network.
- How troubleshoot connection failure? Check container status, shared network, DB hostname/port, credentials, DB name, readiness, logs and JDBC configuration.

**Mistakes:** using localhost as DB host inside the application container; exposing MySQL publicly; forgetting the volume; assuming depends_on guarantees DB readiness; selecting an occupied host port.

## 3. Docker Compose

**Recognise it:** The question explicitly asks for docker-compose.yml, multiple services, shared networking, port mapping or Compose lifecycle commands.

**Generic app + database YAML**
~~~yaml
services:
  db:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: change-me
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: change-me
    volumes:
      - db-data:/var/lib/mysql
    networks: [app-net]
  web:
    image: your-web-image:latest
    restart: unless-stopped
    ports:
      - "8085:8080"
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: change-me
    depends_on: [db]
    networks: [app-net]
volumes:
  db-data:
networks:
  app-net:
    driver: bridge
~~~

Replace images, ports and environment variables for the real application. Flask + Nginx generally uses separate services, with host port 5000 mapped to Flask's internal port and host port 80 mapped to Nginx's internal port. For app + Nagios, place both services on a shared network and configure Nagios to check the app using its service name/internal port.

**Commands**
~~~bash
docker compose up -d --build
docker compose ps
docker compose logs -f
docker compose stop
docker compose start
docker compose down
docker compose down -v
~~~

- up creates/recreates and starts services.
- stop stops containers but keeps them.
- start starts existing stopped containers.
- down removes containers and default network.
- down -v also removes volumes and may delete database data.
- -d means detached mode; --build builds images before starting.

## 4. Jenkins Freestyle CI

**Recognise it:** Freestyle project, GitHub integration, JDK/Maven configuration, automatic build, Maven goals, WAR/JAR, Console Output or archiving.

**Workflow**
1. Install/configure Jenkins and required Git/Maven plugins.
2. Manage Jenkins → Tools: configure JDK and Maven.
3. New Item → Freestyle project.
4. Source Code Management → Git: repository URL and branch (e.g. */main).
5. Trigger: GitHub webhook ending in /github-webhook/ and enable GitHub hook trigger for GITScm polling; or Poll SCM such as H/5 * * * *.
6. Add Maven build step: clean package (or exact goals in question).
7. Archive target/*.war or target/*.jar.
8. Save → Build Now → inspect Console Output and archived artifact.

**Maven verification**
~~~bash
mvn -version
mvn validate
mvn clean
mvn test
mvn package
mvn clean package
~~~

**Viva**
- Jenkins is an automation server for build/test/deploy.
- CI means frequent integration with automated build/test feedback.
- Compile errors produce a non-zero Maven/compiler exit; Jenkins marks build failed.
- If tests fail, do not promote/deploy the faulty artifact.
- Maven manages dependencies in pom.xml and standardises build lifecycle.
- Build creates WAR/JAR; deploy places it in a runtime such as Tomcat.
- Webhook sends an event; Poll SCM checks periodically.
- Select branch using a branch specifier such as */main.

## 5. Jenkins Scripted Pipeline

**Recognise it:** Jenkinsfile/scripted pipeline, GitHub checkout, Maven clean/install/test/package or stages.

**Reusable skeleton**
~~~groovy
node {
    try {
        stage('Checkout') {
            git branch: 'main',
                url: 'https://github.com/OWNER/REPOSITORY.git'
        }
        stage('Build') {
            bat 'mvn clean compile' // Windows agent
            // sh 'mvn clean compile' // Linux agent
        }
        stage('Test') {
            bat 'mvn test'
        }
        stage('Package') {
            bat 'mvn package'
        }
        stage('Archive') {
            archiveArtifacts artifacts: 'target/*.war',
                             fingerprint: true
        }
    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    }
}
~~~

Adapt branch, URL, agent commands and artifact extension. Use bat on Windows and sh on Linux. If the question explicitly asks mvn clean install, use that. Never hard-code credentials in the Jenkinsfile; use Jenkins Credentials.

**Typical stage sequence:** Checkout → Build → Test → Package → Deploy → Verify → Notify. Include only what the question asks.

## 6. Jenkins CI/CD, Tomcat and Docker Hub

**Recognise it:** Pipeline View/Stage View, webhook, Maven build/test, Tomcat deployment, Docker image creation/push or Jenkins credentials.

**Flow**
1. GitHub push triggers Jenkins.
2. Checkout correct repository/branch.
3. Maven validates, compiles, tests and packages.
4. Archive WAR/JAR.
5. Deploy WAR to Tomcat if asked.
6. Build/tag Docker image if asked.
7. Authenticate to Docker Hub using Jenkins Credentials.
8. Push image and verify tag in registry.
9. Inspect Stage View.
10. Send required notification.

**Docker commands**
~~~bash
docker build -t USERNAME/app-name:tag .
docker login
docker push USERNAME/app-name:tag
~~~

Use Jenkins Credentials/credentials binding; never commit passwords or tokens. If Maven fails, later deployment and Docker stages must not run.

**Tomcat:** WAR means Web Application Archive. Tomcat runs a WAR deployed to webapps or through a configured deployment plugin/API. Confirm host, port, credentials and context path. Test the application URL; successful Maven build alone does not prove deployment.

**Viva**
- CI builds/tests integrated code; CD automates delivery/deployment of validated artifacts.
- Stages show progress and pinpoint failure.
- Archive preserves build outputs.
- Credentials keep secrets out of source and logs.
- Find a failure in Stage View → failed stage → Console Output.
- Webhook failure: check URL, public reachability, selected event, delivery response and Jenkins trigger.
- Docker Hub auth failure: check token, permissions and repository name.

## 7. Jenkins email notifications

**Recognise it:** SMTP, email-ext, success/failure notifications or failure-only pipeline email.

**Configuration**
1. Install Email Extension plugin if required.
2. Manage Jenkins → System: configure SMTP host, port, authentication, sender and TLS/SSL.
3. Send test email.
4. Freestyle: add Editable Email Notification and select triggers.
5. Pipeline: call emailext in success/failure branch as requested.

**Failure-only scripted pattern**
~~~groovy
node {
    try {
        stage('Build') {
            bat 'mvn clean package'
        }
    } catch (err) {
        emailext(
            to: 'RECIPIENT@example.com',
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build failed. Check: ${env.BUILD_URL}"
        )
        currentBuild.result = 'FAILURE'
        throw err
    }
}
~~~

Replace recipient and use lab SMTP settings. Never put SMTP passwords in the script. Failure-only means no email on success.

## 8. Kubernetes with Minikube

**Recognise it:** Minikube, Deployment, Pods, replicas, Service, NodePort, scaling, self-healing, rollout/rollback or browser access.

**Objects**
- Cluster: Kubernetes environment.
- Node: machine running workloads.
- Pod: smallest deployable unit.
- Deployment: manages ReplicaSets and desired replicas.
- ReplicaSet: maintains requested Pod count.
- Service: stable endpoint/load-balancing abstraction.
- NodePort: exposes Service on a port on each node.
- Minikube: local Kubernetes environment.

**Workflow**
1. Start Minikube and verify node Ready.
2. Create Deployment using required image.
3. Check Deployment and Pods.
4. Expose Deployment with NodePort or requested Service type.
5. Check Service selector, ports and endpoints.
6. Scale to requested replicas.
7. Verify all desired replicas Ready/Running.
8. Open using minikube service SERVICE --url.
9. If asked, delete a Pod to demonstrate self-healing.
10. If asked, update image, inspect rollout and rollback.

**Commands**
~~~bash
minikube start
minikube status
kubectl get nodes
kubectl create deployment demo-app --image=nginx
kubectl get deployments
kubectl get pods -o wide
kubectl expose deployment demo-app --type=NodePort --port=80 --target-port=80
kubectl get services
kubectl get endpoints demo-app
minikube service demo-app --url
kubectl scale deployment demo-app --replicas=4
kubectl get deployments
kubectl get pods
kubectl delete pod POD_NAME
kubectl rollout status deployment/demo-app
kubectl rollout history deployment/demo-app
kubectl set image deployment/demo-app nginx=nginx:1.27
kubectl rollout undo deployment/demo-app
kubectl delete service demo-app
kubectl delete deployment demo-app
~~~

For a custom image, load it into Minikube (minikube image load IMAGE:TAG) or push it to a registry. A host-built image is not automatically available inside Minikube.

**Viva**
- Deployment manages desired replicas and rolling updates.
- ReplicaSet creates a replacement when a Pod disappears.
- Service provides stable access because Pod IPs can change.
- Manual scaling uses kubectl scale; autoscaling uses HPA and metrics.
- NodePort exposes a Service; Minikube may require its service URL/tunnel.
- Rollback: kubectl rollout undo deployment/NAME.
- Selector labels identify Pods managed or targeted by a resource.

## 9. Kubernetes troubleshooting

| Symptom | Commands | Inspect |
|---|---|---|
| Pod Pending | kubectl describe pod NAME; kubectl get events | CPU/memory, scheduling, PVC, node state |
| ImagePullBackOff | kubectl describe pod NAME | Image/tag, registry access, imagePullSecrets |
| CrashLoopBackOff | kubectl logs NAME; kubectl logs NAME --previous | Startup errors, config, missing env vars |
| No Service endpoints | kubectl get svc; kubectl get pods --show-labels; kubectl describe svc NAME | Selector must match Pod labels; Pods must be Ready |
| Browser cannot access | kubectl get svc; kubectl get endpoints; minikube service NAME --url | NodePort, targetPort, app port, Service type |
| Some replicas not Running | kubectl get pods -o wide; kubectl describe pod NAME | Events, resources, image, probes |
| Wrong image after update | kubectl describe deployment NAME; kubectl rollout history deployment/NAME | Image tag and rollout state |

**Debug order:** Deployment → Pods → Pod Events/Logs → Service → Endpoints → port mapping → browser.

## 10. Nagios monitoring

**Recognise it:** Host uptime, CPU/load, memory, disk, HTTP, MySQL, dashboard, warning/critical states, alerts or recovery.

**Workflow**
1. Run provided Nagios image/container or install as specified.
2. Map dashboard container port to unused host port.
3. Open dashboard using credentials supplied by lab image/instructions.
4. Verify host is UP.
5. Inspect configured services (load/CPU, disk, users, HTTP, DB if configured).
6. Add required host/service checks and thresholds.
7. If asked, stop web container and observe CRITICAL.
8. Restore service and verify OK.

**States**
- OK: check passed.
- WARNING: warning threshold crossed.
- CRITICAL: service down or critical threshold crossed.
- UNKNOWN: check could not determine a valid state, often due to plugin/configuration issue.

**Viva**
- Host check tests machine reachability; service check tests HTTP/DB/etc.
- Detect app failure while server is up using application-level HTTP check, not only ping.
- If Nagios fails, its monitoring/alerts stop; external/redundant monitoring avoids blind spots.
- Distinguish Tomcat/MySQL/network by comparing host reachability, HTTP check, DB check and logs.
- Thresholds define when state becomes WARNING or CRITICAL.

## 11. AWS EC2 deployment

**Recognise it:** Launch Ubuntu EC2, SSH, install Docker/Git/Java, clone GitHub, deploy Nginx/Tomcat/Maven app, Security Group or public IPv4.

**Workflow**
1. AWS Console → EC2 → Launch Instance.
2. Select requested Ubuntu AMI and instance type allowed by lab/account.
3. Create/download key pair; keep private key secure.
4. Security Group: SSH 22 from My IP only; HTTP 80 if needed; app port 8080/9090 only if required and restricted where possible.
5. Launch and wait for instance status checks.
6. Connect using SSH command from EC2 console.
7. Update packages and install tools.
8. Clone repository, build app or create Dockerfile.
9. Run app/container with correct host-to-container port mapping.
10. Test locally with curl, then from browser using public IPv4 and mapped port.
11. Clean up only as the question directs.

**Ubuntu + Nginx Docker example**
~~~bash
sudo apt update
sudo apt install -y git docker.io
sudo systemctl enable --now docker
git clone https://github.com/OWNER/REPOSITORY.git
cd REPOSITORY
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
sudo docker build -t lab-web .
sudo docker run -d --name lab-web -p 80:80 --restart unless-stopped lab-web
sudo docker ps
curl -I http://localhost
~~~

SSH format: ssh -i key.pem ubuntu@PUBLIC_IPV4

For Maven web apps, Dockerfile must match real packaging/runtime. Not every Maven project produces a runnable JAR; WAR generally needs Tomcat. Never upload or commit the EC2 private key.

**Viva**
- Security Groups are instance-level virtual firewalls.
- Restrict SSH to reduce brute-force/unauthorised access.
- If container runs but page is inaccessible, check Security Group, port mapping, app bind address, logs, OS firewall and public IP.
- If Tomcat stops while EC2 runs, VM remains up but app endpoint is unavailable.
- Improve availability with multiple instances/AZs, Load Balancer health checks, Auto Scaling, monitoring and backups.
- CloudWatch provides AWS metrics, logs and alarms.

## 12. Cross-tool integration chain

| Tool | Responsibility | Proof |
|---|---|---|
| Git/GitHub | Source control | Repository and branch |
| Maven | Build/dependencies | clean package; WAR/JAR in target |
| Jenkins | Automation | Successful build and Stage View |
| Tomcat | Runs Java WAR | Application URL responds |
| Docker | Packages app | Running container and mapped port |
| Docker Compose | Runs related containers | docker compose ps |
| Docker Hub | Stores images | Image tag visible in registry |
| Kubernetes | Orchestrates containers | Deployment, Pods, Service, replicas |
| Minikube | Local Kubernetes | Ready node and accessible Service |
| Nagios | Monitoring | Dashboard and alert/recovery |
| AWS EC2 | Cloud VM | Instance checks and public endpoint |

**Integration order:** Source → Build/Test → Package → Containerize (if asked) → Publish image (if asked) → Deploy → Expose → Verify → Monitor → Notify.

## 13. Practical answer-writing template

For almost any 20/25-mark practical, write in this order:
1. **Aim:** one sentence describing deployment/configuration.
2. **Requirements:** tools, images, repository, ports and credentials (never expose real secrets).
3. **Configuration:** YAML, Jenkins job, Deployment or EC2 settings.
4. **Execution:** numbered commands in correct order.
5. **Verification:** exact commands and expected state (container Up, build Success, Pods Ready, Service endpoint, browser page).
6. **Failure demonstration:** only if asked; stop service, break build, delete Pod or inspect alert, then recover.
7. **Viva:** why each component is used and how to troubleshoot.
8. **Cleanup:** remove resources only when asked; do not delete persistent volumes unless intentional.

### Verification cheat sheet
~~~bash
# Docker
docker ps
docker logs CONTAINER
docker inspect CONTAINER
docker network ls
docker volume ls

# Compose
docker compose ps
docker compose logs -f

# Maven
mvn -version
mvn clean test package

# Kubernetes
kubectl get nodes
kubectl get deployments,pods,services
kubectl describe pod POD
kubectl logs POD
kubectl get endpoints SERVICE
kubectl get events

# Linux / EC2
sudo systemctl status docker
sudo docker ps
curl -I http://localhost:PORT
~~~

## 14. Common scenario questions

| Question | Answer |
|---|---|
| Container stops? | Restart policy may restart it; inspect logs and dependencies. |
| Data after container deletion? | Named volume persists independently of container. |
| App cannot connect to DB? | Check network, hostname, port, credentials, readiness and logs. |
| Jenkins build fails? | Read first meaningful error in Console Output; fix and rebuild. |
| WAR generated but app inaccessible? | Build is not deployment; check Tomcat, context path, port and logs. |
| Push does not trigger Jenkins? | Check webhook URL/delivery, public reachability and job trigger. |
| Docker Hub auth fails? | Use Jenkins Credentials; verify token and permissions. |
| Kubernetes Pod Pending? | Describe Pod and inspect Events for scheduling/resource/PVC problems. |
| Service has no endpoints? | Service selector must match ready Pod labels. |
| Kubernetes app not reachable? | Check Service type, selector, port/targetPort, NodePort and Minikube URL. |
| Nagios CRITICAL? | Inspect check, threshold and target logs; restore and verify OK. |
| EC2 page not opening? | Check instance, Security Group, public IP, process, bind address and mapping. |

## 15. 30-minute revision plan

| Time | Topic | Must know |
|---|---|---|
| 0–5 min | Docker + Compose | Network, volume, ports, up/stop/down |
| 5–11 min | Jenkins | Checkout, Maven stages, artifact, webhook |
| 11–17 min | Kubernetes | Deployment, Service, NodePort, scale, rollout |
| 17–21 min | AWS EC2 | Launch, SSH, Security Group, Docker, public URL |
| 21–25 min | Nagios | Host/service checks, thresholds, alert/recovery |
| 25–30 min | Viva/debug | Why, what-if and how-to-fix answers |

### Final memory hooks
- **Docker:** Network connects; volume remembers; port publishes; restart policy recovers.
- **Compose:** YAML defines; up starts; stop pauses; down removes.
- **Jenkins:** Checkout → Build → Test → Package → Archive/Deploy → Notify.
- **Kubernetes:** Deployment desires replicas; ReplicaSet replaces Pods; Service provides stable access.
- **Nagios:** Host reachability + service checks + thresholds + alerts.
- **AWS:** EC2 runs VM; SSH manages it; Security Group opens ports; Docker runs app.

## 16. Source papers covered

This guide consolidates recurring patterns from:
- [SET-1 solutions](SET-1-SOLUTIONS.md) through [SET-9 solutions](SET-9-SOLUTIONS.md)
- [Dated SET-1 solutions](17-11-26/SET-1-SOLUTIONS.md) through [Dated SET-8 solutions](17-11-26/SET-8-SOLUTIONS.md)

Use individual solution files for exact scenario wording, application names, repository URLs, marks and paper-specific configuration. This master file focuses on reusable patterns and viva logic.
