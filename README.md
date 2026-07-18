### Full-Stack Blog App

**Status: ✅ Complete** — the CI/CD pipeline is live and triggers automatically on every push to `main`.

```
Job Scenario:
Setting up a CI/CD pipeline for a full-stack blogging (Twitter-style) app hosted on GitHub.
The pipeline uses Jenkins for build automation, SonarQube for static code analysis, and
Docker for containerizing and publishing the application to Docker Hub.
```

---

### Project Structure

```
/full-stack-blogging-app
├── EKS_Terraform/          # Terraform IaC for an EKS cluster (provisioned manually, not yet wired into Jenkins)
│   ├── main.tf
│   ├── variables.tf
│   └── output.tf
├── src/                     # Application source (Spring Boot, Java 21)
├── Dockerfile               # Container build definition for the app
├── deployment-service.yml   # Kubernetes Deployment + Service manifest
├── RBAC.md                  # Notes on Kubernetes RBAC setup for deployments
├── install_docker.sh        # Docker Engine install script (Ubuntu)
├── Jenkinsfile              # Main CI/CD pipeline definition
├── pom.xml                  # Maven build config (Java 21)
└── README.md
```

---

## Project Overview
This project demonstrates a complete CI/CD pipeline for a full-stack blogging application, using **Jenkins**, **SonarQube**, and **Docker**. A GitHub webhook triggers Jenkins automatically on every push; Jenkins then pulls the latest code, compiles and packages it with Maven, runs static analysis through SonarQube, and builds and pushes a Docker image to Docker Hub — ready to run anywhere.

## Key Technologies & Tools
- **Java 21** — application runtime and language version.
- **Maven** — build tool (compile, package).
- **Jenkins** — CI/CD orchestration.
- **SonarQube** — static code analysis for quality and security checks.
- **Docker** — containerizes the application and publishes it to Docker Hub.
- **Terraform / EKS** (`EKS_Terraform/`) — infrastructure-as-code for provisioning an EKS cluster; currently provisioned separately from the Jenkins pipeline.

## CI/CD Pipeline (`Jenkinsfile`)

The pipeline currently runs these stages, in order:

1. **Git Checkout** — pulls the `main` branch from the GitHub repo.
2. **Compile** — `mvn compile`, using the configured JDK/Maven tools.
3. **SonarQube Analysis** — runs `sonar-scanner` against the configured SonarQube server.
4. **Build** — `mvn package`, produces the Spring Boot jar.
5. **Docker Build & Tag** — builds the app image (`naveenreddy9/devops_30`) using Docker Hub credentials.
6. **Docker Push Image** — pushes the built image to Docker Hub.

> Kubernetes deployment (`deployment-service.yml`) and EKS provisioning (`EKS_Terraform/`) exist in the repo but are **not yet automated** in the Jenkinsfile — they're applied manually via `kubectl`/`terraform` for now.

## CI/CD Trigger (GitHub Webhook)
The pipeline runs automatically on every push to `main` — no manual trigger needed.

- **Jenkins job → Configure → Build Triggers**: `GitHub hook trigger for GITScm polling` is enabled.
- **GitHub repo → Settings → Webhooks**: a webhook posts to `http://<jenkins-host>:8080/github-webhook/` (Content type: `application/json`, event: `push`).
- Check delivery status any time under the webhook's **Recent Deliveries** tab in GitHub.

## EC2 Host Setup (Ubuntu)
The Jenkins server is a plain Ubuntu EC2 instance. These are the actual scripts used to provision it.

### 1. Install Jenkins (`jenkinsinstallation.sh`)
```bash
#!/bin/bash

# Update system packages
sudo apt-get update -y

# Install Java (Jenkins requires Java to run)
sudo apt-get install -y openjdk-11-jdk

# Import Jenkins GPG key and add Jenkins apt repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package lists to include Jenkins repository
sudo apt-get update -y

# Install Jenkins
sudo apt-get install -y jenkins

# Start Jenkins
sudo systemctl start jenkins

# Enable Jenkins to start at boot
sudo systemctl enable jenkins

echo "Jenkins installed successfully! Access it at http://<your-server-ip>:8080"
echo "Retrieve the initial admin password with: sudo cat /var/lib/jenkins/secrets/initialAdminPassword"
```
> `openjdk-11-jdk` here is only the Java version Jenkins itself runs on — it's unrelated to the JDK the pipeline compiles the app with. The app's own build JDK (`JDK 21`) is configured separately in Jenkins under **Manage Jenkins → Tools** (see below).

### 2. Install Docker (`dockerinstall.sh`)
```bash
#!/bin/bash

# Update existing list of packages
sudo apt-get update

# Install prerequisite packages
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker official GPG key
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the Docker stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update the package index again
sudo apt-get update

# Install Docker Engine, CLI, and containerd
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker
sudo systemctl start docker

# Enable Docker to start on boot
sudo systemctl enable docker

# Verify Docker installation
sudo docker --version

# Add current user to the Docker group to avoid using sudo (optional)
sudo usermod -aG docker $USER

echo "Docker installation completed. Please log out and log back in to apply the group changes."
```
> Jenkins itself also runs `docker` commands (in the pipeline's Docker Build/Push stages) as the `jenkins` user — that user needs to be in the `docker` group too: `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`.

### 3. Install SonarQube (`sonarqubeinstall.sh`)
SonarQube runs as a Docker container on the same host. Its bundled Elasticsearch needs a higher `vm.max_map_count` than the Linux default, or the container exits immediately on startup.
```bash
#!/bin/bash

# SonarQube's embedded Elasticsearch requires these, or the container
# fails at startup with "max virtual memory areas vm.max_map_count is too low"
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=131072

# Persist across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf

# Run SonarQube (Community Edition)
docker run -d --name sonarqube --restart unless-stopped -p 9000:9000 sonarqube:lts-community

echo "SonarQube starting up. Access it at http://<server-ip>:9000"
echo "Default login: admin / admin (you'll be prompted to change it on first login)"
```
After logging in, generate a token under **My Account → Security → Generate Tokens** — this is what gets pasted into the Jenkins SonarQube credential (step 2 below).

> Startup takes 30–60 seconds. If `http://<server-ip>:9000` doesn't load right away, check `docker logs sonarqube` — an `AccessDeniedException` or Elasticsearch crash there almost always means the `vm.max_map_count` step above was skipped.

`--restart unless-stopped` makes Docker bring the container back automatically after a crash or host reboot. If it's ever down and needs a manual nudge:
```bash
docker ps -a | grep sonarqube   # check its current status
docker start sonarqube          # start it back up (container already exists, no need to `run` again)
```

## Jenkins Setup

### 1. Configure Tools (**Manage Jenkins → Tools**)
These names must match exactly what the `Jenkinsfile` references (Jenkins tool names are case-sensitive):

| Tool | Name used in Jenkinsfile | Notes |
|---|---|---|
| JDK | `JDK 21` | Either auto-installed (Install automatically → Add Installer → *Install from adoptium.net* → pick a JDK 21 build), or point `JAVA_HOME` at an existing install, e.g. `/usr/lib/jvm/java-21-openjdk-amd64`. |
| Maven | `Maven` | Install automatically → *Install from Apache* → pick a 3.9.x version. |
| SonarQube Scanner | `sonar-scanner` | Referenced via `tool 'sonar-scanner'` in the `environment` block; install automatically from the SonarQube Scanner installer. |

### 2. Configure the SonarQube server (**Manage Jenkins → System → SonarQube servers**)
- **Name**: must match the string passed to `withSonarQubeEnv(...)` in the Jenkinsfile (currently `sonarqubeserver`).
- **Server URL**: your SonarQube instance URL, e.g. `http://localhost:9000` if SonarQube runs on the same host as Jenkins.
- **Server authentication token**: a Secret Text credential holding the SonarQube token generated in step 3 above (*My Account → Security → Generate Tokens*).

### 3. Configure credentials (**Manage Jenkins → Credentials**)
| Credential ID | Kind | Used for |
|---|---|---|
| `dockertoken` | Username with password (Docker Hub username + access token) | `withDockerRegistry` in the Docker Build/Push stages |
| SonarQube token credential | Secret text | Bound to the SonarQube server config above |

## Steps to Reproduce
```bash
git clone https://github.com/VenkataNaveenReddyYaparla/DevOps_Project_30.git
cd DevOps_Project_30
```
1. Provision a Jenkins host and install Jenkins + Docker (see above).
2. Configure the Tools, SonarQube server, and Credentials as described above.
3. Create a Jenkins Pipeline job pointing at this repo's `Jenkinsfile`.
4. Run the pipeline — it checks out, compiles, analyzes, packages, and publishes the Docker image.

## Running the App Manually
After the pipeline pushes the image to Docker Hub, you can run it standalone on any Docker host:
```bash
docker pull naveenreddy9/devops_30:latest
docker run -d --name devops30 -p 8880:8080 naveenreddy9/devops_30:latest
```
Then open `http://<host-ip>:8880/register` to create an account, and log in at `http://<host-ip>:8880/login` with those credentials.

- The app uses an **in-memory H2 database** (`spring.datasource.url=jdbc:h2:mem:...`) — there is no seeded/default user, and all data (including registered accounts) is wiped whenever the container restarts.
- Remember to open the chosen host port (e.g. `8880`) in the EC2 **Security Group** if accessing from outside the instance.
- If a container with the same name already exists, remove it first: `docker rm -f devops30`.

## Troubleshooting
Issues hit while setting this pipeline up, and how they were resolved:

| Symptom | Cause | Fix |
|---|---|---|
| `Tool type "jdk"/"maven" does not have an install ... configured` | Jenkinsfile tool names didn't match the names configured under Manage Jenkins → Tools | Make the `tools { jdk ...; maven ... }` names in the Jenkinsfile match the Tools page exactly (case-sensitive) |
| `error: release version 17 not supported` | JDK tool had "Install automatically" checked but no installer configured, so Jenkins fell back to an older system Java | Add an installer (Adoptium/Temurin 21) under the JDK tool, or set `JAVA_HOME` to a real JDK 21 path on the host |
| `... doesn't look like a JDK directory` | Manually-entered `JAVA_HOME` pointed at a non-existent or unreadable path | Confirm the exact path via `ls /usr/lib/jvm/` on the host, or switch to auto-install instead of a manual path |
| `SonarQube installation ... does not match any configured installation` | Name in `withSonarQubeEnv(...)` didn't match the SonarQube server name under Manage Jenkins → System | Make both names match exactly |
| `Not authorized. Please check sonar.login and sonar.password` | The SonarQube token credential was invalid/expired | Generate a new token in SonarQube (My Account → Security) and update the Jenkins credential |
| `Could not find credentials matching dockertoken` | No credential with ID `dockertoken` existed (or was deleted/renamed) | Create a **Username with password** credential — username = Docker Hub username, password = Docker Hub access token — with ID `dockertoken` |
| `permission denied ... unix:///var/run/docker.sock` | The user running `docker` (e.g. `jenkins`, or an interactive shell user) isn't in the `docker` group | `sudo usermod -aG docker <user>`, then restart the service (`sudo systemctl restart jenkins`) or start a new shell session (`newgrp docker`) |
| Root filesystem near full after increasing the EBS volume size | Resizing the EBS volume in AWS doesn't automatically resize the OS partition/filesystem | `sudo growpart /dev/nvme0n1 1` then `sudo resize2fs /dev/nvme0n1p1` (device names vary — check with `lsblk`) |

**Never commit tokens/passwords into files tracked by git.** `installation_notes.txt` is intentionally untracked (see `.gitignore`) since it's used as a personal scratch file for install notes.

## Contact
Maintained by Venkata Naveen Reddy Yaparla.
