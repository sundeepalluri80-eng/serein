# 17-11-26 / SET-8 — Solutions

Original question paper: 17-11-26\SET-8.docx. Lab Internal II, Software Engineering (23CC505PC), AI & ML.
Repository: `https://github.com/Kumbhambhargavi75/FoodSystem`

---

### Question 1. Docker Compose – Flask API, Nginx [25 MARKS]

**Question:** A restaurant wants to launch an online food ordering website. The application has a Flask API backend and an Nginx web server. Create a Docker Compose file to run both applications in separate containers, exposing Flask on port 5000 and Nginx on port 80. A restaurant has developed a management application using Flask and Nginx. The team wants to run both services on the same Docker host without port conflicts. Configure Docker Compose with appropriate port mappings. A food delivery company wants to deploy its application using Docker Compose. The Flask backend handles customer orders on port 5000, while Nginx serves the frontend on port 80. Write a Docker Compose configuration to run both containers simultaneously. A restaurant wants to make its digital menu application accessible to customers. The application uses Flask for backend operations and Nginx for serving web pages. Configure separate Docker containers and expose them through host ports 5000 and 80 respectively. A food ordering company wants to deploy its Flask API and Nginx server on a single machine. Write a Docker Compose file that runs both services independently, maps their respective ports and allows users to access both applications simultaneously.

**One-line explanation:** Write one `docker-compose.yml` with a Flask service on host 5000 and an Nginx service on host 80, attached to a shared network.

**Solution:**
1. Create a project folder with `docker-compose.yml`, `flask-app/Dockerfile` + `app.py`, and `nginx/Dockerfile` + `index.html` (or an `nginx.conf` proxying to Flask).
2. Flask service: build from `./flask-app`, map `"5000:5000"` — the API is reachable at `http://localhost:5000`.
3. Nginx service: build from `./nginx`, map `"80:80"` — the frontend at `http://localhost`.
4. Put both on one bridge network so Nginx can reach `flask-api:5000` by service name; add `depends_on: [flask-api]`.
5. Run `docker compose up -d --build`, verify `docker compose ps` and `docker compose logs -f`, then open both URLs in a browser.

```yaml
# docker-compose.yml
services:
  flask-api:
    build: ./flask-app
    container_name: food-flask
    restart: unless-stopped
    ports:
      - "5000:5000"
    networks: [food-net]

  nginx:
    build: ./nginx
    container_name: food-nginx
    restart: unless-stopped
    depends_on: [flask-api]
    ports:
      - "80:80"
    networks: [food-net]

networks:
  food-net:
    driver: bridge
```

```dockerfile
# flask-app/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir flask
EXPOSE 5000
CMD ["python", "app.py"]
```
```python
# flask-app/app.py
from flask import Flask
app = Flask(__name__)
@app.route('/')
def home(): return "Food Ordering API is running"
if __name__ == '__main__': app.run(host='0.0.0.0', port=5000)
```
```dockerfile
# nginx/Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```
```powershell
docker compose up -d --build
docker compose ps
docker compose logs -f
# open http://localhost:5000 and http://localhost
docker compose down
```

- All five scenario paragraphs ask for the same outcome: two independent containers with distinct host ports. The file above satisfies each — Flask (orders API) on `5000:5000`, Nginx (frontend/menu) on `80:80`, no conflict because the ports differ, and `depends_on`/one network let them communicate.

---

### Question 2. Jenkins Scripted Pipeline – Food Management System [25 MARKS]

**Question:** A restaurant has developed a Food Management System and wants to automate its application build process. Create a Jenkins Pipeline that clones the source code from GitHub and performs Maven clean and install operations. The food delivery company wants to verify its application before releasing a new version. Create a Jenkins Pipeline with separate Maven build, test and package stages. The Food Management System developers frequently update their source code on GitHub. Configure a Jenkins Pipeline to clone the latest repository code and execute the Maven build process whenever the pipeline runs. A food delivery company wants its Jenkins pipeline to execute automatically whenever developers make changes to the Food Management System GitHub repository. Configure an appropriate GitHub webhook or source-control build trigger. The restaurant wants to generate a deployable JAR/WAR file for its Food Management System after successful testing. Create a Jenkins Pipeline that performs GitHub checkout, Maven clean, install, test and package operations.

**One-line explanation:** Create a Jenkins Pipeline with separate checkout/clean/build/test/package stages and a webhook trigger so each GitHub push produces a fresh artifact.

**Solution:**
1. Jenkins → **New Item** → name `FoodMS_Pipeline` → **Pipeline** → **OK** → Pipeline Definition **Pipeline script** → paste script → **Save**.
2. `stage('Checkout')` clones the repo, `stage('Clean')` runs `mvn clean`, `stage('Build')` runs `mvn install`, `stage('Test')` runs `mvn test`, `stage('Package')` runs `mvn package` and archives `target/*.jar, target/*.war`.
3. Each stage is visible separately in Jenkins; a failure in one stage stops the next.
4. Auto-trigger: Job → **Configure → Build Triggers → GitHub hook trigger for GITScm polling**; GitHub repo → **Settings → Webhooks → Add webhook** → Payload `http://<jenkins>/github-webhook/`, content type `application/json`, "Just the push event". For localhost use `ngrok http 8080`.
5. Alternative for polling: **Poll SCM** → Schedule `H/5 * * * *`.

```groovy
node {
    def repo = 'https://github.com/Kumbhambhargavi75/FoodSystem.git'
    def proj = 'FoodSystem'
    stage('Checkout') { deleteDir(); bat "git clone ${repo}" }
    stage('Clean')    { bat "mvn clean -f ${proj}" }
    stage('Build')    { bat "mvn install -f ${proj}" }
    stage('Test')     { bat "mvn test -f ${proj}" }
    stage('Package')  {
        bat "mvn package -f ${proj}"
        archiveArtifacts artifacts: "${proj}/target/*.jar, ${proj}/target/*.war", allowEmptyArchive: true
    }
}
```

```powershell
git clone https://github.com/Kumbhambhargavi75/FoodSystem.git
cd FoodSystem
mvn clean install test package
Get-ChildItem target\*.jar, target\*.war -ErrorAction SilentlyContinue
ngrok http 8080
```

- Clone + clean/install: the `Checkout`, `Clean` and `Build` stages run `git clone`, `mvn clean` and `mvn install` on every build.
- Separate build/test/package stages: each `stage(...)` block isolates that command and shows status in the Stage View.
- Clone latest code on every run: `deleteDir()` clears the workspace and `git clone` fetches the current HEAD each time.
- Automatic execution on GitHub changes: webhook + **GitHub hook trigger for GITScm polling** (or Poll SCM cron) starts the pipeline on a push.
- Deployable artifact after tests: `stage('Package')` runs only if `Test` passed, then `archiveArtifacts` keeps the JAR/WAR in Jenkins.

---

### Question 3. Kubernetes – Expose & Access Nginx Using Minikube [25 MARKS]

**Question:** A restaurant has developed a Food Management System and wants to run it in a local Kubernetes environment. Start Minikube and deploy the application using its Docker image. During peak lunch hours, multiple customers access the Food Management System simultaneously. Scale the Kubernetes deployment to four replicas to handle increased traffic. Restaurant customers cannot access the Food Management System running inside Minikube. Configure a NodePort service to expose the application and verify its accessibility through a browser. A restaurant administrator wants to monitor the availability of the Food Management System server. Install and run Nagios using Docker, access its monitoring dashboard and observe the host and service status. After updating the Food Management System Docker image, the Kubernetes deployment fails and some pods remain in a Pending state. Identify the possible cause and troubleshoot the deployment using Kubernetes commands.

**One-line explanation:** Deploy the app on Minikube, scale to 4, expose with NodePort, monitor with Nagios, and diagnose Pending pods with `describe`/`events`.

**Solution:**
1. `minikube start --driver=docker`, confirm `kubectl get nodes` shows Ready.
2. Build/load the image: `docker build -t food-app:latest .` then `minikube image load food-app:latest`; create `kubectl create deployment food-app --image=food-app:latest`.
3. Scale: `kubectl scale deployment food-app --replicas=4`; verify `kubectl get pods -o wide` (4 Running).
4. Expose: `kubectl expose deployment food-app --type=NodePort --port=80 --target-port=80`; open `minikube service food-app --url` in a browser.
5. Nagios: `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest` → `http://localhost:8888` (`nagiosadmin`/`nagios`) → **Hosts**/**Services** → check OK/WARNING/CRITICAL.
6. Pending pods: `kubectl get pods` shows `Pending`; run `kubectl describe pod <name>` and `kubectl get events --sort-by=.lastTimestamp` — common causes are image not found/wrong tag (ErrImagePull), insufficient CPU/memory, or unschedulable node. Fix with `minikube image load <correct-image>` / `kubectl set image deployment/food-app food-app=<image>` or increase resources (`minikube start --cpus=4 --memory=4096`), then confirm pods are Running.

```powershell
minikube start --driver=docker
kubectl get nodes
docker build -t food-app:latest .
minikube image load food-app:latest
kubectl create deployment food-app --image=food-app:latest
kubectl scale deployment food-app --replicas=4
kubectl get pods -o wide
kubectl expose deployment food-app --type=NodePort --port=80 --target-port=80
minikube service food-app --url
# Nagios
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
# troubleshoot Pending pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl set image deployment/food-app food-app=food-app:latest
```

- **Deploy from Docker image:** `minikube start` + `docker build` + `minikube image load` makes the image available in the cluster; `kubectl create deployment` starts the pod and `kubectl get pods` verifies Running.
- **Scale to four:** `kubectl scale deployment food-app --replicas=4` and `kubectl get pods` shows four pods.
- **NodePort access:** `kubectl expose --type=NodePort` creates a service reachable through `minikube service food-app --url` in the browser.
- **Nagios monitoring:** Run the Nagios container on 8888, log in, and inspect host uptime and HTTP/load/disk service states and alerts.
- **Pods stuck Pending:** Use `kubectl describe pod` Events and `kubectl get events` to find the cause (wrong/missing image, insufficient resources, no matching node); correct the image tag or resources and re-check until all pods are Running.

---

### Question 4. AWS EC2 Deployment [25 MARKS]

**Question:** A restaurant wants to launch its online Food Management System website on AWS. Create an Ubuntu EC2 instance, install Docker and deploy the website's index.html file using an Nginx Docker container. The Food Management System development team maintains its website source code in a GitHub repository. Connect to the AWS EC2 Ubuntu instance using SSH, clone the repository and build a Docker image using a suitable Dockerfile. A food delivery company has developed its Food Management System as a Maven Web Project. Deploy the application on an AWS EC2 Ubuntu instance by cloning the project from GitHub, building its Docker image and running the container on port 9090. Customers cannot access the deployed Food Management System even though the Docker container is running on EC2. Configure the EC2 security group's inbound rules to allow application traffic on port 9090 and verify website accessibility using the public IPv4 address. A restaurant has successfully tested its Food Management System on an AWS EC2 instance. The administrator wants to stop the Docker container and terminate the EC2 instance after testing. Perform the required cleanup operations.

**One-line explanation:** Launch Ubuntu EC2, serve the site with Nginx/Tomcat containers (app on 9090), open port 9090 in the Security Group, verify, then clean up.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `food-app`, AMI **Ubuntu Server (Free tier eligible)**, **t2.micro**, create `.pem`, security group **SSH 22 from My IP**, **HTTP 80 anywhere**, **Custom TCP 9090 anywhere** → **Launch**.
2. **Connect → SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (`icacls` first on Windows).
3. `sudo apt update && sudo apt-get install -y docker.io git`; clone `https://github.com/Kumbhambhargavi75/FoodSystem.git`.
4. Static site: `FROM nginx:alpine` + `COPY index.html ...`, `docker build -t food-site .`, `docker run -d -p 80:80 food-site`.
5. Maven Web Project: multi-stage Dockerfile (`maven` → `tomcat:9.0`), `docker build -t food-web .`, `docker run -d -p 9090:8080 --name food-web food-web` — host port 9090 maps to Tomcat's 8080.
6. If inaccessible: EC2 → **Security Groups** → **Edit inbound rules** → add **Custom TCP 9090, Source 0.0.0.0/0** → Save; browse `http://<public-ip>:9090`.
7. Cleanup: `sudo docker stop food-site food-web`, `sudo docker rm food-site food-web`; EC2 → **Instance state → Terminate instance**.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/Kumbhambhargavi75/FoodSystem.git
cd FoodSystem
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
sudo docker build -t food-site .
sudo docker run -d --name food-site -p 80:80 food-site
# Maven Web Project -> app on port 9090
cat > Dockerfile.web <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -f Dockerfile.web -t food-web .
sudo docker run -d --name food-web -p 9090:8080 --restart unless-stopped food-web
curl http://localhost:9090
# cleanup
sudo docker stop food-site food-web && sudo docker rm food-site food-web
```

- **EC2 instance + Nginx index.html:** Launch free-tier Ubuntu t2.micro with a key pair, SSH in, install Docker/Git, write `FROM nginx:alpine`, build and run `-p 80:80`, open the public IP.
- **SSH, clone and Dockerfile:** Copy the SSH command from EC2 → Connect → SSH client, run from the `.pem` folder, `git clone <repo>`, create the Nginx Dockerfile shown above and `docker build -t <name> .`.
- **Maven Web Project on port 9090:** Use the multi-stage Dockerfile with Tomcat 9, `docker build -f Dockerfile.web -t food-web .`, `docker run -d -p 9090:8080 food-web` (Tomcat listens on 8080 inside, host maps 9090).
- **Configure inbound rules:** Add **Custom TCP 9090, source 0.0.0.0/0** in the instance's Security Group inbound rules; then `http://<public-ip>:9090` works.
- **Cleanup:** `docker stop`/`docker rm` the containers on the instance, then **Terminate instance** from the EC2 console to stop billing.
