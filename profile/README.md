<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [🛠 Engineering Journal: Architecture & Deployment Runbook](#-engineering-journal-architecture--deployment-runbook)
  - [**1. The Logical Target Architecture**](#1-the-logical-target-architecture)
  - [**2. Roadmap: Path to Destination**](#2-roadmap-path-to-destination)
  - [**VMWare**](#vmware)
    - [Set the Static IP in Photon OS (After Boot)](#set-the-static-ip-in-photon-os-after-boot)
  - [🖥️ + ☁️ + 🛰️ **CI/CD Pipeline**](#-----cicd-pipeline)
  - [A. 🚀 Continuous Integration (CI) Stage: Local Self-Hosted Workstation Execution](#a--continuous-integration-ci-stage-local-self-hosted-workstation-execution)
  - [B. 🎛️Continuous Deployment (CD) Stage: Cloud Registry to Automated Host Rollout](#b-continuous-deployment-cd-stage-cloud-registry-to-automated-host-rollout)
  - [🚨 Issue 1: Race Condition on DB Initialization (data.sql executing before Hibernate schema creation)](#-issue-1-race-condition-on-db-initialization-datasql-executing-before-hibernate-schema-creation)
  - [🚨 Issue 2: Primary Key Generation Mismatch in In-Memory Database (H2)](#-issue-2-primary-key-generation-mismatch-in-in-memory-database-h2)
  - [🚨 Issue 3: SQL Syntax Errors on Special Characters (Python Decorator Seed)](#-issue-3-sql-syntax-errors-on-special-characters-python-decorator-seed)
  - [🚨 Issue 4: Architectural Leakage (DTO acting as Database Entity)](#-issue-4-architectural-leakage-dto-acting-as-database-entity)
  - [🚨 Issue 5: Infrastructure Transition (Migrating Deployment Target to Consolidated App Host)](#-issue-5-infrastructure-transition-migrating-deployment-target-to-consolidated-app-host)
  - [🚨 Issue 6: GitHub Actions Workflow Failure (Invalid YAML Syntax via Hidden Tab Characters)](#-issue-6-github-actions-workflow-failure-invalid-yaml-syntax-via-hidden-tab-characters)
  - [🚨 Issue 7: Service Registry Environment Provisioning (Photon OS Network & Docker Tailoring)](#-issue-7-service-registry-environment-provisioning-photon-os-network--docker-tailoring)
  - [🚨 Issue 8: CI/CD Pipeline Adaptation (Reconfiguring Deployment Matrix for Service Registry)](#-issue-8-cicd-pipeline-adaptation-reconfiguring-deployment-matrix-for-service-registry)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 🛠 Engineering Journal: Architecture & Deployment Runbook

This project aims to create demo microservices. The original monolithic codebase relied on local ports on a single machine. I levelled up the design to a true production-style distribution across independent VMs! Doing this forces me to handle real-world challenges like network routing, distinct hostnames, and decoupled security layers.
When adding Spring Security to this decoupled topology, the architectural golden rule is to centralize authentication at the API Gateway and use JWT (JSON Web Tokens) to pass the user's identity securely to downstream services (like Feign and Product).

This section documents systemic challenges encountered during the orchestration of our Spring Boot microservices, Docker networking, and local CI/CD runners, along with their permanent resolutions.

Setting up a global documentation repo like this turns a messy troubleshooting session into a production-grade internal corporate wiki.

### **1. The Logical Target Architecture**
   Every VM in our environment has a precise, single responsibility. External requests never hit your Product or Feign VMs directly; they must pass through the edge gateway.

   **Topology Breakdown**

   1. The Client: Sends an HTTP login request to the API Gateway.
   2. API Gateway VM (api-gateway-01): Intercepts the request. It holds the Spring Security configuration, validates login credentials, and issues a signed JWT. For subsequent resource requests, it verifies the incoming token.
   3. Eureka Server VM (eureka-server-01): Acts as the service registry. The Gateway, Feign Client, and Product services all register their current IP/hostname here so they can look each other up dynamically.
   4. Feign Service VM (feign-client-01): A downstream consumer service. When it needs data from the Product service, it asks Eureka for Product's current VM IP and makes a clean, declarative REST call.
   5. Product Service VM (product-service-01): The core resource microservice managing product logic and database persistence.

### **2. Roadmap: Path to Destination**

   To avoid pulling your hair out over networking and authorization bugs at the same time, split your implementation into **four distinct phases**.
   
**Phase 1: Establish the Multi-VM Foundation**
   Before writing any new code, deploy your independent virtual machines in VMware Fusion and get them communicating natively over the subnet (192.168.237.x).
   - Spin up your 4 nodes: eureka-server-01, api-gateway-01, feign-client-01, and product-service-01.
   - Manually update the /etc/hosts file on every machine so they can ping one another by hostname instead of raw IPs.
   
**Phase 2: Deploy and Verify Registration**
   Migrate the code you wrote during the Telusko course onto the individual VMs.
   - Update spring.application.name. Recommendation: Change the dot (.) to a hyphen (-). When other microservices look up or register with Eureka, or when the API Gateway routes load-balanced traffic later using standard service IDs (e.g., lb://service-registry), dot notations can sometimes cause parsing issues or hostname resolution conflicts in Spring Cloud discovery clients. Using a hyphen is the standard convention.
   - Spin up your Eureka Server application first.
   - Deploy your API Gateway, Feign Client, and Product services. Ensure that their application.properties files point explicitly to the remote Eureka VM:
     
```text
spring.application.name=service-registry
server.port=8761

# When your other microservices register with Eureka, it will advertise its location using this hostname
eureka.instance.hostname=eureka-server-01
eureka.client.fetch-registry=false
eureka.client.register-with-eureka=false
```
- Open your browser, navigate to http://192.168.237.130:8761, and verify that all 3 client applications show up green in the Eureka dashboard.

**Phase 3: Implement Routed Communication**
Verify that the services can talk across the physical network barrier.
- Test your API Gateway routing rules. A request to http://<gateway-ip>:<port>/products should successfully jump through the gateway to the Product VM.
- Test Feign. Ensure the Feign client successfully queries Eureka to resolve the Product VM's IP address and pulls back data.

**Phase 4: Layers of Spring Security & JWT**
Now that your pipelines are stable, lock down the environment using Spring Security.

1. Gateway Authenticator: Add spring-boot-starter-security to your API Gateway. Configure a login endpoint that utilizes a database or an in-memory user manager to generate a JWT when authentication succeeds.
2. Gateway Filter: Write a global GatewayFilter that intercepts all incoming API requests to check for a valid Authorization: Bearer <JWT> header.
3. Downstream Token Relay (Feign Interceptor): If a user calls the Feign microservice, Feign must pass that JWT down to the Product service. You will write a RequestInterceptor bean inside your Feign application to automatically copy the JWT from the current incoming request context and inject it into the outgoing OpenFeign call header.
4. Resource Validation: Secure the Product service so that it rejects any incoming traffic that lacks a valid token signature.
---
### **VMWare**

When it comes to virtual virtualization, there is a huge difference between disk storage space and system memory (RAM).

- **Where Spring Boot runs:** Your Java applications run entirely inside the VM's RAM. Since your VMs are already powered on, ESXi has already claimed and allocated all the memory swap files it needs.
- **What your compiled code weighs:** A standard Spring Boot .jar file built from Telusko's tutorials is incredibly lightweight—usually only 30 MB to 60 MB total. Copying that small file onto your VM's internal storage won't even make a dent in your remaining space.
Because a Thick Provisioned disk allocates the entire specified storage block immediately upon creation, ESXi is blocking you because the datastore doesn't have enough raw, unallocated gigabytes available right now.
You can easily bypass this roadblock without resizing your datastore by switching the disk provisioning type to Thin Provisioning.

For a dedicated microservice runner like this, 1. Photon Minimal is exactly what you want. It keeps the OS footprint extremely lightweight, fast-booting, and leaves all available RAM and CPU overhead completely free for your Spring Boot applications.

#### Set the Static IP in Photon OS (After Boot)

Once you are booted up and logged into your new feign-client-01 VM as root, you can configure your static network configuration manually in less than a minute.
Photon OS uses systemd-networkd to handle its interfaces.
1. Open the network configuration file:

```terminaloutput
$ sudo vim /etc/systemd/network/10-static-en.network
[Match]
Name=eth0

[Network]
Address=192.168.237.133/24
Gateway=192.168.237.2
DNS=8.8.8.8
DNS=8.8.4.4

$ sudo systemctl restart systemd-networkd
```

### 🖥️ + ☁️ + 🛰️ **CI/CD Pipeline**

This is the absolute perfect playground for a CI/CD pipeline.

Manually compiling a Java project, opening a terminal, SFTP-ing a .jar file over to Photon OS, killing the old process, and running the new one gets old incredibly fast. Automating this means you just hit git push, and a few seconds later your code is live on your ESXi cluster.

Since your datastore is practically out of room (864 MB left), we can’t install a heavy automation server like Jenkins inside your VMs. Instead, we can use a lightweight cloud-to-local hybrid pipeline that costs you 0 MB of server storage.

The Minimalist CI/CD Blueprint:

```textmate
[ Your IDE ] ---> [ GitHub Repository ] ---> [ GitHub Actions Runner ] ---> [ Photon OS VM ]
 (Code Changes)       (git push main)         (Builds .jar & SSH deploys)     (Runs App)
```
### A. 🚀 Continuous Integration (CI) Stage: Local Self-Hosted Workstation Execution



Instead of consuming standard cloud-hosted runners or overloading your ESXi hypervisor storage layers, this pipeline leverages an on-premise, self-hosted processing strategy:

**The Runner Core Listening Daemon:**
Inside `/Users/amitkapila/central-runner`, a background service script (`svc.sh` running `runsvc.sh` and `run.sh`) maintains an active, long-polling connection between your iMac and the GitHub organization runner group (`settings -> Code, planning, and automation -> Actions -> Runner Groups`).
* When a code commit is pushed, this daemon catches the event payload and initializes an isolated workspace directly inside the internal execution directory: `/Users/amitkapila/central-runner/_work/`.

**Parsing the Pipeline Blueprints:**
The runner reads your exact workflow orchestration file located at your repository root: `.github/workflows/deploy.yml`. This instructs the daemon to execute two key local phases:
* **Artifact Compilation:** The runner triggers the native Maven wrapper script (`./mvnw`) checked out from your source files. It executes the packaging command:
  ```bash
  ./mvnw clean package -DskipTests
  ```
  This compiles the Java source code and compiles a fresh, standalone executable archive inside the local file tier at `target/*.jar`.

**Runner & Dockerfile Interaction:**
Once the JAR is verified, the runner transitions execution over to your local container virtualization layer (Docker Desktop / Colima). It references the context blueprint located at `Dockerfile`:
   
```dockerfile
   FROM eclipse-temurin:17-jdk-alpine
   EXPOSE 8761
   COPY target/*.jar app.jar
   ENTRYPOINT ["java","-jar","/app.jar"]
```

**The Interaction:**
The runner instructs Docker to download the secure, minimal base runtime environment image specified by the `FROM` instruction (`eclipse-temurin:17-jdk-alpine`). It injects your compiled `target/*.jar` into the container framework as `app.jar`, exposing communications across port `8761`.

**Local Visibility:**
Because this build sequence happens natively on your host machine via the self-hosted engine, you can instantly see, inspect, and track these newly baked intermediate images and caching layers inside the **Docker Desktop GUI Application** on your Mac workstation.

**Shipping to the Cloud Registry:**
After assembly, the runner uploads the finalized container image layers over a secure pipeline up to **Docker Hub** (the global, managed Docker registry on the cloud), applying the official tag: `${{ secrets.DOCKERHUB_USERNAME }}/service-registry:latest`.

1. When code is pushed to the `main` branch of the `service-registry` repository, GitHub's central orchestration engine evaluates your organization's designated runner group cluster, which can be viewed under the organisation's `settings -> Code, planning, and automation -> Actions -> Runner Groups`.
2. The platform matches the `runs-on: self-hosted` (within the `service-registry/.github/workflows/deploy.yml`) criteria and delegates the execution load directly to the background runner daemon operating on your Mac workstation.
3. Your local workstation handles the compilation lifecycle safely inside its native workspace:
    * Checks out the latest source files.
    * Provisions a clean `eclipse-temurin` JDK 17 environment.
    * Compiles and packages the application binary artifact:
      ```bash
      ./mvnw clean package -DskipTests
      ```
4. The Mac runner communicates with the active local Docker daemon to build the container layer using your custom multi-stage `Dockerfile`, then pushes the immutable image tag up to your central cloud repository:
   ```bash
   # Built image target tracking
   tags: ${{ secrets.DOCKERHUB_USERNAME }}/service-registry:latest

### B. 🎛️Continuous Deployment (CD) Stage: Cloud Registry to Automated Host Rollout
Once the compilation runner finishes pushing the fresh system layer to Docker Hub, the automated deployment stage initiates to pivot your infrastructure safely:

**Target Infrastructure Path Resolution:**

The self-hosted workstation runner loads your secure deployment context variables, extracting `VM_USER=amit` and `VM_HOST=eureka-server-01` directly from your repository panel (`/settings/variables/actions`).
- It initiates a secure, non-interactive shell command framework (`ssh`) over your network. Your Mac resolves the arbitrary hostname alias `eureka-server-01` using its static local system map inside `/etc/hosts` to route directly to the target environment (`192.168.237.130`).

**Remote Container Lifecyle Management:**

Operating directly inside the remote Photon OS virtual machine shell layer, the pipeline strings a rapid, declarative automation script to update the system engine state:
- **Registry Handshake:** Authenticates the remote machine with the cloud registry by passing down your masked authorization token securely:

```yaml
echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
```

- **Graceful Architecture Pull-Down:** 

Safely halts and purges the active running instance to free up internal system sockets and prevent overlapping naming collisions:

```yaml

docker stop service-registry-container || true
docker rm service-registry-container || true
```

- **Production Image Deployment:**

Commands the host Docker engine to pull down the newly validated image layers straight from the **Docker Hub Cloud Registry** and boot the container as an isolated background system process, routing port mappings flawlessly:

```yaml
docker run -d --name service-registry-container -p 8761:8761 --restart always ${{ secrets.DOCKERHUB_USERNAME }}/service-registry:latest
```

C. **How to Bridge the Network Gap (Crucial Detail)**

- Because your ESXi lab is running on local private IPs (192.168.237.x), GitHub's cloud servers won't be able to see them out of the box. Your laptop will act as the "coordinator" that pulls the code from GitHub, compiles it, and securely pushes the final .jar file directly over your local Wi-Fi to your Photon VMs.
- The Self-Hosted Runner (Recommended for low storage): You can download a tiny, lightweight GitHub Agent script directly onto your laptop. Your laptop acts as the worker bridge. It listens for GitHub commands, runs the build locally, and copies the file across your local VMware network to your Photon servers.
- **Set Up the SSH Target on Photon OS**
       Let the runner deploy files automatically without prompting for passwords. Configure SSH key authentication for non-root user.
       On the Mac, generate an SSH key pair and copy it to the target VM:
```terminaloutput
# Run this on your local laptop terminal
ssh-keygen -t ed25519 -b 4096 -f ~/.ssh/id_rsa_lab

# Copy the public key over to the VM
ssh-copy-id -i ~/.ssh/id_rsa_lab.pub user@<VM_HOST>
```
- **Create Your GitHub Actions Workflow File**
  Inside your Spring Boot project repository on your laptop, create a folder structure named .github/workflows/ and add a configuration file named deploy.yml.
```yaml
name: Build and Deploy Question Service

on:
  push:
    branches: [ "main" ]

env:
  DOCKER_API_VERSION: "1.41"

jobs:
  build-and-push:
    runs-on: self-hosted
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build Project JAR with Maven
      run: ./mvnw clean package -DskipTests

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/question-service:latest

  deploy:
    runs-on: self-hosted
    needs: build-and-push
    steps:
    - name: SSH and Run Application Container
      run: |
        ssh -o StrictHostKeyChecking=no ${{ vars.VM_USER }}@${{ vars.VM_HOST }} << 'EOF'
          # Login to Docker Hub inside the VM
          echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
          
          # Tear down any running instance smoothly
          docker stop question-container || true
          docker rm question-container || true

          # Force the VM to fetch the newest image from Docker Hub
          docker pull ${{ secrets.DOCKERHUB_USERNAME }}/question-service:latest
          
          # Spin up the fresh container on Host Port 8082 mapping cleanly to internal 8080
          docker run -d --name question-container -p 8082:8080 --restart always ${{ secrets.DOCKERHUB_USERNAME }}/question-service:latest
        EOF
```

### 🚨 Issue 1: Race Condition on DB Initialization (data.sql executing before Hibernate schema creation)
* **Symptom:** Microservice crashed on startup with a `Table "QUESTION" not found` error during the data seeding phase, even though `ddl-auto=update` or `create` was active.
* **Root Cause:** In Spring Boot 3.x, script-based data initialization (`data.sql`) runs before Hibernate ORM initializes by default.
* **Fix:** Added the following property to defer data execution until after the JPA schema is safely built:
  ```properties
  spring.jpa.defer-datasource-initialization=true

### 🚨 Issue 2: Primary Key Generation Mismatch in In-Memory Database (H2)
Symptom: Application crashed during script execution with NULL not allowed for column "ID".
Root Cause: The Question entity used GenerationType.AUTO. In Hibernate 6, this defaults to a sequence table generator instead of an identity column runner, requiring explicit IDs in manual INSERT statements.
Fix: Refactored the entity primary key to use a direct identity strategy:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private int id;
```

### 🚨 Issue 3: SQL Syntax Errors on Special Characters (Python Decorator Seed)
Symptom: JdbcSQLSyntaxErrorException: Syntax error in SQL statement... on string fields containing contractions (e.g., function's).
Root Cause: Standard SQL (and H2) does not recognize backslash escapes (\') for strings. It treats the backslash as a literal and cuts the string context early.
Fix: Escaped single quotes using standard SQL duplicate single quotes (''):

```sql
'To define a new variable within a function''s scope'
```

### 🚨 Issue 4: Architectural Leakage (DTO acting as Database Entity)
Symptom: Hibernate automatically generated an unwanted question_wrapper table in the database schema.
Root Cause: The QuestionWrapper class was marked with @Entity, turning a data transport object into a persistence object.
Fix: Removed the @Entity definition completely to preserve architectural boundaries. DTOs are now strictly models for API communication.

---

### 🚨 Issue 5: Infrastructure Transition (Migrating Deployment Target to Consolidated App Host)
* **Symptom:** Updates pushed to the microservice repository needed to cleanly target a newly designated, generic application host (`apps-01`) instead of an isolated, service-specific VM alias.
* **Root Cause:** Hardcoded CI/CD variables and local resolution configurations tied deployment execution and testing tools to specific single-purpose hostnames (`question-service-01`).
* **Fix:** Abstracted the network footprint across both local development and remote deployment stages:
  1. Updated the environment-wide `VM_HOST` runner variable under the GitHub Actions secrets configuration panel (`/settings/variables/actions`).
  2. Consolidated local DNS mappings inside the Mac workstation `/etc/hosts` file to resolve the new unified app tier smoothly:
```text
     192.168.237.133   apps-01
```

---

### 🚨 Issue 6: GitHub Actions Workflow Failure (Invalid YAML Syntax via Hidden Tab Characters)
* **Symptom:** Pushing pipeline updates resulted in an immediate workflow rejection under GitHub Actions with the error: `Invalid workflow file: You have an error in your yaml syntax on line 44`.
* **Root Cause:** While introducing the deployment `docker pull` sequence, a hidden `Tab` character or irregular indentation block crept into the text block. Because the YAML specification strictly forbids tab characters for nested indentation, the GitHub Actions parser failed to compile the workflow structure.
* **Fix:** Utilized native terminal debugging utilities to audit and correct the formatting locally on the Mac runner before pushing:
  1. Exited standard text mode in `vim` and executed the following commands to instantly expose invisible spacing, layout endings, and tab configurations (`^I` sequences):
     ```vim
     :set list
     :set listchars=tab:▸\ ,trail:·
     ```
  2. Created a local pre-flight automation validation step using Python's native parsing framework to ensure structural integrity prior to standard Git commits:
     ```bash
     python3 -c "import yaml, sys; yaml.safe_load(sys.stdin)" < .github/workflows/deploy.yml
     ```

---

### 🚨 Issue 7: Service Registry Environment Provisioning (Photon OS Network & Docker Tailoring)
* **Symptom:** The newly provisioned host `eureka-server-01` lacked an operational container runtime environment, rejected non-root docker operations, and dropped outbound packet verification to public registries.
* **Root Cause:** 1. The OS runtime environment was identified as VMware Photon OS (utilizing `tdnf` instead of `apt-get`), where the pre-installed Docker daemon was dormant.
  2. The user account lacked an active, refreshed group association session to target the `/var/run/docker.sock` communication layer.
  3. Dynamic network configurations (`50-dhcp-en.network`) conflicted with lab routing architecture requirements.
* **Fix:** Structured the OS administration layer, authorized access privileges, and applied a declarative static network profile:
  1. **Activated & Persisted the Daemon:** Enabled and initiated the local systemd daemon layout management tier:
     ```bash
     sudo systemctl start docker
     sudo systemctl enable docker
     sudo usermod -aG docker amit
     # Note: Log out and log back in to apply group privileges cleanly
     ```
  2. **Configured Static Network Profile:** Transformed the configuration fallback structure from dynamic DHCP over to an immutable network layout profile inside `systemd-networkd`:
     ```bash
     sudo mv /etc/systemd/network/50-dhcp-en.network /etc/systemd/network/10-static-en.network
     ```
  3. **Declared System Networking Footprint:** Populated `/etc/systemd/network/10-static-en.network` with the designated lab IP framework:
     ```text
     [Match]
     Name=eth0

     [Network]
     Address=192.168.237.130/24
     Gateway=192.168.237.2
     DNS=8.8.8.8
     DNS=8.8.4.4
     ```
---

---

### 🚨 Issue 8: CI/CD Pipeline Adaptation (Reconfiguring Deployment Matrix for Service Registry)
* **Symptom:** The infrastructure needed an automated compilation and deployment sequence to provision the Eureka Server container smoothly onto the newly isolated `eureka-server-01` node without manual host interventions.
* **Root Cause:** The existing template was strictly configured to target the `question-service` lifecycle parameters, which bounded image tags, runtime identifiers, container namespaces, and host routing definitions (`8082->8080`) to the functional domain layer.
* **Fix:** Adapted the pipeline design patterns in `/Users/amitkapila/dev/java/Telusko/service-registry/.github/workflows/deploy.yml` to reflect infrastructure requirements:
    1. **Refactored Artifact Target Tracking:** Retargeted the Docker build and pull mechanisms to reference the service infrastructure namespace:
       ```yaml
       tags: ${{ secrets.DOCKERHUB_USERNAME }}/service-registry:latest
       ```
    2. **Isolated Container Lifetime Management:** Swapped out the runtime context references to isolate and prevent overlapping container management errors:
       ```bash
       docker stop service-registry-container || true
       docker rm service-registry-container || true
       ```
    3. **Aligned Infrastructure Routing Topology:** Adjusted the host-to-container network bridge configuration mappings to expose the Eureka web dash interface uniformly on its native backbone port (`8761:8761` instead of app-tier defaults):
       ```yaml
       docker run -d --name service-registry-container -p 8761:8761 --restart always ${{ secrets.DOCKERHUB_USERNAME }}/service-registry:latest
       ```
