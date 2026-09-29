# Complete DevSecOps CI/CD Pipeline (GitHub Actions + Docker + K8s + Slack Alerts)

A production-oriented DevSecOps CI/CD pipeline for a containerized Flask application, implementing automated testing, security scanning, container vulnerability scanning, Docker image publishing, Kubernetes deployment, Blue-Green releases, rollback capability, secret management, and Slack notifications.

The project demonstrates how application code can move from **Git commit → security validation → container image → Kubernetes deployment → traffic switch → operational notification** through an automated CI/CD workflow.

---

## Business Problem

Software teams need to release application changes quickly without sacrificing security, reliability, or operational visibility.

A manual deployment process introduces several risks:

* Untested code reaching deployment environments
* Secrets accidentally committed to source control
* Vulnerable Python dependencies
* Vulnerable container images
* Manual Docker image publishing
* Deployment downtime during application updates
* Difficulty reverting a problematic release
* Lack of immediate deployment notifications

This project addresses those problems by implementing an automated **DevSecOps CI/CD pipeline** that validates, scans, builds, publishes, deploys, and monitors the release process.

---

# Solution Architecture

```text
                         Developer
                             │
                             │ git push
                             ▼
                    ┌─────────────────┐
                    │     GitHub      │
                    │   Repository    │
                    └────────┬────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │    GitHub Actions     │
                 │       CI/CD           │
                 └───────────┬───────────┘
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
          Testing         Security          Build
          pytest          Bandit            Docker
                         pip-audit             │
                         Gitleaks               ▼
                                            Trivy Scan
                                                │
                                                ▼
                                         Docker Hub
                                                │
                                                ▼
                                      Kubernetes / Kind
                                                │
                                  ┌─────────────┴─────────────┐
                                  │                           │
                              BLUE                        GREEN
                            Deployment                  Deployment
                                  │                           │
                                  └─────────────┬─────────────┘
                                                │
                                         Kubernetes Service
                                                │
                                                ▼
                                        Active Application
                                                │
                                                ▼
                                           Slack Alert
```

---

# CI/CD Pipeline

The GitHub Actions workflow follows this sequence:

```text
Git Push
   │
   ▼
┌──────────────┐
│     TEST     │
│    pytest    │
└──────┬───────┘
       │
       ├──────────────────┐
       ▼                  ▼
┌──────────────┐    ┌──────────────┐
│   SECURITY   │    │     BUILD    │
│    Bandit    │    │ Docker Build │
│  pip-audit   │    │    Trivy     │
│   Gitleaks   │    │ Docker Hub   │
└──────┬───────┘    └──────┬───────┘
       └──────────┬─────────┘
                  ▼
           Kubernetes Deploy
                  │
                  ▼
             GREEN Release
                  │
                  ▼
            Health Validation
                  │
                  ▼
          BLUE → GREEN Traffic
                  │
                  ▼
          Slack Notification
```

The workflow is defined in:

```text
.github/workflows/cicd.yml
```

---

# Automated Testing

The pipeline automatically installs the application's Python dependencies and executes the test suite using `pytest`.

```bash
pytest
```

This prevents the build/deployment stages from proceeding when the application test stage fails.

---

# DevSecOps Security Controls

Security is integrated directly into the CI/CD pipeline rather than being treated as a separate manual activity.

## Bandit

Static security analysis of the Python application:

```bash
python -m bandit -r . --exclude ./test_app.py
```

Used to identify common security issues in Python code.

## pip-audit

Python dependency vulnerability scanning:

```bash
pip-audit
```

This checks project dependencies against known vulnerability information.

## Gitleaks

Secret-leak detection is integrated into GitHub Actions to identify credentials or sensitive information accidentally committed to the repository.

## Trivy

The built Docker image is scanned for HIGH and CRITICAL vulnerabilities before the image is published.

```text
Application
    │
    ▼
Docker Build
    │
    ▼
Trivy Image Scan
    │
    ├── Vulnerability findings
    │
    ▼
Docker Hub
```

The pipeline is configured to report HIGH/CRITICAL vulnerabilities while ignoring unfixed vulnerabilities.

---

# Containerization

The Flask backend is packaged as a Docker image.

The CI pipeline:

1. Builds the image
2. Scans the image with Trivy
3. Authenticates with Docker Hub
4. Tags the image using the Git commit SHA
5. Pushes the image to Docker Hub

Example image tag:

```text
<dockerhub-user>/devsecops-backend:<git-sha>
```

Using the Git commit SHA provides immutable image identification and allows a deployed container to be traced back to a specific source-code revision.

---

# Kubernetes Deployment

The application is deployed to a Kubernetes cluster running on **Kind** for CI/CD integration testing.

Kubernetes resources include:

```text
k8s/
├── configmap.yml
├── secret.yml
├── deployment-blue.yml
├── deployment-green.yml
├── service.yml
└── ingress.yml
```

The application containers expose port:

```text
5000
```

The Kubernetes Service exposes the application through:

```text
NodePort: 30080
```

---

# Blue-Green Deployment

The deployment strategy is designed to reduce release risk by maintaining two application versions:

```text
BLUE
GREEN
```

The Kubernetes deployments use labels to distinguish the two environments.

### BLUE

```yaml
color: blue
```

### GREEN

```yaml
color: green
```

The Service initially selects BLUE:

```yaml
selector:
  app: backend
  color: blue
```

The pipeline then deploys GREEN independently.

---

## Release Flow

```text
                 Existing BLUE
                      │
                      │
                      ▼
               Service → BLUE
                      │
                      │
                Deploy GREEN
                      │
                      ▼
              GREEN Pods Start
                      │
                      ▼
              Health/Readiness
                 Validation
                      │
                      ▼
              GREEN is Ready
                      │
                      ▼
            Service Selector
              BLUE → GREEN
                      │
                      ▼
                Live Traffic
                      │
                      ▼
                    GREEN
```

This allows the new version to become healthy before application traffic is moved to it.

---

# Health Checks

The application deployment uses Kubernetes health probes.

### Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
```

### Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 5000
```

The readiness probe prevents Kubernetes from considering a container ready to receive traffic until the application health endpoint responds successfully.

---

# Rollback Strategy

Blue-Green deployment keeps the previous BLUE deployment available after GREEN becomes active.

If GREEN has a problem, traffic can be switched back:

```text
GREEN
  │
  │ problem detected
  ▼
Service selector
  │
  ▼
BLUE
```

Rollback is therefore performed by changing the Service selector:

```yaml
selector:
  app: backend
  color: blue
```

This avoids rebuilding the previous application version just to restore service.

---

# Secret Management

Sensitive credentials are not hard-coded into the repository.

GitHub Actions secrets are used for credentials such as:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
SLACK_WEBHOOK_URL
```

The workflow references them through GitHub's secrets mechanism:

```yaml
${{ secrets.DOCKERHUB_USERNAME }}
```

The Docker Hub token and Slack webhook are therefore kept outside the source code.

The GitHub-provided:

```text
GITHUB_TOKEN
```

is used where required by GitHub Actions.

---

# Slack Notifications

The pipeline sends CI/CD status notifications to Slack.

The notification provides operational visibility after the workflow completes.

Example successful notification:

```text
 CI/CD Pipeline SUCCESS

Repository: production-ready-devsecops-pipeline
Branch: main
Commit: 5a47862
Deployment: Blue-Green deployment completed successfully
```

A failed pipeline generates a failure notification so that deployment problems do not depend on manually monitoring the GitHub Actions interface.

---

# Security Pipeline

The project implements security controls at multiple stages:

```text
Source Code
    │
    ├── pytest
    │
    ├── Bandit
    │
    ├── pip-audit
    │
    └── Gitleaks
          │
          ▼
      Docker Build
          │
          ▼
      Trivy Scan
          │
          ▼
      Docker Registry
          │
          ▼
      Kubernetes
```

This demonstrates a shift-left security approach where security validation occurs before deployment.

---

# Deployment Verification

The pipeline verifies Kubernetes resources after deployment using commands such as:

```bash
kubectl get deployment
kubectl get pods
kubectl get svc
kubectl get endpoints
```

GREEN pods are specifically validated using:

```bash
kubectl get pods -l app=backend,color=green
```

The Service endpoints are then checked after the traffic switch.

---

# 📸 Screenshots & Results

### 1. GitHub Actions Pipeline Blue-Green Deployment

<img width="1570" height="285" alt="Slack-pipeline" src="https://github.com/user-attachments/assets/51b8db29-173e-45bb-be53-f9f141583eb2" />

### 3. GREEN Pods

### 5. Slack Notification

<img width="1396" height="458" alt="slack-notification" src="https://github.com/user-attachments/assets/83c6a930-fc19-4898-a4ea-00d22d98bafa" />


### 6. Security Scanning



# Technology Stack

| Category               | Technology                                 |
| ---------------------- | ------------------------------------------ |
| Source Control         | Git / GitHub                               |
| CI/CD                  | GitHub Actions                             |
| Application            | Python / Flask                             |
| Testing                | pytest                                     |
| SAST                   | Bandit                                     |
| Dependency Security    | pip-audit                                  |
| Secret Detection       | Gitleaks                                   |
| Container              | Docker                                     |
| Container Security     | Trivy                                      |
| Registry               | Docker Hub                                 |
| Orchestration          | Kubernetes                                 |
| Kubernetes Environment | Kind                                       |
| Deployment Strategy    | Blue-Green                                 |
| Health Checks          | Kubernetes Liveness / Readiness Probes     |
| Notifications          | Slack                                      |
| Configuration          | Kubernetes ConfigMap                       |
| Secrets                | Kubernetes Secret + GitHub Actions Secrets |

---

# Project Structure

```text
Complete-DevSecOps-CI/CD-Pipeline/
│
├── .github/
│   └── workflows/
│       └── cicd.yml
│
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   └── ...
│
├── k8s/
│   ├── configmap.yml
│   ├── secret.yml
│   ├── deployment-blue.yml
│   ├── deployment-green.yml
│   ├── service.yml
│   └── ingress.yml
│
├── tests/
│   └── ...
│
├── README.md
├── LICENSE
└── .gitignore
```

---

# Engineering Outcomes

This project demonstrates an automated software delivery process that:

* Validates application changes automatically
* Integrates security into CI
* Detects vulnerable Python dependencies
* Detects secrets in source control
* Scans container images for vulnerabilities
* Builds immutable Docker images using Git commit SHA
* Publishes images to a container registry
* Deploys applications to Kubernetes
* Uses health checks before traffic activation
* Implements Blue-Green deployment
* Keeps the previous deployment available for rollback
* Automates traffic switching
* Provides Slack-based operational visibility
* Uses external secret management rather than hard-coded credentials

---

# Skills Demonstrated

### DevOps

* Git
* GitHub
* GitHub Actions
* Docker
* Docker Hub
* Kubernetes
* Kind
* CI/CD
* Blue-Green deployments
* Rollback strategies
* Health checks

### DevSecOps

* SAST with Bandit
* Dependency vulnerability scanning with pip-audit
* Secret scanning with Gitleaks
* Container security scanning with Trivy
* Secure credential handling
* Shift-left security

### Kubernetes

* Deployments
* Services
* ConfigMaps
* Secrets
* Labels and selectors
* Readiness probes
* Liveness probes
* Rolling deployment concepts
* Blue-Green deployment
* Traffic switching
* Rollback

### Automation & Operations

* GitHub Actions workflow orchestration
* Immutable image tagging
* Automated deployment verification
* Slack notifications
* CI/CD failure visibility

---

# Why This Project Matters

This project goes beyond demonstrating individual DevOps tools.

It demonstrates how those tools can be combined into an automated software delivery system:

```text
Code
 ↓
Test
 ↓
Secure
 ↓
Build
 ↓
Scan
 ↓
Publish
 ↓
Deploy
 ↓
Validate
 ↓
Switch Traffic
 ↓
Notify
 ↓
Rollback if Required
```

The key objective is to make application delivery **repeatable, security-aware, observable, and reversible** rather than dependent on manual deployment steps.


# Author

**Yasir-Z**

DevOps / DevSecOps portfolio project focused on secure automation, Kubernetes deployments, CI/CD engineering, Slack Notification.

