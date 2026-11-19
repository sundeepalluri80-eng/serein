# SET-8 — Solutions

Original question paper: `SET-8.docx` (top level). Lab Internal II, Software Engineering (23CC501PC), CSM.

---

### Question 1. Docker Compose – Multi-Container Deployment with Nagios [25 MARKS]

**Question:** Task: Deploy a web application and Nagios monitoring tool as separate services using Docker Compose. Create a `docker-compose.yml` file defining the web application and Nagios as separate services. Use the web application Docker image available in Docker Hub repository. Configure the required ports and networking so that the services can communicate where required. Start the services and verify that both the web application and Nagios interface are accessible through a browser. Expected Output: A valid `docker-compose.yml` file containing the web application and Nagios services. Successful deployment of both containers using Docker Compose. Scenario-Based Questions: A developer wants to temporarily stop all running services without removing their containers, networks or configuration. Which command? After stopping with `docker compose stop`, which command starts the same containers again? Testing complete — which command stops and removes the containers and default network? A new service has been added to the YAML — which command applies the update? The web application container is running but Nagios reports the application is unavailable. What would you check in the Nagios host/service configuration, container networking and port mapping?

**One-line explanation:** Write a Compose file with a web service and a Nagios service on a shared network, then manage them with Compose commands and debug failed checks.

**Solution:**
1. Create `docker-compose.yml` with `web` (`wordpress:latest` or your Docker Hub app image, mapped to `8000:80`) and `nagios` (`jasonrivers/nagios:latest`, mapped to `8888:80`).
2. Put both on one bridge network so Nagios can reach the web service by name; add `depends_on: [web]` if Nagios should start after the app.
3. `docker compose up -d`, then `docker compose ps` — both containers Up.
4. Browser checks: `http://localhost:8000` (web app) and `http://localhost:8888` (Nagios, login `nagiosadmin`/`nagios`) → **Hosts**/**Services**.
5. If Nagios says the app is unavailable: verify the check command points to the right host and port (web container name + 80, or host 8000 from the Nagios container), confirm both are on the same Compose network, check `ports` mapping, and read `docker compose logs nagios`.

```yaml
services:
  web:
    image: wordpress:latest
    container_name: lab-web
    restart: unless-stopped
    ports:
      - "8000:80"
    networks: [lab-net]
  nagios:
    image: jasonrivers/nagios:latest
    container_name: lab-nagios
    restart: unless-stopped
    depends_on: [web]
    ports:
      - "8888:80"
    networks: [lab-net]
networks:
  lab-net:
    driver: bridge
```
```powershell
docker compose up -d
docker compose ps
docker compose logs -f nagios
docker compose stop
docker compose start
docker compose down
docker compose up -d --build
```

- **Stop all services without removing anything:** `docker compose stop` — keeps containers, networks and config.
- **Start the same containers again:** `docker compose start`.
- **Stop and remove containers + default network:** `docker compose down` (`-v` also deletes volumes — data loss).
- **Apply an updated Compose file:** `docker compose up -d --build`.
- **Nagios reports app unavailable:** Check the Nagios host/service definition (host name and check command, e.g. `check_http!-H web -p 80`), ensure both containers share `lab-net` so the name `web` resolves, verify the `ports` mapping (`8000:80`) and inspect `docker compose logs nagios` / `docker inspect` for the real error.

---

### Question 2. Jenkins Pipeline View – Build Automation and Deployment [25 MARKS]

**Question:** Task: Create a Jenkins Pipeline for the web application using the following GitHub repository. Fork the repository into your own GitHub account: `https://github.com/archanareddyse/OLTRS.git`. The pipeline should perform: Source Code Management: Clone the forked GitHub repository. Continuous Integration (CI): Configure GitHub Webhook to trigger the pipeline automatically on code commits; build the application using Maven and execute test cases; generate the WAR file and deploy it on Apache Tomcat. Continuous Delivery (CD): Create a Docker image of the application; tag and push the image to the student's own Docker Hub repository using Jenkins credentials. Expected Output: Successful webhook-triggered pipeline execution. Successful Maven build/test, WAR generation and deployment on Apache Tomcat. Successful Docker image creation and push to Docker Hub. A Jenkins Pipeline/Stage View displaying stage-wise status and email notification of the result. Scenario-Based Questions: A push does not trigger Jenkins — what to check? Maven build fails — how to skip Docker Build/Push? Docker Hub push fails on authentication — how to configure credentials? Pipeline fails during deployment — how to identify the failed stage and inspect errors? Email only on failure — how to implement in Scripted Pipeline?

**One-line explanation:** Build a forked-OLTRS pipeline with webhook CI, Maven build/test, WAR deployment to Tomcat on 8085, Docker image push, Stage View and failure email.

**Solution:**
1. Fork `https://github.com/archanareddyse/OLTRS.git` into your GitHub account; write the missing `Dockerfile` (Tomcat 9 base).
2. Jenkins → **New Item** → `OLTRS_FullPipeline` → **Pipeline** → Definition **Pipeline script** → paste script → **Save**.
3. GitHub → your fork → **Settings → Webhooks → Add webhook** → Payload URL `https://<ngrok>/github-webhook/` → `application/json` → push event. Jenkins job → **Build Triggers → GitHub hook trigger for GITScm polling**.
4. Credentials: **Manage Jenkins → Credentials → Add Credentials → Username with password**, ID `dockerhub-creds`.
5. Tomcat locally: edit `C:\apache-tomcat-9\conf\server.xml` HTTP port to `8085`, `tomcat-users.xml` add `manager-gui`/`manager-script` roles and user `admin`/`1234`, run `bin\startup.bat`.
6. After a build, open the job's **Stage View** and **Console Output**; check the email on failure.

```groovy
node {
    def image = '<your-dockerhub-user>/oltrs:latest'
    try {
        stage('Checkout') { deleteDir(); bat 'git clone https://github.com/<your-account>/OLTRS.git' }
        stage('Build & Test') {
            bat 'mvn clean install -f OLTRS'
            bat 'mvn test -f OLTRS'
        }
        stage('Deploy WAR to Tomcat') {
            bat 'powershell -Command "Copy-Item OLTRS\\target\\*.war C:\\apache-tomcat-9\\webapps\\ROOT.war -Force"'
        }
        stage('Docker Build & Tag') {
            bat 'docker build -t oltrs:latest -f OLTRS/Dockerfile OLTRS'
            bat "docker tag oltrs:latest ${image}"
        }
        stage('Docker Push') {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                bat 'echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin'
                bat "docker push ${image}"
            }
        }
    } catch (err) {
        emailext to: 'team@example.com', subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline failed: ${err.getMessage()}\n${env.BUILD_URL}console"
        currentBuild.result = 'FAILURE'
        throw err
    }
}
```

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
```
```powershell
ngrok http 8080
docker login
# Tomcat manager: http://localhost:8085/manager/html   admin / 1234
```

- **Push does not trigger Jenkins:** Verify the webhook URL ends with `/github-webhook/`, is publicly reachable (ngrok), content type is `application/json`, push event is selected, the job ticks **GitHub hook trigger for GITScm polling**, and GitHub **Recent Deliveries** shows 200.
- **Skip Docker stages on Maven failure:** Wrap stages in `try/catch`, set `currentBuild.result='FAILURE'` and rethrow; a failed stage aborts later stages. Optionally guard Docker stages with `if (currentBuild.currentResult == 'SUCCESS')`.
- **Docker Hub auth failure:** Create Jenkins **Username with password** credentials with ID `dockerhub-creds` and inject them with `withCredentials(...)`; never hard-code the password. Verify `docker login` works locally.
- **Find the failed stage:** Use the job's **Stage View** (red stage) → click it → **Console Output** for the exact error; the `catch` email contains the message and build URL.
- **Email only on failure:** Call `emailext` only in `catch` (success path sends nothing), or set **Editable Email Notification → Triggers = Failure only**.

---

### Question 3. Containerization and Deployment using Minikube [25 MARKS]

**Question:** Task: Deploy the Maven web application Docker image available in the student's own Docker Hub repository using Minikube and Kubernetes. Start a Minikube cluster and verify it is running successfully. Create a Kubernetes Deployment to deploy the application image from Docker Hub. Create a Kubernetes Service to expose the application using NodePort; configure `targetPort` to match the application's container port and use a valid NodePort. Verify the Deployment and Service using kubectl. Scale the Deployment to four replicas. List and verify nodes, Deployment status, replica count and Pods; confirm all four replicas are Running. Verify the Service configuration, NodePort mapping and application accessibility through the Minikube service URL. Scenario-Based Questions: scale 2 → 4 and verify; pods running but no external access; 4 replicas configured but only 2 Running; Service ENDPOINTS `<none>`; delete Deployment and Service and verify.

**One-line explanation:** Deploy the Docker Hub image in Minikube with a NodePort Service, scale to 4, verify every resource, and clean up.

**Solution:**
1. `minikube start --driver=docker`; `minikube status`; `kubectl get nodes` (Ready).
2. `kubectl create deployment oltrs-app --image=<your-dockerhub-user>/oltrs:latest`.
3. `kubectl expose deployment oltrs-app --type=NodePort --port=8080 --target-port=8080 --node-port=30080` — `targetPort` matches the Tomcat container port 8080.
4. Verify: `kubectl get deployments`, `kubectl get svc oltrs-app`, `kubectl describe svc oltrs-app`, `kubectl get endpoints oltrs-app`.
5. Scale: `kubectl scale deployment oltrs-app --replicas=4`; `kubectl get pods -o wide` → 4 Running; `minikube service oltrs-app --url` → browser.

```powershell
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl create deployment oltrs-app --image=<your-dockerhub-user>/oltrs:latest
kubectl expose deployment oltrs-app --type=NodePort --port=8080 --target-port=8080 --node-port=30080
kubectl get deployments,svc -o wide
kubectl describe svc oltrs-app
kubectl get endpoints oltrs-app
kubectl scale deployment oltrs-app --replicas=4
kubectl get pods -o wide
minikube service oltrs-app --url
kubectl delete deployment oltrs-app
kubectl delete service oltrs-app
kubectl get deployments,services,pods
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: oltrs-app }
spec:
  replicas: 4
  selector:
    matchLabels: { app: oltrs-app }
  template:
    metadata:
      labels: { app: oltrs-app }
    spec:
      containers:
        - name: oltrs-app
          image: <your-dockerhub-user>/oltrs:latest
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: oltrs-app }
spec:
  type: NodePort
  selector: { app: oltrs-app }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

- **Scale 2 → 4:** `kubectl scale deployment oltrs-app --replicas=4`; verify `kubectl get deployments` shows `4/4` and `kubectl get pods` lists four pods.
- **No external access:** Check/create a NodePort Service with correct `selector`, `port`, `targetPort=8080` and a valid `nodePort`; use `minikube service oltrs-app --url`.
- **Only 2 of 4 Running:** `kubectl get pods -o wide`, `kubectl describe pod <name>`, `kubectl logs <name>` — look for ImagePullBackOff, resource pressure or failing probes.
- **ENDPOINTS `<none>`:** Service selector does not match pod labels; compare `kubectl get pods --show-labels` with the Service selector and fix the labels.
- **Delete and verify:** `kubectl delete deployment oltrs-app`, `kubectl delete service oltrs-app`; `kubectl get deployments,services,pods` confirms removal.

---

### Question 4. Deployment in AWS Environment using Amazon EC2 [25 MARKS]

**Question:** Task: Deploy the web application on an Amazon EC2 instance and make it accessible through its public address. Launch an EC2 instance by selecting the specified AWS Region, operating system AMI, instance type and required configuration. Create and configure a Security Group with appropriate inbound and outbound rules. Clone the web application source code from the specified GitHub repository. Deploy the web application and verify that it runs successfully. Verify accessibility through the EC2 Public IPv4 address or Public DNS using a browser or AWS CLI. Scenario-Based Questions: cannot connect using SSH — which settings/permissions/rules to check? App runs at port 8080 but not reachable via public IP — what to verify? No Public IPv4 but internet access needed — what configuration? Data must survive stop/start — which storage? Public HTTP but SSH restricted — how to configure inbound rules?

**One-line explanation:** Launch free-tier Ubuntu EC2, open the right Security Group ports, clone and run the app in Docker, verify via public IP/DNS, and know the troubleshooting answers.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: Region (e.g. ap-south-1), name `oltrs-web`, AMI **Ubuntu Server (Free tier eligible)**, 64-bit, **t2.micro**.
2. Create `.pem` key pair; **Network settings → Create security group**: inbound **SSH 22 → My IP**, **HTTP 80 → Anywhere**, **Custom TCP 8080 → Anywhere**; outbound all. Launch.
3. **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (use `icacls` if needed).
4. `sudo apt update && sudo apt-get install -y docker.io git`, clone `https://github.com/archanareddyse/OLTRS.git`, write the Dockerfile, `docker build -t oltrs-app .`, `docker run -d -p 8080:8080 --restart unless-stopped oltrs-app`.
5. Browser → `http://<public-ip>:8080` or the Public DNS.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/archanareddyse/OLTRS.git
cd OLTRS
cat > Dockerfile <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -t oltrs-app .
sudo docker run -d --name oltrs -p 8080:8080 --restart unless-stopped oltrs-app
curl http://localhost:8080
```

- **SSH failure:** Check the instance is Running, correct Region/public IP, matching `.pem`, username `ubuntu`, restricted key permissions, and Security Group inbound 22 from your IP.
- **Port 8080 not reachable:** Verify Security Group inbound allows **Custom TCP 8080 from 0.0.0.0/0**, the container is mapped with `-p 8080:8080`, the app listens on 0.0.0.0, and no OS firewall blocks it.
- **No Public IPv4:** Enable Auto-assign public IPv4 at launch or attach an **Elastic IP**; ensure the subnet is public with a route to an Internet Gateway.
- **Data after stop/start:** Use **EBS** volumes (persistent block storage); instance-store data is lost. For databases use RDS with backups.
- **Public HTTP, restricted SSH:** Inbound: **HTTP 80 from 0.0.0.0/0** (and 8080), **SSH 22 from your IP only**; outbound allow all.
