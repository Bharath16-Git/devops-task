# Formlabs DevOps Home Assignment

# Overview

This repository contains my implementation of the Formlabs DevOps take-home assignment.

The provided application is a small Python Flask service that returns Hello, World!. The task was to containerize it, deploy it to Kubernetes, and automate the build, test, and deployment process whenever changes are pushed to Git.

I used Docker, Kubernetes, and GitHub Actions. I kept the implementation straightforward, with a focus on repeatable deployments, basic container security, health checks, and validation at each stage.

## Tech Stack


| Component | Technology |
|---|---|
| Application | Python 3.11, Flask, Gunicorn |
| Containerization | Docker |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |
| Local Kubernetes | Docker Desktop |
| CI Kubernetes | kind |


## Repository Structure


```text
devops-task/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── helloapp/
│   ├── __init__.py
│   ├── app.py
│   └── test.py
├── kubernetes/
│   ├── deployment.yaml
│   └── service.yaml
├── .dockerignore
├── .gitignore
├── Dockerfile
├── requirements.txt
├── build.sh
├── run.sh
├── test.sh
└── README.md
```


The Flask application, its tests, and the original shell scripts were provided in the starter repository.

## 1. Containerization

I created a Dockerfile using python:3.11-slim as the base image.

The Dockerfile installs the dependencies from requirements.txt, copies the application code, and starts Gunicorn on port 8080.

I also configured the container to run under a non-root user and added a .dockerignore file to keep unnecessary files out of the build context.

### Build the image

```powershell
docker build -t formlabs-helloapp:local .
```

### Run the container

```powershell
docker run -d --name formlabs-app -p 8080:8080 formlabs-helloapp:local
```

### Verify

curl http://localhost:8080

Expected response:

<p>Hello, World!</p>

I verified the image locally and confirmed that the container returned HTTP 200.

## 2. Kubernetes Deployment

I created a Kubernetes Deployment and Service using standard YAML manifests.

The Deployment runs two application replicas and includes:

Readiness and liveness probes on port 8080.

CPU and memory requests and limits.

A locally available container image with imagePullPolicy: Never.

The Service uses ClusterIP and forwards traffic from port 80 to container port 8080.

I used standard Kubernetes manifests instead of Helm because the application only requires a Deployment and Service. For this scope, plain YAML is easier to review and maintain.

### Local Kubernetes setup

The application image must be loaded into the Kubernetes node before deployment.

For my Docker Desktop environment, I used PowerShell to export the image and import it into the cluster's containerd runtime:

docker save -o formlabs-helloapp.tar formlabs-helloapp:local

docker cp .\formlabs-helloapp.tar desktop-control-plane:/root/formlabs-helloapp.tar

docker exec desktop-control-plane ctr -n k8s.io images import /root/formlabs-helloapp.tar

The node name may differ depending on the local cluster configuration.

### Deploy the application

kubectl apply -f kubernetes/
kubectl rollout status deployment/formlabs-helloapp
kubectl get pods -l app=formlabs-helloapp

### Access the Service

kubectl port-forward service/formlabs-helloapp 8081:80

In another terminal:

curl http://localhost:8081

During local testing, both replicas reached the Ready state, and the Service returned HTTP 200 with the expected application response.

## 3. CI/CD Automation

The GitHub Actions workflow is defined in .github/workflows/ci-cd.yml.

It runs on pushes to master and pull requests targeting master.

The pipeline performs the following steps:

Checks out the repository.

Sets up Python and installs dependencies.

Runs the provided unit tests.

Builds the Docker image.

Creates a temporary Kubernetes cluster using kind.

Loads the Docker image into the cluster.

Applies the Kubernetes manifests.

Waits for the Deployment rollout to finish.

Verifies the application response through the Kubernetes Service.

The workflow fails if the unit tests, image build, deployment rollout, or HTTP verification fails.

### CI environment

I used kind to create a temporary Kubernetes environment within GitHub Actions.

This keeps the deployment test reproducible without requiring a separate cloud account or persistent cluster.

The workflow uses the chart-testing cluster created by the kind action when loading the image.

During the first CI run, the image-loading step failed because it referenced the default kind cluster name. I corrected the cluster reference, committed the fix, and reran the workflow successfully.

The successful run can be reviewed under the repository's GitHub Actions tab.

## 4. Design Decisions

Docker: I used a Python slim image and a non-root user. This keeps the container setup relatively lightweight and avoids running the application with unnecessary root privileges.

Kubernetes: A Deployment manages the application replicas, while the ClusterIP Service provides a stable internal endpoint. Readiness and liveness probes support basic health monitoring.

GitHub Actions: Since the repository is already hosted on GitHub, GitHub Actions provides a simple way to automate the build, test, and deployment process without maintaining a separate CI server.

kind: I selected kind for CI because it provides a lightweight Kubernetes cluster that can be created and removed as part of each workflow run.

Plain YAML: I avoided Helm and additional infrastructure tooling because they would add complexity without much benefit for a single application Deployment and Service.

## 5. Testing and Validation

I tested the application locally before configuring the automated deployment workflow.


| Validation | Result |
|---|---|
| Docker image build | Passed |
| Container startup | Passed |
| Local application HTTP test | HTTP 200 |
| Kubernetes rollout | Passed |
| Kubernetes replicas | 2/2 Ready |
| Kubernetes Service HTTP test | HTTP 200 |
| Python unit tests in CI | Passed |
| GitHub Actions workflow | Passed |


The local Kubernetes deployment and CI pipeline both returned the expected Hello, World! response.

## 6. Production Considerations

The current setup is intended to demonstrate the requested DevOps workflow. It is not a persistent production deployment.

The Kubernetes cluster created in GitHub Actions is temporary and is removed after execution. This validates deployment behavior but does not provide a continuously running service.

For a production environment, I would consider:

Publishing versioned container images to ECR or GHCR.

Using a persistent Kubernetes cluster such as EKS or GKE.

Separating development, staging, and production configurations.

Adding automated dependency and container vulnerability scanning.

Centralizing metrics, logs, and alerts.

Introducing deployment approvals and rollback procedures.

Managing access and secrets through least-privilege policies.

I kept these improvements outside the current scope to focus on a working and easily reproducible implementation.

Final Notes

The assignment covers the requested Docker image, Kubernetes Deployment and Service, and Git-triggered build, test, and deployment automation.

The implementation was tested locally and through GitHub Actions. The final result is a small, maintainable deployment workflow with clear validation steps and documented design choices.