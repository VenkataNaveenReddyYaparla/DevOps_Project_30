### Full-Stack Blog App
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
This project demonstrates a CI/CD pipeline for a full-stack blogging application, using **Jenkins**, **SonarQube**, and **Docker**. Jenkins pulls the latest code, compiles and packages it with Maven, runs static analysis through SonarQube, then builds and pushes a Docker image to Docker Hub.

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

## Jenkins Setup

### 1. Install Jenkins & Docker on the host
- Jenkins (RHEL/Amazon Linux, via `pkg.jenkins.io` repo) with `java-21-openjdk` as a prerequisite.
- Docker Engine on Ubuntu via `install_docker.sh` (adds the official Docker apt repo, installs `docker-ce`, enables the service, adds the Jenkins/deploy user to the `docker` group).

### 2. Configure Tools (**Manage Jenkins → Tools**)
These names must match exactly what the `Jenkinsfile` references (Jenkins tool names are case-sensitive):

| Tool | Name used in Jenkinsfile | Notes |
|---|---|---|
| JDK | `JDK 21` | Either auto-installed (Install automatically → Add Installer → *Install from adoptium.net* → pick a JDK 21 build), or point `JAVA_HOME` at an existing install, e.g. `/usr/lib/jvm/java-21-openjdk-amd64`. |
| Maven | `Maven` | Install automatically → *Install from Apache* → pick a 3.9.x version. |
| SonarQube Scanner | `sonar-scanner` | Referenced via `tool 'sonar-scanner'` in the `environment` block; install automatically from the SonarQube Scanner installer. |

### 3. Configure the SonarQube server (**Manage Jenkins → System → SonarQube servers**)
- **Name**: must match the string passed to `withSonarQubeEnv(...)` in the Jenkinsfile (currently `sonarqubeserver`).
- **Server URL**: your SonarQube instance URL.
- **Server authentication token**: a Secret Text credential holding a SonarQube user/analysis token (generate under SonarQube → *My Account → Security → Generate Tokens*).

### 4. Configure credentials (**Manage Jenkins → Credentials**)
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

## SonarQube Server
For local/dev setups, SonarQube itself can run as a container on the same Jenkins host:
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
```
Then point the Jenkins SonarQube server config at `http://<host-ip>:9000` (or `http://localhost:9000` if Jenkins and SonarQube run on the same box).

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
