# CI/CD Pipeline with GitHub, Jenkins, SonarQube & Docker

Automate your software delivery pipeline—from code commit to deployment—using GitHub, Jenkins, SonarQube, and Docker.

---

##  Project Overview

1. Code is version-controlled on **GitHub**.
2. A **GitHub webhook** triggers a **Jenkins** build whenever code is pushed.
3. **SonarQube** performs static code analysis to maintain quality and catch vulnerabilities.
4. **Docker** builds and runs your application in a container.
5. The app is deployed and made accessible via a web endpoint.

---

## Architecture Diagram

[ GitHub ] → Webhook → [ Jenkins ] → SonarQube Analysis → Docker Build → Deployment

---

## Prerequisites

- An Ubuntu-based server or VM for running services (separate instances or containers recommended).
- GitHub repository containing your project code.
- Docker installed on the deployment server.
- Access to a browser or network port for Jenkins, SonarQube, and your app.

---

## Step-by-Step Setup

### 1. Provision Servers
Create three Ubuntu servers for:
- **Jenkins** (open port `8080`)
- **SonarQube** (open port `9000`)
- **Docker deployment** (open port where the app will run, e.g., `8085`)

### 2. Install Jenkins
```bash
sudo hostnamectl set-hostname jenkins
sudo apt update
sudo apt install openjdk-11-jre -y

# Add Jenkins repo and install
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update
sudo apt install jenkins -y

sudo systemctl start jenkins
sudo systemctl enable jenkins
```
Access Jenkins at `http://<JENKINS_IP>:8080`, initialize using the admin password from:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 3. Set Up SonarQube
```bash
sudo hostnamectl set-hostname sonarqube
sudo apt update
sudo apt install openjdk-17-jre -y
# Download and install SonarQube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
sudo apt install unzip
unzip sonarqube-*.zip
cd sonarqube-*/bin/linux-x86-64
./sonar.sh start
```
Open SonarQube at `http://<SONAR_IP>:9000` and log in (default: `admin`/`admin`)

### 4. Configure Jenkins Plugins
Install these plugins via **Manage Jenkins → Manage Plugins**:
- GitHub integration, Pipeline, SonarQube Scanner, Docker Pipeline, SSH build agents

Then configure:
- **Global Tool Configuration** – add SonarQube Scanner
- **Configure System** – add SonarQube server URL and token

Generate a SonarQube token via **My Account → Security**

### 5. Add GitHub Webhook
In your GitHub repo settings, add a webhook pointing to:
```
http://<JENKINS_IP>:8080/github-webhook/
```
Enable "GitHub hook trigger for GITScm polling" in Jenkins job configuration

### 6. Jenkins Project Configuration
Create a **Freestyle Project**:
- Set Git repo URL and branch (e.g., main).
- Set environment variables & credentials.
- Add build steps:
  1. **Run SonarQube analysis**
  2. **Build Docker image**
  3. **Deploy application** via remote shell or direct Docker run.

#### Example Execute Shell Script
```bash
# Copy files to web folder
cp -r /var/lib/jenkins/workspace/Automated-pipeline/* /home/ubuntu/web

# Change to target directory
cd /home/ubuntu/web

# Build Docker image
docker build -t website .

# Run container on port 8081
docker run -d -p 8081:80 --name website-container website
```

### 7. Final Workflow

- Push code → GitHub triggers Jenkins
- Jenkins:
  - Pulls code
  - Runs SonarQube analysis
  - Builds and deploys Docker container
- App is accessible via the deployment host (e.g., `http://<DEPLOY_IP>:8081`)

---

## Summary Table

| Component     | Purpose                          | Access/Port        |
|---------------|----------------------------------|--------------------|
| GitHub        | Code source and trigger          | GitHub interface   |
| Jenkins       | CI/CD orchestration              | 8080               |
| SonarQube     | Code quality analysis            | 9000               |
| Docker Host   | Runs the application container   | e.g., 8081         |

---

## Notes & Best Practices

- Ensure firewall/security groups allow only necessary access (e.g., IP-restricted access).
- Validate your `sshd_config` before restarting SSH to prevent lockouts.
- Use non-sudo Docker access by adding `ubuntu` to the `docker` group:  
  `sudo usermod -aG docker ubuntu`
- Consider enhancing with multi-branch pipelines or automated rollback later.

---

### 🛡️ Fixing Docker Permission Issues for Jenkins

By default, the `jenkins` user cannot access `/var/run/docker.sock`.  
To grant Jenkins permission to run Docker commands without `sudo`, run:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo systemctl restart docker
```

> **Note:** After running these commands, you may need to log out and back in, or restart the system, for changes to take effect.
