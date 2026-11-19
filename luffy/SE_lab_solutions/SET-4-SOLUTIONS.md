# SET-4 — Solutions

Original question paper: `SET-4.docx` (top level). Lab Internal II, Software Engineering (23CC505PC).
Repository: `https://github.com/Kumbhambhargavi75/HospitalManagementSystem`

---

### Question 1. Jenkins Scripted Pipeline [25 MARKS]

**Question:** a) You are assigned to automate the build process of a Hospital Management System hosted on GitHub. Write a Jenkins Scripted Pipeline. b) During the execution of the Hospital Management System pipeline, the Maven build may fail due to compilation errors. How do you solve this? c) The development team wants to organise the Hospital Management System CI process into separate stages. Write a Jenkins Scripted Pipeline. d) A hospital wants to ensure that its management application works correctly before deployment. Write a Jenkins Scripted Pipeline and solve it. e) A hospital software development team wants Jenkins to automatically build and test the Hospital Management System whenever a developer triggers the pipeline.

**One-line explanation:** Write a Groovy `node { stage { ... } }` scripted Jenkinsfile that clones, builds, tests and packages the Hospital Management System, with separate stages and a trigger.

**Solution:**
1. Jenkins → **New Item** → name `HMS_Pipeline` → **Pipeline** → **OK** → scroll to **Pipeline** section → Definition: **Pipeline script** → paste the script below → **Save** → **Build Now**.
2. The script clones the repo with `bat 'git clone ...'`, runs `mvn clean`, `mvn install`, `mvn test`, `mvn package` using the configured tool name `MAVEN-HOME`.
3. Separate `stage(...)` blocks give a stage view in Jenkins; a failing command inside a stage aborts the rest automatically.
4. `mvn test` is the gate before deployment — if tests fail the pipeline stops and nothing is deployed.
5. For automatic runs: Job → **Configure** → **Build Triggers** → tick **GitHub hook trigger for GITScm polling** (webhook) or **Poll SCM** `H/5 * * * *`, or use `triggers { githubPush() }` in the script.

```groovy
node {
    def repo = 'https://github.com/Kumbhambhargavi75/HospitalManagementSystem.git'
    def proj = 'HospitalManagementSystem'

    stage('Checkout') {
        deleteDir()                                  // clean workspace
        bat "git clone ${repo}"                      // (a) clone source
    }
    stage('Clean') {
        bat "mvn clean -f ${proj}"                   // (c) separate stage
    }
    stage('Build') {
        bat "mvn install -f ${proj}"                 // (a) compile + install
    }
    stage('Test') {
        bat "mvn test -f ${proj}"                    // (d) gate before deploy
    }
    stage('Package') {
        bat "mvn package -f ${proj}"                 // produce the artifact
        archiveArtifacts artifacts: "${proj}/target/*.war, ${proj}/target/*.jar", allowEmptyArchive: true
    }
}
```

```powershell
# local verification of the same commands
git clone https://github.com/Kumbhambhargavi75/HospitalManagementSystem.git
cd HospitalManagementSystem
mvn clean install test package
Get-ChildItem target\*.war, target\*.jar -ErrorAction SilentlyContinue
```

- **(a) Automate the build:** The script above checks out the GitHub repo and runs the Maven lifecycle (`clean → install → test → package`) inside Jenkins, so every run produces a fresh artifact without manual commands.
- **(b) Maven build fails with compilation errors:** Open **Build Now → Console Output**, read the first `[ERROR] ... .java:[line,col]` message, fix the Java source or `pom.xml` dependency version locally, verify with `mvn clean install`, push the fix, and rebuild. The script stops at the failing stage, so later stages do not run.
- **(c) Organise into separate stages:** Each `stage('Checkout')`, `stage('Clean')`, `stage('Build')`, `stage('Test')`, `stage('Package')` block is a separate stage; Jenkins shows them individually in the build's Stage View and marks the exact failed stage red.
- **(d) Verify before deployment:** `stage('Test') { bat "mvn test -f ${proj}" }` runs unit tests; a non-zero exit aborts the pipeline before any deployment stage. Add `bat "mvn verify -f ${proj}"` for integration checks if needed.
- **(e) Automatic build and test on trigger:** Enable **Build Triggers → GitHub hook trigger for GITScm polling** and configure a GitHub webhook (`Settings → Webhooks → Add webhook`, Payload `http://<jenkins-url>/github-webhook/`, `application/json`, push event); or tick **Poll SCM** with `H/5 * * * *`.

---

### Question 2. Jenkins Email Notification Configuration [25 MARKS]

**Question:** a) You are working on the Hospital Management System project. The development team wants to receive an email whenever a Jenkins build fails. Configure Jenkins Email Notifications using the Email Extension Plugin and send an automatic failure notification to the project team. b) The development team wants to automate failure alerts. Write a Jenkins Scripted Pipeline that clones the GitHub repository, builds the application using Maven, executes test cases and sends an email to the team whenever the build fails using the `emailext` command. c) As a Jenkins administrator, verify the email notification configuration. Create a Jenkins pipeline, intentionally introduce a build failure using an invalid command, detect the failure and send an email alert. d) The team wants notifications for both successful and failed builds. Configure Jenkins Email Notifications to send a success email when the build completes successfully and a failure email when it fails.

**One-line explanation:** Install/configure the Email Extension plugin with Gmail SMTP, then add `emailext` in a scripted pipeline (or an Editable Email Notification post-build action) to mail the team on failure and success.

**Solution:**
1. Gmail: **myaccount.google.com → Security → 2-Step Verification ON → App passwords → Other (Custom name) "Jenkins-Demo" → Generate** and copy the 16-digit password.
2. Jenkins → **Manage Jenkins → Plugins → Available** → search `Email Extension Plugin` → install (already active on this machine). Then **Manage Jenkins → Configure System → E-mail Notification**: SMTP server `smtp.gmail.com`, tick **Use SMTP Authentication**, User = your Gmail ID, Password = 16-digit app password, tick **Use SSL**, SMTP Port `465`; click **Test configuration by sending test e-mail**.
3. In the same page, **Extended E-mail Notification**: SMTP server `smtp.gmail.com`, port `465`, SSL on, add Gmail credentials, default content type `text/html`.
4. Job → **Configure → Post-build Actions → Add post-build action → Editable Email Notification** → Project Recipient List = team emails → Triggers = **Failure** (and **Success** for part d) → Save.
5. Script version: wrap the build in `try/catch` and call `emailext` in the `catch` for failure, plus `emailext` after success if required.

```groovy
node {
    def team = 'hospital-team@example.com'
    try {
        stage('Checkout') { deleteDir(); bat 'git clone https://github.com/Kumbhambhargavi75/HospitalManagementSystem.git' }
        stage('Build')    { bat 'mvn install -f HospitalManagementSystem' }
        stage('Test')     { bat 'mvn test -f HospitalManagementSystem' }
        // (d) success mail
        emailext to: team, subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build ${env.BUILD_URL} completed successfully."
    } catch (err) {
        // (b) failure mail via emailext
        emailext to: team, subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build failed: ${err.getMessage()}\nConsole: ${env.BUILD_URL}console",
                 attachLog: true
        currentBuild.result = 'FAILURE'
        throw err
    }
}
```

```groovy
// (c) intentional failure to verify the alert
node {
    try {
        bat 'this-command-does-not-exist'      // forces a non-zero exit
    } catch (err) {
        emailext to: 'hospital-team@example.com',
                 subject: "TEST ALERT - intentional failure ${env.JOB_NAME}",
                 body: "Verification pipeline failed as planned: ${err.getMessage()}"
        throw err
    }
}
```

- **(a) Failure email via plugin:** Install Email Extension, set Gmail SMTP (`smtp.gmail.com`, SSL, 465, app password) in Configure System, then in the job add **Editable Email Notification** with the team's addresses and the **Failure** trigger.
- **(b) `emailext` in the pipeline:** The `catch` block above sends a mail with the build number and console link whenever clone/build/test throws; `attachLog: true` attaches the console log.
- **(c) Verify the configuration:** The second snippet runs an invalid command (`this-command-does-not-exist`), which fails; the `catch` sends the test alert. Check the inbox and the build's Console Output; also use **Test configuration by sending test e-mail** in Configure System.
- **(d) Success and failure mails:** Add both the success `emailext` call (inside `try`) and the failure one (inside `catch`), or in **Editable Email Notification → Triggers** add **Success** and **Failure** with separate recipients/subjects.

---

### Question 3. Working with Minikube and Nagios [25 MARKS]

**Question:** The Hospital Management System development team wants to run its application inside a Kubernetes cluster. Create a deployment using Minikube with the application Docker image and verify that the application pod is running successfully. Due to an increase in patient registrations, the Hospital Management System needs multiple running instances. Use Kubernetes to scale the application deployment to four replicas. Hospital staff need to access the Hospital Management System through a web browser. Expose the application deployment using a NodePort service and access it through Minikube. The hospital IT administrator wants to monitor the availability of hospital application servers. Run the Nagios monitoring tool using Docker and access its web dashboard to check host and service status. After deploying the Hospital Management System in Minikube, the application pod repeatedly restarts and becomes unavailable. Identify the cause and troubleshoot the Kubernetes pod to restore application availability.

**One-line explanation:** Deploy and scale the HMS image to 4 pods in Minikube, expose it via NodePort, run Nagios for monitoring, and debug a CrashLoopBackOff pod with `describe`/`logs`.

**Solution:**
1. `minikube start --driver=docker` → `kubectl get nodes` (Ready). Build/load the image: `docker build -t hms-app:latest .` then `minikube image load hms-app:latest`.
2. Create the Deployment: `kubectl create deployment hms-app --image=hms-app:latest --replicas=1`, then `kubectl get pods` (STATUS Running) and `kubectl describe pod <name>`.
3. Scale to four: `kubectl scale deployment hms-app --replicas=4`; verify `kubectl get deployments` shows `4/4`.
4. Expose: `kubectl expose deployment hms-app --type=NodePort --port=8080 --target-port=8080`; open the URL from `minikube service hms-app --url`.
5. Nagios: `docker pull jasonrivers/nagios:latest`, `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest`, open `http://localhost:8888` (`nagiosadmin`/`nagios`) → **Hosts**/**Services**.
6. Troubleshoot restarts: `kubectl get pods` shows `CrashLoopBackOff`; run `kubectl describe pod <name>` (look at **Last State**, exit code, Events) and `kubectl logs <name> --previous`; common causes are wrong image tag, missing env (DB URL), port mismatch, or OOM. Fix the Deployment (`kubectl set image ...` or edit YAML), delete the bad pod, and verify `Running`.

```powershell
minikube start --driver=docker
kubectl get nodes
docker build -t hms-app:latest .
minikube image load hms-app:latest
kubectl create deployment hms-app --image=hms-app:latest
kubectl get pods
kubectl scale deployment hms-app --replicas=4
kubectl get deployments,pods -o wide
kubectl expose deployment hms-app --type=NodePort --port=8080 --target-port=8080
minikube service hms-app --url
# Nagios
docker pull jasonrivers/nagios:latest
docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
# troubleshooting a restarting pod
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl delete pod <pod-name>
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hms-app
spec:
  replicas: 4
  selector:
    matchLabels: { app: hms-app }
  template:
    metadata:
      labels: { app: hms-app }
    spec:
      containers:
        - name: hms-app
          image: hms-app:latest
          ports:
            - containerPort: 8080
```

- **Deploy and verify:** `kubectl create deployment` makes a ReplicaSet and pod; `kubectl get pods` with STATUS `Running` and `kubectl describe pod` (Events: "Pulled", "Created", "Started") confirms success.
- **Scale to four replicas:** `kubectl scale deployment hms-app --replicas=4` changes `spec.replicas`; `kubectl get deployments` then shows `4/4` and four pods appear.
- **NodePort access:** `kubectl expose ... --type=NodePort` creates a Service on a node port 30000–32767; `minikube service hms-app --url` prints the browser URL that reaches the pods.
- **Nagios monitoring:** Run the `jasonrivers/nagios` container on port 8888, open the dashboard with `nagiosadmin`/`nagios`, and read **Hosts** (UP/DOWN) and **Services** (HTTP/load/disk status, OK/WARNING/CRITICAL).
- **Pod repeatedly restarting:** Check `kubectl describe pod` for `CrashLoopBackOff` and exit codes, then `kubectl logs --previous` for the real error (e.g. DB connection refused, wrong image, missing env var). Correct the image/env/command, delete the pod and confirm it comes up Running.

---

### Question 4. Deployment in AWS Environment [25 MARKS]

**Question:** A hospital wants to make its Hospital Management System homepage accessible over the internet. As a DevOps engineer, launch an Ubuntu EC2 instance using a free-tier eligible configuration and deploy the hospital's index.html webpage using Docker and Nginx. The Hospital Management System development team has uploaded its website source code to GitHub. Connect to an Ubuntu EC2 instance, clone the GitHub repository, create a Dockerfile using Nginx and build a Docker image for the hospital website. A hospital has developed its management application as a Maven Web Project and wants to host it on AWS. Deploy the Maven Web Project on an Ubuntu EC2 instance by cloning its GitHub repository, creating a Docker image and running the application inside a Docker container. The Hospital Management System Docker container is running successfully on an AWS EC2 instance, but hospital staff cannot access the website through the public IPv4 address. Identify and configure the appropriate inbound security group rules to make the application accessible. After successfully deploying the Hospital Management System website on AWS EC2, the administrator wants to stop the running Docker container and terminate the EC2 instance to avoid unnecessary resource usage. Perform the required cleanup operations.

**One-line explanation:** Launch free-tier Ubuntu EC2, serve `index.html`/the WAR with Docker, open the right Security Group ports, verify via public IP, then clean up container and instance.

**Solution:**
1. AWS Console → **Services** → **EC2** → **Launch Instance**: name `hms-web`, AMI **Ubuntu Server (Free tier eligible)**, **t2.micro**, create `.pem` key pair, security group with **SSH 22 from My IP** and **HTTP 80 anywhere** (add **8080** for Tomcat), storage 8 GB → **Launch Instance**.
2. Select instance → **Connect** → **SSH client** → copy `ssh -i key.pem ubuntu@<public-ip>`; run from the `.pem` folder (`icacls` on Windows if the key is rejected).
3. Install tools: `sudo apt update && sudo apt-get install -y docker.io git`.
4. For `index.html`: clone the repo, write the Nginx Dockerfile below, `docker build -t hms-site .`, `docker run -d -p 80:80 --name hms-site hms-site`, open `http://<public-ip>`.
5. For the Maven Web Project: use the multi-stage Dockerfile with `maven` + `tomcat:9.0`, build, and run with `-p 8080:8080`.
6. If the site does not load, EC2 → **Security Groups** → **Edit inbound rules** → add **HTTP (80) 0.0.0.0/0** and **Custom TCP 8080 0.0.0.0/0** → **Save rules**, then retry the public IP.
7. Cleanup: `sudo docker stop hms-site && sudo docker rm hms-site`; EC2 console → select instance → **Instance state** → **Terminate instance**.

```powershell
icacls "C:\Users\abhir\aws\key.pem" /inheritance:r
icacls "C:\Users\abhir\aws\key.pem" /grant:r "$($env:USERNAME):(R)"
cd C:\Users\abhir\aws
ssh -i key.pem ubuntu@<public-ip>
```
```bash
# --- inside Ubuntu EC2 ---
sudo apt update && sudo apt-get install -y docker.io git
git clone https://github.com/Kumbhambhargavi75/HospitalManagementSystem.git
cd HospitalManagementSystem
# (a) static index.html on Nginx
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
sudo docker build -t hms-site .
sudo docker run -d --name hms-site -p 80:80 --restart unless-stopped hms-site
# (c) Maven Web Project -> WAR on Tomcat
cat > Dockerfile.web <<'EOF'
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q clean package
FROM tomcat:9.0
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
EOF
sudo docker build -f Dockerfile.web -t hms-web .
sudo docker run -d --name hms-web -p 8080:8080 --restart unless-stopped hms-web
curl http://localhost:80 ; curl http://localhost:8080
# cleanup
sudo docker stop hms-site hms-web && sudo docker rm hms-site hms-web
```

- **Static index.html on Nginx:** Clone the source, write a `FROM nginx:alpine` Dockerfile that `COPY`s `index.html` into `/usr/share/nginx/html`, build and run with `-p 80:80`; open `http://<public-ip>`.
- **Nginx Dockerfile from GitHub:** The `FROM nginx:alpine` snippet above is the Dockerfile; `docker build -t <name> .` creates the image and `docker run -d -p 80:80` serves the page.
- **Maven Web Project on EC2:** Use the multi-stage Dockerfile (`maven` build stage → `tomcat:9.0` runtime stage), `docker build -f Dockerfile.web -t hms-web .`, then `docker run -d -p 8080:8080 hms-web`.
- **Cannot access through public IPv4:** The container is fine but inbound traffic is blocked. Add Security Group inbound rules: **HTTP 80 from 0.0.0.0/0** and **Custom TCP 8080 from 0.0.0.0/0** (SSH stays My IP only), then recheck the public IP.
- **Cleanup:** Stop and remove the containers (`docker stop`/`docker rm`) on the instance, then in the EC2 console choose **Instance state → Terminate instance** so no resources are billed.
