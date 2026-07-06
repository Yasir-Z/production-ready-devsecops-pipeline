# DevSecOps Portfolio Project

## Project Overview

This project demonstrates a complete end-to-end DevSecOps CI/CD pipeline for a Python Flask application. The pipeline automates testing, security scanning, containerization, image vulnerability scanning, publishing to Docker Hub, and deployment to Kubernetes.

The primary objective of this project is to implement secure software delivery by integrating automated security checks throughout the CI/CD pipeline while following DevSecOps best practices.

---

# Architecture

Architecture Diagram

                    Developer
                        │
                  git push
                        │
                        ▼
              GitHub Repository
                        │
                        ▼
              GitHub Actions CI
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
     Pytest          Bandit         pip-audit
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                  Gitleaks Scan
                        │
                        ▼
                 Docker Build
                        │
                        ▼
                 Trivy Scan
                        │
              HIGH/CRITICAL?
                 │          │
               YES          NO
                │            ▼
           Pipeline Fails  Docker Hub
                              │
                              ▼
                     Kubernetes Cluster
                              │
                         Deployment
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
                  Pod 1             Pod 2
                     ▲                 ▲
                     └──── Service ────┘
                              ▲
                           Ingress
                              ▲
                           Internet

The deployment flow follows this architecture:

Developer → GitHub → GitHub Actions → Security Gates → Docker Build → Trivy Image Scan → Docker Hub → Kubernetes Deployment → Service → Ingress → Users

---

# Technology Stack

### Application

* Python 3.12
* Flask

### Version Control

* Git
* GitHub

### CI/CD

* GitHub Actions

### Security

* Pytest
* Bandit
* pip-audit
* Gitleaks
* Trivy

### Containerization

* Docker
* Docker Hub

### Container Orchestration

* Kubernetes

---

# CI/CD Pipeline

The GitHub Actions workflow performs the following stages automatically whenever code is pushed to the `main` branch.

## 1. Checkout Repository

Downloads the latest source code onto the GitHub Actions runner.

## 2. Setup Python

Installs Python 3.12.

## 3. Install Dependencies

Installs all project dependencies from `requirements.txt`.

## 4. Run Unit Tests

Executes Pytest to ensure the application functions correctly before continuing.

## 5. Static Application Security Testing (SAST)

Bandit scans the Python source code for common security issues.

## 6. Dependency Vulnerability Scan

pip-audit checks installed Python packages for known CVEs.

## 7. Secret Detection

Gitleaks scans the repository to prevent accidental exposure of credentials or secrets.

## 8. Docker Image Build

Builds a Docker image only after all previous quality and security gates have passed.

## 9. Container Image Scan

Trivy scans the Docker image for HIGH and CRITICAL vulnerabilities.

## 10. Publish Docker Image

The validated image is tagged with the Git commit SHA and pushed to Docker Hub.

---

# Security Controls

This project implements multiple automated security gates.

| Tool      | Purpose                             |
| --------- | ----------------------------------- |
| Pytest    | Application testing                 |
| Bandit    | Static code security analysis       |
| pip-audit | Dependency vulnerability scanning   |
| Gitleaks  | Secret detection                    |
| Trivy     | Docker image vulnerability scanning |

The pipeline follows the **fail-fast principle**, preventing insecure or broken code from progressing to later deployment stages.

---

# Kubernetes Deployment

The application is deployed using Kubernetes with the following resources:

* Deployment
* Service
* ConfigMap
* Secret
* Ingress

Production-oriented practices demonstrated include:

* Multiple replicas
* Rolling updates
* Rollback strategy
* Liveness Probe
* Readiness Probe
* Resource requests and limits
* External configuration using ConfigMaps
* Sensitive configuration using Secrets

---

# Project Screenshots

## GitHub Actions

![GitHub Actions](images/github-actions.png)

---

## Docker Hub

![Docker Hub](images/dockerhub.png)

---

## Trivy Scan

![Trivy](images/trivy.png)

---

## Kubernetes

![Kubernetes](images/kubernetes.png)

---

# 📂 Project Structure

```text
devsecops-portfolio/
│
├── backend/
├── frontend/
├── k8s/
├── .github/workflows/
├── images/
├── README.md
└── .gitignore
```

---

# Running the Project

Clone the repository:

```bash
git clone https://github.com/Yasir-Z/production-ready-devsecops-pipeline.git
```

Build the Docker image:

```bash
docker build -t devsecops-backend:v1 backend/
```

Run the container:

```bash
docker run -p 5000:5000 devsecops-backend:v1
```

Deploy to Kubernetes:

```bash
kubectl apply -f k8s/
```

---

# Future Improvements

* Terraform for infrastructure provisioning
* Ansible for configuration management
* Prometheus monitoring
* Grafana dashboards
* Kubernetes Horizontal Pod Autoscaler
* Production-ready cloud deployment

---

# Yasir Zafar

Built as a practical DevSecOps portfolio project demonstrating secure CI/CD pipelines, Docker, Kubernetes, and automated security scanning.
