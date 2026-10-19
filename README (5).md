# Jenkins + SonarQube + Docker + Trivy CI/CD Pipeline

A project log of the AWS EC2 CI/CD setup, including architecture, setup
sequence, pipeline flow, issues encountered, resolutions, and next
steps.

> **Current status:** The latest reported image, `my-web-app:7`, had 0
> High/Critical vulnerabilities in Trivy. The deployment command
> returned a container ID. Health checks and rollback are planned for
> the next session.

## Table of contents

1.  [Project overview](#1-project-overview)
2.  [Architecture and flow diagram](#2-architecture-and-flow-diagram)
3.  [Infrastructure](#3-infrastructure)
4.  [Pipeline workflow](#4-pipeline-workflow)
5.  [Setup from scratch](#5-setup-from-scratch)
6.  [Issues and resolutions](#6-issues-and-resolutions)
7.  [Validation commands](#7-validation-commands)
8.  [Security notes](#8-security-notes)
9.  [Next steps](#9-next-steps)

## 1. Project overview

**Goal:** Automate code checkout, code-quality analysis, Quality Gate
enforcement, Docker image build, vulnerability scanning, and website
deployment.

**Repository:** https://github.com/Rajesh-210/Jenkins-Sonarqube-Docker\
**Jenkins job:** `Onix-Digital-CI-CD`\
**Application container:** `my-web-app`\
**Web server:** Nginx\
**Deployment port:** TCP 80

Technologies: AWS EC2, Ubuntu, GitHub, Jenkins, SonarQube, Docker,
Nginx, Trivy, SSH.

## 2. Architecture and flow diagram

### Architecture

``` mermaid
flowchart LR
    Dev[Developer] -->|git push| GH[GitHub repository]
    GH -->|webhook| J[Jenkins EC2]
    J -->|analysis request| SQ[SonarQube EC2]
    SQ -->|Quality Gate result| J
    J -->|SSH and source archive| D[Docker EC2]
    D --> B[Build Docker image]
    B --> T[Trivy scan]
    T -->|Pass| Deploy[Replace running container]
    T -->|Fail| Keep[Fail pipeline and keep old container]
    Deploy --> N[Nginx website on port 80]
    Browser[Browser] --> N
```

### Pipeline flow and decisions

``` mermaid
flowchart TD
    A[GitHub push] --> B[Jenkins Checkout]
    B --> C[SonarQube Analysis]
    C --> D{Quality Gate passes?}
    D -- No --> X[Stop pipeline]
    D -- Yes --> E[Transfer workspace to Docker EC2]
    E --> F[Build image tagged with BUILD_NUMBER]
    F --> G[Trivy scan HIGH and CRITICAL]
    G --> H{Scan passes?}
    H -- No --> Y[Stop; current container remains]
    H -- Yes --> I[Remove old container]
    I --> J[Run new container]
    J --> K[Pipeline success]
```

Trivy runs before the command that removes the existing container. Thus,
a failed vulnerability scan should not remove the current website. A
failure after container removal can still cause downtime; a health check
and rollback are planned.

## 3. Infrastructure

Recorded addresses during the project (verify these before reuse; EC2
public IPs can change):

  -----------------------------------------------------------------------
  Component         Public IP         Private IP        Port
                    recorded                            
  ----------------- ----------------- ----------------- -----------------
  Jenkins EC2       `13.201.33.18`    `172.31.5.251`    `8080`

  SonarQube EC2     `13.234.48.65`    `172.31.11.91`    `9000`

  Docker EC2        `13.206.255.91`   `172.31.6.186`    Website `80`, SSH
  (`dockre`)                                            `22`
  -----------------------------------------------------------------------

Key configuration:

-   GitHub webhook: `http://13.201.33.18:8080/github-webhook/`
-   SonarQube webhook: `http://13.201.33.18:8080/sonarqube-webhook/`
-   Jenkins SonarQube server name: `sonar-server`
-   SonarQube scanner tool name: `sonarqube`
-   Jenkins SSH credential ID: `docker-ssh`
-   Jenkins Sonar token credential ID: `sonar-token`
-   Remote build directory: `/opt/app-ci`
-   Docker image name: `my-web-app`

Use restrictive security-group rules. Allow administrative interfaces
only from trusted sources, and do not expose SSH broadly to the
internet.

## 4. Pipeline workflow

1.  **Checkout:** Jenkins checks out branch `main` from GitHub.
2.  **SonarQube Analysis:** Scanner analyzes the source using project
    key `jenkins-sonarqube-docker`.
3.  **Quality Gate:** `waitForQualityGate abortPipeline: true` blocks
    deployment if the gate fails.
4.  **Remote build:** Jenkins transfers the workspace to Docker EC2 over
    SSH.
5.  **Docker build:** The image is tagged with Jenkins `BUILD_NUMBER`.
6.  **Trivy scan:** The image is scanned for High and Critical
    vulnerabilities. `--exit-code 1` makes the scan fail when matching
    findings exist.
7.  **Deploy:** Only after a successful scan does the pipeline remove
    the previous container and run the new one.

### Trivy command used

``` bash
TMPDIR=/var/tmp/trivy-tmp trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  my-web-app:<BUILD_NUMBER>
```

The scan ran on Docker EC2 because that is where the image is built. The
latest report showed `my-web-app:7`, Alpine 3.24.2, with 0 High/Critical
vulnerabilities.

This is not a claim that the image has no vulnerabilities of any
severity. The shown report had secrets listed as not scanned (`-`); this
configuration is a vulnerability scan.

## 5. Setup from scratch

### Step 1: Create EC2 instances

Create separate Ubuntu EC2 instances for Jenkins, SonarQube, and
Docker/Nginx. Record private and public IP addresses. Configure security
groups for only the required traffic.

### Step 2: Install and verify Jenkins

Install a supported Java version and Jenkins using official instructions
for the selected Ubuntu release. Complete the initial setup and install
required plugins.

``` bash
sudo systemctl status jenkins
java -version
```

Jenkins UI was accessed on port `8080`.

### Step 3: Install and configure SonarQube

Install SonarQube on its dedicated EC2, verify it is healthy and
reachable from Jenkins, create the project, and generate a token. Store
the token in Jenkins Credentials rather than in source code.

### Step 4: Install Docker and Trivy on Docker EC2

Install Docker Engine and Trivy using their official instructions.
Verify:

``` bash
docker --version
docker ps
trivy --version
```

The SSH user needs access to Docker and the remote workspace. The
following commands were used to fix the workspace permission issue:

``` bash
sudo mkdir -p /opt/app-ci
sudo chown ubuntu:ubuntu /opt/app-ci
sudo chmod 755 /opt/app-ci
```

### Step 5: Configure the Dockerfile

The Dockerfile after remediation was:

``` dockerfile
FROM nginx:stable-alpine

RUN apk upgrade --no-cache

COPY . /usr/share/nginx/html/
```

The package upgrade resolved the two High findings in the test image.
Rebuild and scan regularly as base-image packages and vulnerability
databases change.

### Step 6: Configure Jenkins credentials and tools

Configure these Jenkins items:

  Type                               ID/name
  ---------------------------------- ----------------
  SSH private-key credential         `docker-ssh`
  SonarQube secret-text credential   `sonar-token`
  SonarQube server configuration     `sonar-server`
  Sonar Scanner tool installation    `sonarqube`

Set up the pipeline job to use the GitHub repository and branch `main`.
Enable **GitHub hook trigger for GITScm polling**.

### Step 7: Configure webhooks

GitHub webhook:

`http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`

SonarQube webhook:

`http://<JENKINS_PUBLIC_IP>:8080/sonarqube-webhook/`

Replace the placeholder with the current Jenkins address. The webhook
URLs above were used during this project with the recorded Jenkins IP.

### Step 8: Run and verify a build

Push a small change to GitHub. Confirm the Jenkins stages pass in order.
Then verify the container and website with the commands in Section 7.

## 6. Issues and resolutions

  -------------------------------------------------------------------------------------------
  Issue encountered        Why it happened /       Resolution taken
                           symptom                 
  ------------------------ ----------------------- ------------------------------------------
  Git push rejected with   The remote branch       Reconcile with the remote branch before
  `fetch first`            contained commits not   pushing; do not force-push over remote
                           present locally         history

  GitHub push did not      Webhook or job trigger  Configure the GitHub webhook and enable
  trigger Jenkins          was not yet correctly   **GitHub hook trigger for GITScm
                           configured              polling**; a later push triggered a build

  Pipeline needed to       Analysis alone did not  Added a Quality Gate stage using
  enforce code quality     block deployment when   `waitForQualityGate abortPipeline: true`
                           the Quality Gate failed and configured the SonarQube webhook

  SSH could not create     SSH user lacked         Created the directory with `sudo`, then
  `/opt/app-ci`            directory write         assigned ownership to `ubuntu`
                           permission              

  Docker returned          Deployment user lacked  Added the user to the `docker` group and
  permission denied on     access to the Docker    refreshed membership; verified Docker
  `/var/run/docker.sock`   daemon                  commands worked

  Trivy database download  Temporary files were    Created `/var/tmp/trivy-tmp`, assigned it
  failed with disk quota   being written to a      to `ubuntu`, and set
  exceeded                 constrained `/tmp`      `TMPDIR=/var/tmp/trivy-tmp`
                           filesystem              

  Original image showed 64 The original Nginx      Changed the base image to
  vulnerabilities          image was based on      `nginx:stable-alpine` and rebuilt
  including 1 Critical     Debian and reported     
                           many findings           

  Alpine test image still  `libexpat` and `libx11` Added `RUN apk upgrade --no-cache`,
  had 2 High findings      were behind the fixed   rebuilt with `--no-cache`, and rescanned
                           versions shown by Trivy 

  Trivy blocked deployment The scan used           Fixed packages instead of ignoring
  when findings existed    `--exit-code 1` for     findings; subsequent scan showed zero
                           High/Critical findings  High/Critical findings

  Website CSS/appearance   Website was reachable,  Check that static assets are copied to the
  needed verification      but styling had         Nginx document root and validate the live
                           previously needed       page after deploy
                           checking                
  -------------------------------------------------------------------------------------------

### Helpful troubleshooting commands

Jenkins service and logs:

``` bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -n 100 --no-pager
```

Docker and container logs:

``` bash
sudo systemctl status docker
docker ps -a
docker logs --tail 100 my-web-app
```

Git state:

``` bash
git status
git log --oneline -5
git diff --check
```

## 7. Validation commands

Run on Docker EC2 after a successful deployment:

``` bash
docker ps
docker inspect my-web-app
curl -I http://localhost
```

Expected for a healthy static website: an HTTP response such as
`200 OK`. If not, inspect the container logs, port mapping, EC2 security
group, and current public IP.

To rescan a particular image:

``` bash
TMPDIR=/var/tmp/trivy-tmp trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  my-web-app:<BUILD_NUMBER>
```

In Jenkins, inspect **Onix-Digital-CI-CD → latest build → Console
Output**. Confirm Checkout, SonarQube Analysis, Quality Gate, Docker
build, Trivy scan, and deployment completed.

## 8. Security notes

-   Never commit SSH private keys, SonarQube tokens, or passwords.
-   Restrict inbound access to Jenkins, SonarQube, and SSH.
-   The `docker` group grants powerful, effectively root-level
    privileges.
-   `ssh-keyscan` avoids interactive host-key prompts but does not
    authenticate a host by itself. Verify and pin the host-key
    fingerprint for stronger security.
-   Consider HTTPS for webhooks and administrative interfaces.
-   Consider pinning the Nginx image by digest and updating it through a
    controlled process.
-   The current Trivy rule blocks High/Critical vulnerability findings,
    not all possible security issues.
-   Add a separate secrets scan if required by your security policy.
-   Do not remove the old container until the new container has passed a
    health check; add rollback handling.

## 9. Next steps

Planned for the next work session:

-   [ ] Add a post-deployment health check.
-   [ ] Add rollback if the new container fails its health check.
-   [ ] Verify the live site is serving the expected build.
-   [ ] Consider a separate Trivy secrets scan.
-   [ ] Review security-group rules and SSH host-key verification.
-   [ ] Consider base-image digest pinning and a regular update process.

## Final outcome

The project now has a GitHub-triggered Jenkins pipeline with SonarQube
analysis, Quality Gate enforcement, Docker image building, and Trivy
vulnerability scanning before deployment. The latest reported image
`my-web-app:7` showed zero High/Critical vulnerabilities. Health-check
validation and rollback remain planned improvements.
