# 17-11-26 / SET-4 — Solutions

Original question paper: 17-11-26\SET-4.docx. Lab Internal II, Software Engineering (23CC501PC), CSM.

---

### Question 1. Deployment using Docker Compose [25 MARKS]

**Question:** Task: Deploy the web application along with a MySQL database using Docker Compose. Create a `docker-compose.yml` file defining the web application and MySQL services. Use the web application image available in the Docker Hub repository. Configure the MySQL service with the required database name, username, password, environment variables and port mapping. Start the services and verify that both containers are running and the web application is accessible. Expected Output: A valid `docker-compose.yml` file containing the web application and MySQL services. Successful startup of both services and verification of application accessibility. Scenario-Based Questions – Docker Compose: A developer wants to temporarily stop all running services without removing their containers, networks or configuration. Which Docker Compose command should be used? After stopping the services using `docker compose stop`, the developer wants to start the same containers again without recreating them. Which command should be executed? Testing is complete, and the developer wants to stop and remove the containers and the default network created by Compose. Which command should be used? A new service has been added to `docker-compose.yml`. Which command should be used to apply the updated configuration and start the services? The frontend, backend and database services are running. The developer wants to stop only the backend service while keeping the other two running. Which command should be executed?

**One-line explanation:** Write a Compose file with a web service (Docker Hub image) and MySQL with environment variables and port mapping, then manage them with Compose lifecycle commands.

**Solution:**
1. Create `docker-compose.yml` with `web` (e.g. `wordpress:latest` from Docker Hub, `-p 8000:80` — avoid 8080 used by Jenkins) and `db` (`mysql:8.0` with `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`).
2. Add `depends_on: [db]`, a named volume for `/var/lib/mysql`, and a shared network so the web container reaches the DB by service name.
3. Start with `docker compose up -d`; verify with `docker compose ps` and `docker compose logs -f`.
4. Open `http://localhost:8000` and confirm the web application loads.

```yaml
services:
  web:
    image: wordpress:latest
    container_name: lab-web
    restart: unless-stopped
    depends_on: [db]
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: labdb
      WORDPRESS_DB_USER: labuser
      WORDPRESS_DB_PASSWORD: labpass
    volumes:
      - wp_data:/var/www/html
    networks: [lab-net]
  db:
    image: mysql:8.0
    container_name: lab-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: labdb
      MYSQL_USER: labuser
      MYSQL_PASSWORD: labpass
    volumes:
      - db_data:/var/lib/mysql
    networks: [lab-net]
volumes:
  db_data:
  wp_data:
networks:
  lab-net:
```
```powershell
docker compose up -d
docker compose ps
docker compose logs -f
docker compose stop
docker compose start
docker compose down
docker compose down -v
docker compose up -d --build
docker compose stop db
```

- **Temporarily stop all services without removing anything:** `docker compose stop` — containers, networks and volumes remain and can be restarted.
- **Start the same containers again after `stop`:** `docker compose start` — reuses the existing containers (no recreation).
- **Stop and remove containers + default network:** `docker compose down` (add `-v` only if you also want to delete named volumes — data loss).
- **Apply changes after adding a service to the YAML:** `docker compose up -d --build` — recreates/creates services from the updated file.
- **Stop only the backend/database service:** `docker compose stop db` (or `stop backend`), leaving the other services running.

---

### Question 2. Jenkins Scripted Pipeline – CI/CD Automation [25 MARKS]

**Question:** Task: Create a Groovy-based Jenkins Scripted Pipeline (`Jenkinsfile`) for the Online Ticket Reservation System (OLTRS) using the repository `https://github.com/archanareddyse/OLTRS.git`. Check out the source code from the specified GitHub repository. Configure a GitHub webhook to trigger the Jenkins pipeline automatically whenever code is pushed. Build and test the OLTRS application using Maven. After a successful build and test, create a Docker image and tag it appropriately. Push the image to the student's own Docker Hub repository using Jenkins credentials. Expected Output: A working Groovy-based Jenkinsfile that executes the required CI/CD stages. Automatic pipeline triggering through GitHub webhook and successful Maven build and test, followed by Docker image creation and push. Scenario-Based Questions: A developer pushes a new commit to the OLTRS GitHub repository, but Jenkins does not start the pipeline. What configurations would you check? The Maven build fails during the Build stage. How will you ensure that the Docker Build and Push stages are skipped? The pipeline successfully builds the application, but the Docker Hub push fails due to authentication. How should credentials be configured and used securely in the Jenkinsfile? The pipeline completes the Checkout and Build stages but fails during deployment. How would you identify the failed stage and inspect its error details in Jenkins? The project team wants an email only when the pipeline fails, while successful builds should complete without an email. How would you implement this condition in the Scripted Pipeline?

**One-line explanation:** Create a scripted Jenkinsfile with checkout, Maven build/test, Docker build/tag/push, a webhook trigger, Docker Hub credentials and failure-only email.

**Solution:**
1. Fork `https://github.com/archanareddyse/OLTRS.git` into your GitHub account (its Dockerfile is empty — write it as shown below).
2. Jenkins → **New Item** → `OLTRS_CICD` → **Pipeline** → Definition **Pipeline script** → paste the script → **Save**.
3. Jenkins → **Manage Jenkins → Credentials → System → Global credentials → Add Credentials** → Username with password → Docker Hub username/password → ID `dockerhub-creds`.
4. GitHub → repo → **Settings → Webhooks → Add webhook** → Payload URL `http://<jenkins-url>/github-webhook/` (ngrok HTTPS URL for localhost) → `application/json` → "Just the push event". Jenkins job → **Configure → Build Triggers → GitHub hook trigger for GITScm polling**.
5. Script stages: Checkout → Build (`mvn clean install`) → Test (`mvn test`) → Docker Build → Docker Push. A failing stage aborts the rest. `withCredentials` injects Docker Hub login securely.

```groovy
node {
    def repo = 'https://github.com/<your-account>/OLTRS.git'
    def image = '<your-dockerhub-user>/oltrs:latest'
    try {
        stage('Checkout') { deleteDir(); bat "git clone ${repo}" }
        stage('Build')    { bat 'mvn clean install -f OLTRS' }
        stage('Test')     { bat 'mvn test -f OLTRS' }
        stage('Docker Build') {
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
                 body: "Pipeline failed at a stage: ${err.getMessage()}\n${env.BUILD_URL}console"
        currentBuild.result = 'FAILURE'
        throw err
    }
}
```

```dockerfile
# Dockerfile (OLTRS repo is a WAR project — Tomcat 9 uses javax.servlet)
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
```
```powershell
docker login
git clone https://github.com/archanareddyse/OLTRS.git
ngrok http 8080
```

- **Push does not trigger Jenkins:** Check (1) GitHub webhook exists, payload URL ends with `/github-webhook/` and is a valid public/ngrok URL, content type `application/json`, push event selected; (2) Jenkins job has **GitHub hook trigger for GITScm polling** ticked; (3) the GitHub plugin is installed; (4) webhook **Recent Deliveries** shows HTTP 200.
- **Skip Docker stages when Maven build fails:** Put Build in its own `stage` (or `try/catch`), set `currentBuild.result = 'FAILURE'` and rethrow — in scripted pipelines an exception aborts subsequent stages, so Build/Push never run. Optionally guard with `if (currentBuild.currentResult == 'SUCCESS')`.
- **Docker Hub push authentication:** Store username/password as Jenkins **Username with password** credentials with ID `dockerhub-creds` and inject with `withCredentials(...)`; never hard-code them in the Jenkinsfile. Verify locally with `docker login`.
- **Identify the failed stage:** Open the build → **Stage View** (failed stage is red) → click it → **Console Output** for the exact error; also use **Build → Console Output** and `currentBuild.result`. The `catch` block records the message.
- **Email only on failure:** Send mail only inside `catch { emailext ... }`; do not call `emailext` on the success path. Alternatively in post-build **Editable Email Notification** set triggers to **Failure only**.

---

### Question 3. Containerization and Deployment using Minikube [25 MARKS]

**Question:** Task: Deploy the Maven web application Docker image available in the student's own Docker Hub repository using Minikube and Kubernetes. Start a Minikube cluster and verify that it is running successfully. Pull and deploy the web application image from Docker Hub by creating a Kubernetes Deployment. Create a Kubernetes Service to expose the application using NodePort. Configure the service target port to match the application's container port and use a valid NodePort. Scale the Deployment to four replicas. List and verify the Kubernetes nodes, Deployment status, replica count and Pods. Delete the created Kubernetes Deployment and Service, and verify that the resources have been removed. Expected Output: A running Minikube cluster with the web application deployed successfully. Four running application Pods managed by the Deployment. A working NodePort Service and successful access to the application through the Minikube service URL. Scenario-Based Questions: The existing Deployment has two replicas and the administrator wants four — which command and how to verify? Pods are running but users cannot access from outside — which resource to check/create? Deployment configured for four but only two Pods Running — which commands to identify the reason? Service ENDPOINTS shows `<none>` — what could cause this and how to troubleshoot selector/labels? Which commands delete the Deployment and Service and verify removal?

**One-line explanation:** Start Minikube, deploy the Docker Hub image, expose it with NodePort, scale to 4, verify everything, and clean up.

**Solution:**
1. `minikube start --driver=docker`, `minikube status`, `kubectl get nodes` (Ready).
2. Deploy: `kubectl create deployment oltrs-app --image=<your-dockerhub-user>/oltrs:latest` (Kubernetes pulls from Docker Hub).
3. Expose: `kubectl expose deployment oltrs-app --type=NodePort --port=8080 --target-port=8080 --node-port=30080` — `targetPort` must match the container's 8080, NodePort in range 30000–32767.
4. Scale: `kubectl scale deployment oltrs-app --replicas=4`; verify `kubectl get nodes`, `kubectl get deployments`, `kubectl get pods -o wide`, `kubectl get svc`, `kubectl describe svc oltrs-app`.
5. Access: `minikube service oltrs-app --url` → open in browser.
6. Cleanup: `kubectl delete deployment oltrs-app`, `kubectl delete service oltrs-app`, then `kubectl get deployments,services, pods` (not found).

```powershell
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl create deployment oltrs-app --image=<your-dockerhub-user>/oltrs:latest
kubectl expose deployment oltrs-app --type=NodePort --port=8080 --target-port=8080 --node-port=30080
kubectl scale deployment oltrs-app --replicas=4
kubectl get deployments,pods,svc -o wide
kubectl describe svc oltrs-app
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

- **Scale 2 → 4:** `kubectl scale deployment oltrs-app --replicas=4` (or edit `replicas: 4` in YAML); verify with `kubectl get deployments` (`4/4`) and `kubectl get pods`.
- **Pods running but no external access:** Check/create a **Service** of type NodePort with the correct `selector`, `port`, `targetPort` (must equal container port) and a valid `nodePort`; use `minikube service <name> --url`.
- **Only 2 of 4 pods Running:** Use `kubectl get pods -o wide`, `kubectl describe pod <name>` and `kubectl logs <name>`; look for ImagePullBackOff, insufficient resources or failing readiness/liveness probes.
- **ENDPOINTS `<none>`:** The Service selector matches no pod labels. Compare `kubectl get pods --show-labels` with `kubectl describe svc oltrs-app` selector; fix `matchLabels`/labels so they match, then endpoints appear.
- **Delete and verify:** `kubectl delete deployment oltrs-app` and `kubectl delete service oltrs-app`; `kubectl get deployments,services,pods` should show no such resources.

---

### Question 4. Deployment in AWS Environment using Amazon EC2 [25 MARKS]

**Question:** Task: Deploy the web application on an Amazon EC2 instance and make it accessible through its public address. Launch an EC2 instance by selecting the specified AWS Region, operating system AMI, instance type and required configuration. Create and configure a Security Group with appropriate inbound and outbound rules. Deploy the web application and verify that it runs successfully on the instance. Verify application accessibility using a web browser. Expected Output: A successfully launched EC2 instance with the required configuration and Security Group. The web application running successfully and accessible through the instance's public address. Scenario-Based Questions: A developer cannot connect to an EC2 instance using SSH. Which instance settings, key-pair permissions and Security Group rules should be checked? The web application runs at port 8080 but users cannot access it through the public IP. What should be verified? An EC2 instance has no Public IPv4 address but must be accessed over the internet. What networking/addressing configuration is required? Application data must remain available after stop/start. Which storage option should be considered? An organization wants HTTP public but SSH restricted to administrators. How should inbound rules be configured?

**One-line explanation:** Launch free-tier Ubuntu EC2 with a correctly configured Security Group, deploy the app in Docker, verify from the public IP, and know the troubleshooting answers.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: choose Region (e.g. ap-south-1), name `oltrs-web`, AMI **Ubuntu Server (Free tier eligible)**, 64-bit, **t2.micro**.
2. Create a `.pem` key pair and store it. **Network settings → Create security group**: Inbound **SSH (22) → My IP**, **HTTP (80) → Anywhere**, **Custom TCP 8080 → Anywhere**; outbound allow all. Launch.
3. Select instance → **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`, run from the `.pem` folder (use `icacls` on Windows if needed).
4. `sudo apt update && sudo apt-get install -y docker.io git`, clone the repo, write a Dockerfile, `docker build -t oltrs-app .`, `docker run -d -p 8080:8080 --restart unless-stopped oltrs-app`.
5. Browser → `http://<public-ip>:8080`.

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

- **Cannot connect via SSH:** Verify the instance is Running, correct Region/public IP, the `.pem` matches the key pair, username is `ubuntu`, key permissions are restricted (`icacls`), and the Security Group allows inbound 22 from your IP.
- **Port 8080 not reachable via public IP:** Check Security Group inbound allows **Custom TCP 8080 from 0.0.0.0/0**, the container is running and mapped with `-p 8080:8080`, the app listens on 0.0.0.0 (not 127.0.0.1), and the OS firewall is not blocking.
- **No Public IPv4 address:** Enable **Auto-assign public IPv4** at launch, or allocate an **Elastic IP** and associate it with the instance; also ensure the subnet is public (route to an Internet Gateway).
- **Data must survive stop/start:** Use **EBS** (root or additional volume) for persistent block storage; EBS volumes survive stop/start while instance-store data does not. For a DB use RDS with automated backups.
- **Public HTTP but restricted SSH:** Inbound rules: **HTTP 80 from 0.0.0.0/0** (and 8080 if used), **SSH 22 from your IP only** (or a bastion security group); outbound allow all.
