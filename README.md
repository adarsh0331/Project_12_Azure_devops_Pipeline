<img width="2829" height="1500" alt="diagram-export-9-30-2025-6_27_38-PM" src="https://github.com/user-attachments/assets/a0669b87-2cc4-4532-8d69-6f5a7bad5aca" />


# CI/CD Pipeline for Java Application

This document outlines the Continuous Integration and Continuous Deployment (CI/CD) pipeline for a Java application, implemented using Azure DevOps, with integrations for Maven, SonarQube, Docker, and Trivy. The pipeline automates the build, test, code quality analysis, containerization, vulnerability scanning, and deployment to a self-hosted virtual machine (VM).



## Table of Contents

- [Overview](#overview)  
- [Infrastructure Setup](#infrastructure-setup)  
- [Tools and Services](#tools-and-services)  
- [Pipeline Flow](#pipeline-flow)  
- [Pipeline Stages](#pipeline-stages)  
  - [Build & Test](#build--test)  
  - [SonarQube Analysis](#sonarqube-analysis)  
  - [Docker Build & Push](#docker-build--push)  
  - [Trivy Vulnerability Scan](#trivy-vulnerability-scan)  
  - [Deployment](#deployment)  
- [Key YAML Snippets](#key-yaml-snippets)  
- [Example Dockerfile](#example-dockerfile)  
- [Best Practices](#best-practices)  
- [Prerequisites](#prerequisites)  
- [Setup Instructions](#setup-instructions)  
  - [Creating a Self-Hosted Agent in Azure](#creating-a-self-hosted-agent-in-azure)  
  - [Creating Docker Hub Service Connection](#creating-docker-hub-service-connection)  
  - [Creating SonarQube Service Connection](#creating-sonarqube-service-connection)  
  - [Additional Setup Steps](#additional-setup-steps)  
- [Troubleshooting](#troubleshooting)  
- [Contributing](#contributing)  



## Overview

The CI/CD pipeline automates the process of building, testing, analyzing, and deploying a Java application. Key features include:

- **Automated Builds:** Triggered on code pushes to the main branch or pull requests.  
- **Code Quality:** Static analysis with SonarQube to enforce quality gates.  
- **Containerization:** Docker image creation and storage in Docker Hub.  
- **Security:** Vulnerability scanning with Trivy to ensure secure images.  
- **Deployment:** Deployment to a self-hosted Ubuntu VM running the application in a Docker container.  

The pipeline ensures high code quality, security, and traceability, with modular stages for maintainability.



## Infrastructure Setup

The pipeline deploys to a self-hosted virtual machine with the following specifications:

- **VM Type:** Azure Standard B2s (2 vCPUs, 4 GiB memory)  
- **Operating System:** Ubuntu 20.04 LTS  
- **Ports Open:**  
  - 22 (SSH): For remote access and pipeline agent communication.  
  - 8080 (Application): For the Java application running in a Docker container.  
  - 9000 (SonarQube): For accessing the SonarQube dashboard.  

**Networking:**  
- Default Virtual Network (VNet) and Network Security Group (NSG) configured in Azure.  
- NSG rules allow inbound traffic on ports 22, 8080, and 9000 from trusted IP ranges.  

**Self-Hosted Agent:**  
- Azure DevOps agent installed on the VM to execute pipeline tasks.  



## Tools and Services

- **Azure DevOps:** CI/CD orchestration platform for pipeline execution and repository management.  
- **Maven:** Build tool for compiling, testing, and packaging the Java application.  
- **SonarQube:** Static code analysis tool for code quality and security checks.  
- **Docker:** Containerization platform for building and deploying the application.  
- **Trivy:** Vulnerability scanner for Docker images to detect security issues.  
- **Docker Hub:** Registry for storing and retrieving Docker images.  



## Pipeline Flow

The pipeline is triggered on code pushes to the main branch or pull requests. It progresses through the following stages:

1. **Build & Test:** Compiles the Java code, runs unit tests, and publishes test results.  
2. **SonarQube Analysis:** Performs static code analysis and enforces quality gates.  
3. **Docker Build & Push:** Builds a Docker image from the compiled JAR and pushes it to Docker Hub.  
4. **Trivy Vulnerability Scan:** Scans the Docker image for vulnerabilities, failing the pipeline if critical issues are found.  
5. **Deployment:** Pulls the Docker image to the self-hosted VM and runs the container.  



## Pipeline Stages

### Build & Test

- **Trigger:** Push to main branch or pull requests.  
- **Tasks:**  
  - Maven compiles the Java code using the `pom.xml` configuration.  
  - Runs unit tests with JUnit.  
  - Publishes JUnit test results to Azure DevOps for visibility.  

**Output:** A compiled JAR file stored as a pipeline artifact.  



### SonarQube Analysis

- **Purpose:** Ensures code quality and security through static analysis.  
- **Tasks:**  
  - Runs SonarQube scanner to analyze code for bugs, vulnerabilities, and code smells.  
  - Enforces quality gates (e.g., no critical issues, sufficient test coverage).  
  - Fails the pipeline if quality gates are not met.  

**Output:** Analysis results available in the SonarQube dashboard.  



### Docker Build & Push

- **Purpose:** Containerizes the application for consistent deployment.  
- **Tasks:**  
  - Builds a Docker image using a Dockerfile that includes the compiled JAR.  
  - Tags the image with the build ID for traceability.  
  - Pushes the image to Docker Hub using a service connection.  

**Output:** Docker image stored in Docker Hub.  



### Trivy Vulnerability Scan

- **Purpose:** Ensures the Docker image is free of critical vulnerabilities.  
- **Tasks:**  
  - Scans the Docker image using Trivy for known vulnerabilities in dependencies and base images.  
  - Configured to fail the pipeline if critical or high-severity vulnerabilities are detected.  

**Output:** Scan report published as a pipeline artifact.  



### Deployment

- **Purpose:** Deploys the application to the self-hosted VM.  
- **Tasks:**  
  - Uses SSH to connect to the VM.  
  - Pulls the latest Docker image from Docker Hub.  
  - Runs a new container from the pulled image, mapping port 8080.  

**Output:** Application running on the VM, accessible via port 8080.  



## Key YAML Snippets

```yaml
# Pipeline Trigger

trigger:
  branches:
    include:
      - main

pool:
  name: 'Test-Runner'

variables:
  buildConfiguration: 'Release'
  dockerRegistryServiceConnection: 'docker-service-connection'   # Azure DevOps Docker service connection name
  imageRepository: 'adarshbarkunta/myapp'
  containerRegistry: 'docker-service-connection'   # Replace with your ACR/Docker Hub registry
  dockerfilePath: 'Dockerfile'

  tag: '$(Build.BuildId)'

stages:
  # =========================
  # 1. Build & Test with Maven
  # =========================
  - stage: Build
    displayName: "Build and Test"
    jobs:
      - job: MavenBuild
        displayName: "Maven Compile & Test"
        steps:
          - task: Maven@4
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'clean compile test'
              publishJUnitResults: true
              testResultsFiles: 'target/surefire-reports/*.xml'

  # =========================
  # 2. SonarQube Analysis
  # =========================    
  - stage: SonarQube
    displayName: "Code Quality Scan"
    jobs:
      - job: SonarScan
        displayName: "SonarQube Analysis"
        steps:
          - task: SonarQubePrepare@6
            inputs:
              SonarQube: 'SonarQubeServiceConnection'   # Service connection in Azure DevOps
              scannerMode: 'Other'
              configMode: 'manual'
              extraProperties: |
                sonar.projectKey=myapp
                sonar.projectName=myapp
                # sonar.sources=java-cicd-project/spring-boot-app/src/main/java
                # sonar.tests=java-cicd-project/spring-boot-app/src/test/java
                # sonar.java.binaries=java-cicd-project/spring-boot-app/target/classes

          - task: Maven@4
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'clean verify'
              publishJUnitResults: true
              testResultsFiles: 'target/surefire-reports/*.xml'

          - task: SonarQubeAnalyze@6
            displayName: "Run SonarQube Analysis"

          - task: SonarQubePublish@6
            inputs:
              pollingTimeoutSec: '300'

          - task: Maven@4
            displayName: "Maven Build JAR for Docker"
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'package -DskipTests'
          - script: |
              echo "Listing target folder contents:"
              ls -l target/
            displayName: "Show built JAR"

  # =========================
  # 4. Build & Push Docker Image
  # =========================
  - stage: DockerBuild
    displayName: "Build & Push Docker Image"
    jobs:
      - job: Docker
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: '$(dockerRegistryServiceConnection)'
              repository: '$(imageRepository)'
              command: 'buildAndPush'
              dockerfile: '$(dockerfilePath)'
              tags: |
                $(tag)

  # =========================
  # 5. Trivy Scan
  # =========================
  - stage: Trivy
    displayName: "Trivy Vulnerability Scan"
    dependsOn: DockerBuild
    jobs:
      - job: Scan
        steps:
          - script: |
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
              sudo mv ./bin/trivy /usr/local/bin/trivy
              trivy --version
              trivy image $(imageRepository):$(tag) --exit-code 1 --severity HIGH,CRITICAL
            displayName: "Run Trivy Scan"

  # =========================
  # 6. Deploy
  # =========================
  - stage: Deploy
    displayName: "Deploy Application"
    dependsOn: Trivy
    condition: succeeded()
    jobs:
      - job: DeployApp
        steps:
          - script: |
              echo "Pulling image from registry..."
              docker pull $(imageRepository):$(tag)

              echo "Starting new container..."
              docker run -d --name myapp -p 8080:8080 $(imageRepository):$(tag)
            displayName: "Deploy to Container Service"
````



## Example Dockerfile

```dockerfile
##artifact build stage
FROM maven AS buildstage
RUN mkdir /opt/mindcircuit13
WORKDIR /opt/mindcircuit13
COPY . .
RUN mvn clean install    ## artifact -- .war

### tomcat deploy stage
FROM tomcat
WORKDIR webapps
COPY --from=buildstage /opt/mindcircuit13/target/*.war .
RUN rm -rf ROOT && mv *.war ROOT.war
EXPOSE 8080
```



## Best Practices

* **Modular Stages:** Pipeline stages are separated for clarity and independent execution, enabling easier debugging and maintenance.
* **Security-First:** Trivy scans enforce security by failing the pipeline on critical vulnerabilities.
* **Reusable Connections:** Service connections for Docker Hub and SonarQube are configured once and reused, reducing configuration overhead.
* **Artifact Traceability:** Docker images are tagged with build IDs, ensuring traceability between builds and deployments.
* **Quality Gates:** SonarQube enforces strict code quality standards, preventing subpar code from reaching production.
* **Self-Hosted Agent:** Using a self-hosted agent reduces dependency on cloud-hosted agents and provides control over the environment.



## Prerequisites

* **Azure DevOps Account:** With a project and repository set up.
* **Azure Subscription:** For creating and managing the self-hosted VM.
* **Self-Hosted VM:** Ubuntu 20.04 VM in Azure with Docker and Azure DevOps agent installed.
* **Docker Hub Account:** For storing Docker images.
* **SonarQube Server:** Running on the VM (port 9000) with a service connection in Azure DevOps.
* **SSH Service Connection:** Configured in Azure DevOps for VM access.
* **Maven Project:** A Java application with a valid `pom.xml` .
* **Trivy:** Available as a Docker image for vulnerability scanning.



## Setup Instructions

### Creating a Self-Hosted Agent in Azure

**Provision the VM:**

1. Log in to the Azure Portal.
2. Navigate to **Virtual Machines > Create > Azure Virtual Machine**.
3. Select **Ubuntu Server 20.04 LTS** as the image.
4. Choose **Standard B2s** (2 vCPUs, 4 GiB memory) for the size.
5. Configure an admin username and SSH public key for authentication.
6. In the Networking tab, ensure the default VNet is used and add NSG rules to allow inbound traffic on:

   * Port 22 (SSH)
   * Port 8080 (Application)
   * Port 9000 (SonarQube)
7. Review and create the VM.

**Install Dependencies on the VM:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
sudo apt install openjdk-17-jdk -y
sudo apt install maven -y
```

### Setting Up a Self-Hosted Azure DevOps Agent

1. In Azure DevOps, navigate to **Project Settings > Agent Pools**.  
2. Create a new agent pool (e.g., `Self-Hosted-Pool`) or use the default pool.  
3. Select the pool, click **New Agent**, and download the agent package.
   
4.Create the agent

   ```bash
  mkdir myagent && cd myagent
  # download the agent
  wget https://download.agent.dev.azure.com/agent/4.261.0/vsts-agent-linux-x64-4.261.0.tar.gz
  tar zxvf vsts-agent-linux-x64-4.261.0.tar.gz
````

5. On the VM, extract and configure the agent:
   
   ```bash
   cd ~/agent
   ./config.sh
   ```

7. Follow the prompts:

   * Enter the Azure DevOps URL (e.g., `https://dev.azure.com/<organization>`).
   * Provide a **Personal Access Token (PAT)** with **Agent Pools (read, manage)** scope, created in Azure DevOps under **User Settings > Personal Access Tokens**.
   * Name the agent (e.g., `Test-Runner`).
   * Accept defaults for other settings.

8. Start the agent:

   ```bash
   ./run.sh
   ```

**Verify Agent Status:**

* In Azure DevOps, go to **Project Settings > Agent Pools > Self-Hosted-Pool**, and confirm the agent is online.



### Creating Docker Hub Service Connection

1. Log in to Docker Hub.
2. Go to **Account Settings > Security > Access Tokens**.
3. Create a new access token with **Read, Write, Delete** permissions and copy it.
4. In Azure DevOps, navigate to **Project Settings > Service Connections**.
5. Click **New Service Connection > Docker Registry**.
6. Select **Docker Hub** as the registry type.
7. Fill in the details:

   * Docker Registry: `https://index.docker.io/v1/`
   * Docker ID: Your Docker Hub username
   * Password: Paste the access token
   * Service Connection Name: `DockerHubServiceConnection`
8. Save and verify the connection.



### Creating SonarQube Service Connection

**Set Up SonarQube on the VM:**

```bash
ssh <username>@<vm-public-ip>
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```

* Access SonarQube at: `http://<vm-public-ip>:9000`
* Log in with default credentials (**admin / admin**) and change the password.

**Generate SonarQube Token:**

* In the SonarQube dashboard, go to **My Account > Security**.
* Generate a new token and copy it.

**Configure Service Connection in Azure DevOps:**

1. Go to **Project Settings > Service Connections**.
2. Click **New Service Connection > SonarQube**.
3. Fill in the details:

   * Server URL: `http://<vm-public-ip>:9000`
   * Token: Paste the SonarQube token
   * Service Connection Name: `SonarQubeServiceConnection`
4. Save and verify the connection.



### Additional Setup Steps

**Prepare the Application:**

* Ensure the Java application has a `pom.xml` with Maven dependencies and JUnit tests.
* Create a `Dockerfile` in the repository root (see example above).
* Commit and push the code to the Azure DevOps repository.

**Set Up the Pipeline:**

* In Azure DevOps, create a new pipeline and link it to your repository.
* Add the `azure-pipelines.yml` file (see YAML snippets above).
* Ensure the pipeline references the correct agent pool (**Self-Hosted-Pool**).

**Run the Pipeline:**

* Push changes to the main branch or create a pull request.
* Monitor the pipeline in Azure DevOps for build, test, analysis, and deployment status.


## Troubleshooting

* **Build Failure:** Check Maven logs for compilation or test errors. Ensure JDK 17 is installed on the VM.
* **SonarQube Failure:** Verify the SonarQube service connection and server accessibility. Check quality gate settings.
* **Docker Push Failure:** Confirm Docker Hub credentials in the service connection.
* **Trivy Failure:** Review the scan report for specific vulnerabilities and update dependencies or base images.
* **Deployment Failure:** Ensure SSH access to the VM and correct Docker commands in the pipeline.
* **Agent Offline:** Check the agent service status on the VM (`sudo ./svc.sh status`) and ensure the PAT is valid.



## Contributing

To contribute to this pipeline:

1. Fork the repository.
2. Create a feature branch:

   ```bash
   git checkout -b feature/new-feature
   ```
3. Update the pipeline or application code.
4. Test locally and in a test pipeline.
5. Submit a pull request with detailed changes.
