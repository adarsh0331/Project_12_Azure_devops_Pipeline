
# CI/CD Pipeline for Java Application

This document outlines the Continuous Integration and Continuous Deployment (CI/CD) pipeline for a Java application, implemented using Azure DevOps, with integrations for Maven, SonarQube, Docker, and Trivy. The pipeline automates the build, test, code quality analysis, containerization, vulnerability scanning, and deployment to a self-hosted virtual machine (VM).

---

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
- [Best Practices](#best-practices)  
- [Prerequisites](#prerequisites)  
- [Setup Instructions](#setup-instructions)  
  - [Creating a Self-Hosted Agent in Azure](#creating-a-self-hosted-agent-in-azure)  
  - [Creating Docker Hub Service Connection](#creating-docker-hub-service-connection)  
  - [Creating SonarQube Service Connection](#creating-sonarqube-service-connection)  
  - [Additional Setup Steps](#additional-setup-steps)  
- [Troubleshooting](#troubleshooting)  
- [Contributing](#contributing)  

---

## Overview
The CI/CD pipeline automates the process of building, testing, analyzing, and deploying a Java application. Key features include:

- **Automated Builds**: Triggered on code pushes to the main branch or pull requests.  
- **Code Quality**: Static analysis with SonarQube to enforce quality gates.  
- **Containerization**: Docker image creation and storage in Docker Hub.  
- **Security**: Vulnerability scanning with Trivy to ensure secure images.  
- **Deployment**: Deployment to a self-hosted Ubuntu VM running the application in a Docker container.  

The pipeline ensures high code quality, security, and traceability, with modular stages for maintainability.

---

## Infrastructure Setup
The pipeline deploys to a self-hosted virtual machine with the following specifications:

- **VM Type**: Azure Standard B2s (2 vCPUs, 4 GiB memory)  
- **Operating System**: Ubuntu 20.04 LTS  
- **Ports Open**:  
  - `22` (SSH) – For remote access and pipeline agent communication  
  - `8080` (Application) – For the Java application running in a Docker container  
  - `9000` (SonarQube) – For accessing the SonarQube dashboard  

- **Networking**:  
  - Default Virtual Network (VNet) and Network Security Group (NSG) configured in Azure.  
  - NSG rules allow inbound traffic on ports 22, 8080, and 9000 from trusted IP ranges.  

- **Self-Hosted Agent**: Azure DevOps agent installed on the VM to execute pipeline tasks.  

---

## Tools and Services
- **Azure DevOps** – CI/CD orchestration platform for pipeline execution and repository management.  
- **Maven** – Build tool for compiling, testing, and packaging the Java application.  
- **SonarQube** – Static code analysis tool for code quality and security checks.  
- **Docker** – Containerization platform for building and deploying the application.  
- **Trivy** – Vulnerability scanner for Docker images to detect security issues.  
- **Docker Hub** – Registry for storing and retrieving Docker images.  
- **JUnit** – Testing framework for unit tests, integrated with Maven.  

---

## Pipeline Flow
The pipeline is triggered on code pushes to the main branch or pull requests. It progresses through the following stages:

1. **Build & Test** – Compile code, run unit tests, publish results.  
2. **SonarQube Analysis** – Static analysis for bugs, vulnerabilities, code smells.  
3. **Docker Build & Push** – Containerize app, push to Docker Hub.  
4. **Trivy Vulnerability Scan** – Scan Docker image for vulnerabilities.  
5. **Deployment** – Deploy to self-hosted VM with Docker.  

---

## Pipeline Stages

### Build & Test
- **Trigger**: Push to main branch or pull requests.  
- **Tasks**:  
  - Compile Java code with Maven.  
  - Run JUnit tests.  
  - Publish test results to Azure DevOps.  
- **Output**: Compiled JAR stored as pipeline artifact.  

### SonarQube Analysis
- **Purpose**: Ensure code quality and security.  
- **Tasks**:  
  - Run SonarQube scanner.  
  - Enforce quality gates (e.g., no critical issues, test coverage).  
- **Output**: Results available in SonarQube dashboard.  

### Docker Build & Push
- **Purpose**: Containerize app for deployment.  
- **Tasks**:  
  - Build Docker image using Dockerfile.  
  - Tag image with build ID.  
  - Push to Docker Hub.  
- **Output**: Image stored in Docker Hub.  

### Trivy Vulnerability Scan
- **Purpose**: Ensure secure images.  
- **Tasks**:  
  - Scan Docker image with Trivy.  
  - Fail pipeline if critical/high vulnerabilities found.  
- **Output**: Scan report published.  

### Deployment
- **Purpose**: Run app on VM.  
- **Tasks**:  
  - SSH to VM.  
  - Pull Docker image.  
  - Stop/remove old container.  
  - Run new container on port `8080`.  
- **Output**: App running and accessible on port `8080`.  

---

## Key YAML Snippets
```yaml
# Pipeline Trigger
trigger:
  branches:
    include:
      - main

# Pool Configuration
pool:
  name: 'Self-Hosted-Agent'

# Variables
variables:
  mavenPomFile: 'pom.xml'
  dockerImageName: 'myapp:latest'
  dockerHubConnection: 'DockerHubServiceConnection'
  sonarQubeServiceConnection: 'SonarQubeServiceConnection'

# Stages
stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: Maven@3
            inputs:
              mavenPomFile: '$(mavenPomFile)'
              goals: 'clean package'
              publishJUnitResults: true
              testResultsFiles: '**/surefire-reports/TEST-*.xml'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '1.17'

  - stage: SonarQube
    dependsOn: Build
    jobs:
      - job: SonarQubeAnalysis
        steps:
          - task: SonarQubePrepare@5
            inputs:
              SonarQube: '$(sonarQubeServiceConnection)'
              scannerMode: 'CLI'
              configMode: 'manual'
              cliProjectKey: 'myapp'
              cliSources: '.'
          - task: SonarQubeAnalyze@5
          - task: SonarQubePublish@5
            inputs:
              pollingTimeoutSec: '300'

  - stage: Docker
    dependsOn: SonarQube
    jobs:
      - job: BuildAndPush
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: '$(dockerHubConnection)'
              repository: 'myorg/myapp'
              command: 'buildAndPush'
              Dockerfile: 'Dockerfile'
              tags: '$(Build.BuildId)'

  - stage: Trivy
    dependsOn: Docker
    jobs:
      - job: VulnerabilityScan
        steps:
          - script: |
              docker pull aquasec/trivy:latest
              docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity CRITICAL,HIGH myorg/myapp:$(Build.BuildId)
            displayName: 'Run Trivy Scan'

  - stage: Deploy
    dependsOn: Trivy
    jobs:
      - job: DeployToVM
        steps:
          - task: SSH@0
            inputs:
              sshEndpoint: 'SelfHostedVM'
              runOptions: 'commands'
              commands: |
                docker pull myorg/myapp:$(Build.BuildId)
                docker stop myapp || true
                docker rm myapp || true
                docker run -d --name myapp -p 8080:8080 myorg/myapp:$(Build.BuildId)
````

---

## Example Dockerfile

```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/myapp.jar /app/myapp.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/myapp.jar"]
```

---

## Best Practices

* **Modular Stages** – Independent execution for clarity & debugging.
* **Security-First** – Trivy scans fail builds on vulnerabilities.
* **Reusable Connections** – Docker & SonarQube service connections reused.
* **Artifact Traceability** – Images tagged with build IDs.
* **Quality Gates** – Enforced via SonarQube.
* **Self-Hosted Agent** – Full control over execution environment.

---

## Prerequisites

* Azure DevOps account with project & repo.
* Azure subscription for VM.
* Self-hosted Ubuntu 20.04 VM with Docker + DevOps agent.
* Docker Hub account.
* SonarQube server running (port `9000`).
* SSH service connection.
* Java app with `pom.xml` & JUnit tests.
* Trivy available via Docker.

---

## Setup Instructions

### Creating a Self-Hosted Agent in Azure

1. Provision VM (Ubuntu 20.04, Standard B2s).
2. Open ports `22`, `8080`, `9000`.
3. SSH into VM → install Docker, Java 17, Maven.
4. Configure Azure DevOps agent using PAT.
5. Run agent as a service.
6. Verify agent is online in DevOps.

### Creating Docker Hub Service Connection

1. Generate Docker Hub access token.
2. In Azure DevOps → Service Connections → Docker Registry → Docker Hub.
3. Enter credentials & save.

### Creating SonarQube Service Connection

1. Run SonarQube in Docker on VM (port `9000`).
2. Generate token in SonarQube.
3. In Azure DevOps → Service Connections → SonarQube.
4. Enter server URL + token.

### Additional Setup Steps

* Configure SSH service connection for VM.
* Ensure repo has `pom.xml` and `Dockerfile`.
* Add `azure-pipelines.yml` to repo.

---

## Troubleshooting

* **Build Failure**: Check Maven logs, ensure JDK 17 installed.
* **SonarQube Failure**: Verify service connection, check quality gate.
* **Docker Push Failure**: Validate Docker Hub credentials.
* **Trivy Failure**: Review vulnerabilities, update dependencies.
* **Deployment Failure**: Check SSH & Docker commands.
* **Agent Offline**: Restart service, validate PAT.

---

## Contributing

1. Fork the repository.
2. Create feature branch: `git checkout -b feature/new-feature`.
3. Update pipeline or code.
4. Test locally and in pipeline.
5. Submit PR with details.
