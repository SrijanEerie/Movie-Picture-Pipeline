# Movie Picture Pipeline

## Overview

Movie Picture Pipeline is a DevOps project that implements automated Continuous Integration and Continuous Deployment pipelines for a web application using GitHub Actions.

The application consists of a React/TypeScript frontend and a Python/Flask backend. Docker is used for containerization, Amazon Elastic Container Registry (ECR) is used for container image storage, and Amazon Elastic Kubernetes Service (EKS) is used for Kubernetes deployment.

The CI/CD pipelines automate code validation, testing, Docker image creation, image publishing, and Kubernetes deployment.

---

## Project Architecture

```text
                         GitHub Repository
                                |
                  +-------------+-------------+
                  |                           |
                  v                           v
             Frontend Code               Backend Code
          React / TypeScript            Python / Flask
                  |                           |
                  v                           v
            Frontend CI                 Backend CI
                  |                           |
            Lint / Test /             Lint / Test /
               Build                    Build
                  |                           |
                  +-------------+-------------+
                                |
                         GitHub Actions
                                |
                  +-------------+-------------+
                  |                           |
                  v                           v
             Frontend CD                 Backend CD
                  |                           |
                  v                           v
             Docker Build                Docker Build
                  |                           |
                  v                           v
              Amazon ECR                 Amazon ECR
                  |                           |
                  +-------------+-------------+
                                |
                                v
                           Amazon EKS
                                |
                    +-----------+-----------+
                    |                       |
                    v                       v
                Frontend                Backend
                Kubernetes              Kubernetes
                Deployment              Deployment
```

---

## Technologies Used

### Frontend

- React
- TypeScript
- Node.js
- npm
- ESLint
- Docker

### Backend

- Python
- Flask
- Pipenv
- Pytest
- uWSGI
- Docker

### DevOps and Cloud

- GitHub Actions
- Docker
- Amazon ECR
- Amazon EKS
- Kubernetes
- kubectl
- Kustomize
- Terraform
- AWS CLI

---

# CI/CD Workflows

The project contains four GitHub Actions workflows:

```text
.github/
└── workflows/
    ├── frontend-ci.yml
    ├── backend-ci.yml
    ├── frontend-cd.yml
    └── backend-cd.yml
```

The workflows provide separate CI and CD automation for the frontend and backend applications.

---

# Frontend Continuous Integration

## Workflow

`Frontend Continuous Integration`

## File

`.github/workflows/frontend-ci.yaml`

## Triggers

The workflow supports:

- Pull requests targeting the `main` branch
- Frontend path filtering
- Manual execution using `workflow_dispatch`

## Pipeline

```text
Pull Request
      |
      +-------------------+
      |                   |
      v                   v
    Lint                Test
      |                   |
      +---------+---------+
                |
                v
              Build
```

## Lint Job

The lint job:

1. Checks out the repository
2. Sets up Node.js
3. Restores the npm dependency cache
4. Installs dependencies using `npm ci`
5. Runs the ESLint command

```bash
npm run lint
```

## Test Job

The test job:

1. Checks out the repository
2. Sets up Node.js
3. Restores the npm dependency cache
4. Installs dependencies using `npm ci`
5. Runs the frontend test suite

```bash
CI=true npm test
```

The lint and test jobs run independently and in parallel.

## Build Job

The build job uses GitHub Actions `needs` so that the Docker build runs only after the lint and test jobs have completed successfully.

The frontend Docker image uses the backend API URL as a Docker build argument:

```bash
docker build \
  --build-arg=REACT_APP_MOVIE_API_URL=${{ secrets.MOVIE_API_URL }} \
  --tag=mp-frontend:${{ github.sha }} .
```

The image is tagged using the Git commit SHA for traceability.

---

# Backend Continuous Integration

## Workflow

`Backend Continuous Integration`

## File

`.github/workflows/backend-ci.yaml`

## Triggers

The workflow supports:

- Pull requests targeting the `main` branch
- Backend path filtering
- Manual execution using `workflow_dispatch`

## Pipeline

```text
Pull Request
      |
      +-------------------+
      |                   |
      v                   v
    Lint                Test
      |                   |
      +---------+---------+
                |
                v
              Build
```

## Lint Job

The backend lint job:

1. Checks out the repository
2. Sets up Python 3.10
3. Installs Pipenv
4. Installs project dependencies
5. Runs the backend lint command

```bash
pipenv run lint
```

## Test Job

The backend test job:

1. Checks out the repository
2. Sets up Python 3.10
3. Installs Pipenv
4. Installs project dependencies
5. Runs the backend test suite

```bash
pipenv run test
```

The lint and test jobs run in parallel.

## Build Job

The backend Docker build runs only after successful completion of the lint and test jobs.

The image is built from the backend application directory:

```text
starter/backend
```

The Docker image is tagged with the Git commit SHA:

```bash
docker build --tag=mp-backend:${{ github.sha }} .
```

---

# Frontend Continuous Deployment

## Workflow

`Frontend Continuous Deployment`

## File

`.github/workflows/frontend-cd.yaml`

## Triggers

The workflow supports:

- Pushes to the `main` branch
- Frontend path filtering
- Manual execution using `workflow_dispatch`

## Pipeline

```text
Push to main
      |
      +-------------------+
      |                   |
      v                   v
    Lint                Test
      |                   |
      +---------+---------+
                |
                v
          Docker Build
                |
                v
            Amazon ECR
                |
                v
             Amazon EKS
                |
                v
        Frontend Deployment
```

## Lint and Test

The deployment pipeline first validates the frontend application by running linting and tests.

The Docker build and deployment stages depend on successful completion of these validation jobs.

## Docker Build

The frontend Docker image is built with the backend API URL supplied through a GitHub Secret:

```bash
docker build \
  --build-arg=REACT_APP_MOVIE_API_URL=${{ secrets.MOVIE_API_URL }} \
  --tag=${{ secrets.ECR_FRONTEND_REPOSITORY }}:${{ github.sha }} .
```

This configures the frontend application to communicate with the deployed backend API.

## ECR Authentication

The workflow uses the AWS ECR login action:

```yaml
uses: aws-actions/amazon-ecr-login@v2
```

AWS credentials are accessed securely through GitHub repository secrets.

## Image Publishing

The generated Docker image is pushed to Amazon ECR:

```bash
docker push ${{ secrets.ECR_FRONTEND_REPOSITORY }}:${{ github.sha }}
```

The Git commit SHA is used as the image tag.

## Kubernetes Deployment

Kustomize is used to update the frontend deployment with the newly built image:

```bash
kustomize edit set image frontend=${{ secrets.ECR_FRONTEND_REPOSITORY }}:${{ github.sha }}
```

The Kubernetes manifests are then applied to the EKS cluster:

```bash
kustomize build | kubectl apply -f -
```

---

# Backend Continuous Deployment

## Workflow

`Backend Continuous Deployment`

## File

`.github/workflows/backend-cd.yaml`

## Triggers

The workflow supports:

- Pushes to the `main` branch
- Backend path filtering
- Manual execution using `workflow_dispatch`

## Pipeline

```text
Push to main
      |
      +-------------------+
      |                   |
      v                   v
    Lint                Test
      |                   |
      +---------+---------+
                |
                v
          Docker Build
                |
                v
            Amazon ECR
                |
                v
             Amazon EKS
                |
                v
         Backend Deployment
```

## Lint and Test

The backend deployment pipeline validates the application before building and deploying the Docker image.

Linting:

```bash
pipenv run lint
```

Testing:

```bash
pipenv run test
```

The build and deployment stages execute only after successful validation.

## Docker Build

The backend image is built and tagged with the Git commit SHA:

```bash
docker build \
  --tag=${{ secrets.ECR_BACKEND_REPOSITORY }}:${{ github.sha }} .
```

## ECR Authentication

The workflow uses:

```yaml
uses: aws-actions/amazon-ecr-login@v2
```

AWS credentials are accessed through GitHub repository secrets.

## Image Publishing

The backend Docker image is pushed to Amazon ECR:

```bash
docker push ${{ secrets.ECR_BACKEND_REPOSITORY }}:${{ github.sha }}
```

## Kubernetes Deployment

The backend Kubernetes deployment image is updated using Kustomize:

```bash
kustomize edit set image backend=${{ secrets.ECR_BACKEND_REPOSITORY }}:${{ github.sha }}
```

The updated manifests are applied using:

```bash
kustomize build | kubectl apply -f -
```

---

# GitHub Actions Secrets

Sensitive AWS credentials and deployment configuration are stored as GitHub Repository Secrets rather than being hard-coded into workflow files.

The workflows use the following secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
ECR_FRONTEND_REPOSITORY
ECR_BACKEND_REPOSITORY
MOVIE_API_URL
```

Secrets are referenced through GitHub Actions expressions:

```yaml
${{ secrets.AWS_ACCESS_KEY_ID }}
```

```yaml
${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

```yaml
${{ secrets.ECR_FRONTEND_REPOSITORY }}
```

```yaml
${{ secrets.ECR_BACKEND_REPOSITORY }}
```

```yaml
${{ secrets.MOVIE_API_URL }}
```

No AWS credential values are stored directly in the workflow source code.

---

# Docker Configuration

## Frontend

The frontend Dockerfile accepts the backend API URL as a build argument:

```dockerfile
ARG REACT_APP_MOVIE_API_URL
ENV REACT_APP_MOVIE_API_URL=${REACT_APP_MOVIE_API_URL}
```

This allows the backend API endpoint to be configured during the Docker image build.

## Backend

The backend application is containerized using Python 3.10 Alpine and uWSGI.

The backend application exposes port:

```text
5000
```

---

# Kubernetes Deployment

Both applications contain Kubernetes deployment manifests.

```text
starter/
├── frontend/
│   └── k8s/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
│
└── backend/
    └── k8s/
        ├── deployment.yaml
        ├── service.yaml
        └── kustomization.yaml
```

Kustomize dynamically updates the container image reference during deployment.

Each deployment uses the Git commit SHA as the Docker image tag.

This provides traceability between the source-code commit and the deployed container image.

---

# Backend API

The backend exposes the movie catalog through the `/movies` endpoint.

Deployed backend endpoint:

```text
http://ae71be944ac240c3aa1a7ee7c6fe6eef-944530289.us-east-1.elb.amazonaws.com/movies
```

The API returns the movie catalog in JSON format.

Example response:

```json
{
  "movies": [
    {
      "id": "123",
      "title": "Top Gun: Maverick"
    },
    {
      "id": "456",
      "title": "Sonic the Hedgehog"
    },
    {
      "id": "789",
      "title": "A Quiet Place"
    }
  ]
}
```

---

# Local Development

## Frontend

Navigate to the frontend directory:

```bash
cd starter/frontend
```

Install dependencies:

```bash
npm ci
```

Start the frontend application:

```bash
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start
```

The frontend development server runs on:

```text
http://localhost:3000
```

## Backend

Navigate to the backend directory:

```bash
cd starter/backend
```

Install dependencies:

```bash
pipenv install
```

Start the backend application:

```bash
pipenv run serve
```

The backend API runs on:

```text
http://localhost:5000
```

The movie endpoint is:

```text
http://localhost:5000/movies
```

---

# Testing

## Frontend Tests

Install dependencies:

```bash
npm ci
```

Run tests:

```bash
CI=true npm test
```

Run lint:

```bash
npm run lint
```

## Backend Tests

Install dependencies:

```bash
pipenv install
```

Run tests:

```bash
pipenv run test
```

Run lint:

```bash
pipenv run lint
```

---

# CI/CD Design

The pipelines follow several DevOps principles.

## Automated Validation

Linting and automated tests are executed before application builds and deployments.

## Parallel Execution

Independent lint and test jobs run in parallel to reduce pipeline execution time.

## Job Dependencies

Docker builds and deployment stages use GitHub Actions `needs` so they execute only after the required validation jobs succeed.

## Immutable Image Tagging

Docker images are tagged using the Git commit SHA:

```text
${{ github.sha }}
```

This provides a direct relationship between a Git commit and its corresponding container image.

## Secure Credentials

AWS credentials are stored in GitHub Repository Secrets and referenced through the GitHub Actions secrets context.

## Path Filtering

Frontend and backend workflows use path filters so changes to one application do not unnecessarily trigger the other application's workflow.

## Manual Execution

All workflows support manual execution using:

```yaml
workflow_dispatch:
```

---

# Repository Structure

```text
Movie-Picture-Pipeline/
│
├── .github/
│   └── workflows/
│       ├── frontend-ci.yaml
│       ├── backend-ci.yaml
│       ├── frontend-cd.yaml
│       └── backend-cd.yaml
│
├── starter/
│   ├── frontend/
│   │   ├── src/
│   │   ├── k8s/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── package-lock.json
│   │
│   └── backend/
│       ├── movies/
│       ├── k8s/
│       ├── Dockerfile
│       ├── Pipfile
│       └── Pipfile.lock
│
├── setup/
│   ├── init.sh
│   └── terraform/
│
└── README.md
```

---

# Deployment Flow

```text
                    Developer
                        |
                        v
                  GitHub Repository
                        |
             +----------+----------+
             |                     |
             v                     v
        Frontend Changes      Backend Changes
             |                     |
             v                     v
        Frontend CI            Backend CI
             |                     |
       Lint / Test /           Lint / Test /
          Build                   Build
             |                     |
             v                     v
        Frontend CD            Backend CD
             |                     |
             v                     v
          Docker                 Docker
             |                     |
             v                     v
        Amazon ECR             Amazon ECR
             |                     |
             +----------+----------+
                        |
                        v
                   Amazon EKS
                        |
              +---------+---------+
              |                   |
              v                   v
          Frontend             Backend
          Service                API
```

---

# Project Deliverables

The project implements:

- Frontend Continuous Integration workflow
- Backend Continuous Integration workflow
- Frontend Continuous Deployment workflow
- Backend Continuous Deployment workflow
- Automated linting
- Automated testing
- Docker image builds
- Git SHA based Docker image tagging
- Amazon ECR integration
- Amazon EKS deployment
- Kubernetes deployment using Kustomize
- GitHub Actions manual workflow execution
- Pull request based CI
- Main branch based CD
- GitHub Secrets based AWS authentication
- Frontend API URL configuration through Docker build arguments
- Automated frontend and backend deployment workflows

---

# GitHub Repository

[Movie Picture Pipeline - GitHub Repository](https://github.com/SrijanEerie/Movie-Picture-Pipeline)

---

# License

This project was developed as part of the Udacity DevOps CI/CD project.
