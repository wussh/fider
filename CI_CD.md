# CI/CD Pipeline Explanation

This project utilizes a **CI/CD (Continuous Integration/Continuous Delivery)** pipeline to automate the processes of building, testing, and deploying the application. The pipeline ensures that every code change is thoroughly validated, maintaining the codebase in a deployable state and reducing the risk of bugs in production.

## Overview of the CI/CD Pipeline

The CI/CD pipeline automates the following key processes:
- **Continuous Integration (CI)**: Automatically builds and tests the application whenever code is pushed or a pull request is made to the `main` branch.
- **Continuous Delivery (CD)**: Prepares the application for deployment by building and pushing a Docker image to Docker Hub, and then deploying it to a server via SSH, but only when changes are merged into the `main` branch.

This setup ensures that the code is always tested and ready for production, with deployments being automated following successful pipeline runs.

## Tools and Technologies Used

The pipeline leverages the following tools and technologies:
- **[GitHub Actions](https://github.com/features/actions)**: Orchestrates the CI/CD workflow, defining jobs and steps to automate the process.
- **[Docker](https://www.docker.com/)**: Containerizes the application for consistent deployment across environments.
- **[Go](https://golang.org/)** and **[Node.js](https://nodejs.org/)**: The primary programming languages used for the server and UI, respectively.
- **[PostgreSQL](https://www.postgresql.org/)**: A relational database used for testing the application's data layer.
- **[MinIO](https://min.io/)**: An object storage service used to simulate S3-compatible storage during tests.
- **[golangci-lint](https://golangci-lint.run/)** and **[ESLint](https://eslint.org/)**: Linters for maintaining code quality in Go and JavaScript/TypeScript.
- **[SSH](https://en.wikipedia.org/wiki/Secure_Shell)**: Used for securely deploying the application to the server.

## Pipeline Stages

The pipeline is divided into two main jobs:

1. **Build, Test, and Lint** (`build-test-lint` job):
   - **Build**: Compiles the Go server code and builds the Node.js UI.
   - **Test**: Runs unit tests for both the server (Go) and the UI (Node.js).
   - **Lint**: Checks the code for style and quality issues using linters for both Go and JavaScript/TypeScript.

2. **Deploy** (`deploy` job):
   - Builds a Docker image of the application.
   - Pushes the image to Docker Hub, tagged as `latest` and with the commit hash.
   - Deploys the application to the server via SSH by:
     - Cloning the repository to the server.
     - Stopping any running containers with `docker compose down`.
     - Starting the containers with `docker compose up -d`.
   - This job only runs after the `build-test-lint` job succeeds and only on pushes to the `main` branch.

## How to Trigger the Pipeline

The pipeline is automatically triggered in the following scenarios:
- **Pushes** to the `main` branch.
- **Pull requests** targeting the `main` branch.

This ensures that all changes are validated before merging and that the `main` branch is always in a deployable state.

## Environment Variables and Secrets

To function correctly, the pipeline relies on several environment variables and secrets:

### Secrets
These must be configured in the GitHub repository settings under **Secrets and variables** > **Actions**:
- **`DOCKERHUB_USERNAME`**: Your Docker Hub username.
- **`DOCKERHUB_TOKEN`**: Your Docker Hub access token (for authentication when pushing images).
- **`DC_HOST`**: The hostname or IP address of the deployment server.
- **`DC_USER`**: The SSH username for the deployment server.
- **`DC_PASS`**: The SSH password for the deployment server.
- **`DC_PORT`**: The SSH port for the deployment server.

### Environment Variables
These are set within the workflow file and used during the pipeline execution:
- **`COMMITHASH`**: The unique identifier of the current commit (e.g., `github.sha`).
- **`VERSION`**: The version of the application, derived from the branch or tag name.
- **`DATABASE_URL`**: Connection string for the PostgreSQL database used in testing.
- **`NODE_ENV`**: Set to `test` to configure the Node.js environment for testing.
- **`BLOB_STORAGE_S3_ENDPOINT_URL`**: The endpoint URL for MinIO, simulating an S3-compatible storage service.
- **`REPO_URL`**: The URL of the GitHub repository.
- **`PROJECT_DIR`**: The directory on the server where the project will be cloned (e.g., `/home/user/prod`).

## Monitoring and Logs

You can monitor the status of pipeline runs directly in the GitHub repository:
- Navigate to the **Actions** tab to view all workflow runs.
- Each run provides detailed logs for every step, which are essential for diagnosing any issues or failures.

## Troubleshooting

Here are some common issues and how to resolve them:

- **Build Failures**:
  - Check the build logs for compilation errors in Go or Node.js.
  - Ensure all dependencies are correctly installed and up-to-date by reviewing `go.mod`, `package.json`, and lock files.

- **Test Failures**:
  - Review the test output to identify which tests are failing.
  - Verify that the test environment (e.g., PostgreSQL and MinIO services) is properly configured and accessible.

- **Linting Issues**:
  - Examine the linter output to identify code style or quality issues.
  - Fix any reported errors or warnings in the code to meet the project's linting standards.

- **Deployment Failures**:
  - Confirm that the Docker Hub credentials (`DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`) are correctly set in GitHub Secrets.
  - Ensure that the SSH credentials (`DC_HOST`, `DC_USER`, `DC_PASS`, `DC_PORT`) are correctly set in GitHub Secrets.
  - Verify that the server has Git and Docker Compose installed.
  - Check the SSH connection and permissions on the server.
  - Ensure the Docker image builds successfully locally before pushing to the repository.

## Example Workflow File

Below is an example of the GitHub Actions workflow file that implements this pipeline:

```yaml
name: CI and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: yourusername/yourimage:latest

  deploy:
    name: Deploy to Server via SSH
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - name: SSH into server and deploy
        uses: cross-the-world/ssh-scp-ssh-pipelines@latest
        env:
          REPO_URL: ${{ github.repositoryUrl }}
          PROJECT_DIR: /home/user/prod
        with:
          host: ${{ secrets.DC_HOST }}
          user: ${{ secrets.DC_USER }}
          pass: ${{ secrets.DC_PASS }}
          port: ${{ secrets.DC_PORT }}
          connect_timeout: 10s
          first_ssh: |
            echo "Cloning repository..."
            rm -rf ${{ env.PROJECT_DIR }}
            git clone ${{ env.REPO_URL }} ${{ env.PROJECT_DIR }}
            cd ${{ env.PROJECT_DIR }}
            echo "Running docker compose down..."
            docker compose down -f docker-compose-prod.yml
            echo "Running docker compose up -d..."
            docker compose -f docker-compose-prod.yml up -d
```
