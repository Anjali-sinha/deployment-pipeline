# Automated Multi-Environment Deployment Pipeline

## 1. What this project includes

Concepts used:

- **Git and GitHub:** track code changes, work with branches, and review changes through pull requests.
- **Continuous Integration (CI):** run automated tests when code is pushed or a pull request is opened or updated.
- **Docker:** package the application and its runtime dependencies into a portable image.
- **Container Registry:** publish versioned Docker images to GitHub Container Registry (GHCR).
- **Continuous Delivery/Deployment (CD):** make a tested image available to staging and production environments.
- **AWS EC2:** run the application on a Linux virtual machine.
- **AWS Systems Manager (SSM):** execute deployment commands on a managed EC2 instance without requiring GitHub-hosted runners to connect directly over SSH.
- **Deployment approvals:** require a human approval before a production deployment proceeds.

## 2. High-level architecture

```text
Developer changes code
        |
        v
Feature branch -> Pull request
        |
        v
GitHub Actions: test job
  - Checkout code
  - Set up Python
  - Install test dependencies
  - Run pytest
        |
        +---- Failure: stop; fix the code/tests
        |
        v
Merge/push to develop or main
        |
        v
GitHub Actions: build-and-push job
  - Log in to ghcr.io using GITHUB_TOKEN
  - Build the Docker image
  - Tag it with the commit SHA
  - Push it to GitHub Container Registry
        |
        +-----------------------------+
        |                             |
        v                             v
Push to develop                 Push to main
        |                             |
        v                             v
Staging deployment job          Production deployment job
(automatic when configured)     (GitHub Environment approval)
        |                             |
        v                             v
AWS EC2 staging                 AWS EC2 production target
via SSM                         via SSM
```

The intended deployment path uses GitHub Actions to authenticate to AWS with **OpenID Connect (OIDC)** and obtain temporary AWS credentials. SSM then asks the EC2 instance to pull the image and start the container. This avoids placing permanent AWS access keys in GitHub. OIDC and the appropriate IAM trust/permissions must already be configured by an authorized administrator; a workflow file alone does not grant AWS access.

### What runs where?

| Component | Where it runs | Purpose |
|---|---|---|
| Flask app and pytest tests | Your Windows laptop during development; GitHub runner during CI | Application and correctness checks |
| Git | Your laptop and GitHub | Tracks source history |
| GitHub Actions runner | GitHub-hosted temporary Ubuntu VM | Executes tests, builds images, and runs workflow steps |
| Docker image | Built by Docker locally and by the workflow | Packaged application artifact |
| GHCR (`ghcr.io`) | GitHub Container Registry | Stores the Docker image |
| EC2 | AWS Linux instance | Runs the deployed application container |
| SSM | AWS Systems Manager | Sends commands to the EC2 instance |
| GitHub Environment `production` | GitHub repository settings | Holds protection rules, including required approval if available/configured |

## 3. Application overview

The project uses a small Python Flask API so the focus can stay on the deployment workflow rather than application complexity.

| Endpoint | Expected response | Why it exists |
|---|---|---|
| `GET /` | JSON welcome message | Confirms the application responds |
| `GET /health` | `{"status":"healthy"}` | Basic deployment/health verification |
| `GET /version` | `{"version":"1.0.0"}` | Shows the application version |

The tests in `test_app.py` check the status code and response data for these endpoints. A health endpoint is useful for a basic smoke test after deployment, but it is not a complete production-readiness or monitoring system.

## 4. Repository layout

```text
deployment-pipeline/
├── app.py                       # Flask application and endpoints
├── test_app.py                  # Automated tests
├── requirements.txt             # Runtime dependencies (Flask, Gunicorn)
├── requirements-dev.txt         # Runtime dependencies plus pytest
├── Dockerfile                   # Instructions for building the image
├── .gitignore                   # Files Git should ignore
├── .dockerignore                # Files excluded from Docker build context
├── README.md                    # Project documentation
└── .github/
    └── workflows/
        └── ci.yml                # CI, image publishing, deployment jobs
```

## 5. Branching approach

The project uses three kinds of branches:

- **`feature/*`:** make a focused change, such as `feature/update-health-endpoint`.
- **`develop`:** integration branch; intended to publish an image and deploy to staging after changes are merged.
- **`main`:** release branch; intended to publish an image and deploy to production after the release is merged and approval requirements are satisfied.

Typical development cycle:

1. Create a feature branch from `develop`.
2. Make a change and run tests locally.
3. Commit and push the feature branch.
4. Open a pull request targeting `develop`.
5. Review the CI result and fix failures before merging.
6. Confirm the new version on staging.
7. When the release is ready, open a pull request from `develop` to `main`.
8. After merge, approve the production deployment in GitHub if the environment is configured to require approval.

Avoid committing every change directly to `main`. Keep `main` releasable.

## 6. Run locally on Windows

Use PowerShell. These commands assume Python and Git are installed and Docker Desktop is running.

### Create and activate a virtual environment

```powershell
cd "$HOME\Documents\deployment-pipeline"
py -m venv .venv
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, you can use the environment's Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\.venv\Scripts\python.exe -m pytest -v
```

### Install dependencies and run tests

```powershell
python -m pip install -r requirements-dev.txt
python -m pytest -v
```

Expected result: all three endpoint tests pass.

### Run the application directly

```powershell
python app.py
```

Open `http://localhost:5000/health` in a browser. Stop the server with `Ctrl+C`.

## 7. Docker: build and run locally

The `Dockerfile` starts from a Python slim image, installs runtime dependencies, copies `app.py`, and starts the Flask application with Gunicorn.

Build the image from the repository root:

```powershell
docker build -t deployment-demo:1.0.0 .
```

Run it:

```powershell
docker run -d --name deployment-demo -p 5000:5000 deployment-demo:1.0.0
```

Test it at `http://localhost:5000/health`.

Useful commands:

```powershell
docker ps
docker logs deployment-demo
docker stop deployment-demo
docker rm deployment-demo
```

Concepts:

- An **image** is the packaged template for the application.
- A **container** is a running instance of an image.
- `-p 5000:5000` maps port 5000 on your laptop to port 5000 in the container.
- `--name` gives the container a readable name.

## 8. What the GitHub Actions workflow does

The workflow lives at `.github/workflows/ci.yml`. GitHub automatically discovers workflow YAML files under `.github/workflows/`.

### Triggers

The current design listens for:

- Pushes to `main` and `develop`.
- Pull requests targeting `main` and `develop`.

### Job 1: `test`

This job runs on a temporary GitHub-hosted Ubuntu runner and:

1. Checks out the repository with `actions/checkout`.
2. Sets up Python with `actions/setup-python`.
3. Installs dependencies from `requirements-dev.txt`.
4. Runs `python -m pytest -v`.

If a test fails, the job fails. Because the image job declares `needs: test`, image publishing does not proceed when tests fail.

### Job 2: `build-and-push`

This job runs only for **push events**, after `test` succeeds. It:

1. Uses `docker/login-action` to authenticate to GHCR with the workflow's `GITHUB_TOKEN`.
2. Uses `docker/build-push-action` to build the Docker image from the repository's `Dockerfile`.
3. Pushes the image to GHCR.
4. Tags the image with `${{ github.sha }}`.

The image name used in this project is:

```text
ghcr.io/anjali-sinha/deployment-pipeline:<FULL_COMMIT_SHA>
```

Use the actual lowercase namespace and image name configured in `ci.yml`. `github.sha` is the commit identifier, so the image points to a specific source revision instead of a moving `latest` tag.

The `packages: write` permission is needed for publishing. The `GITHUB_TOKEN` is provided for the workflow; it is not a personal GitHub password. The package can be private, in which case the EC2 host needs authentication to pull it.

### Job 3: `deploy-staging` (intended automation)

The intended condition is a push to `develop`, after `build-and-push` succeeds. The job authenticates to AWS, asks SSM to run deployment commands on the staging instance, and checks the `/health` endpoint.

The deployment commands are intended to:

1. Pull the image tagged with the triggering commit SHA.
2. Remove the old staging container.
3. Start the new container with port 5000 mapped to container port 5000.
4. Retry the health endpoint briefly and report failure if it never becomes healthy.

**Current limitation:** this job is not operational until the required GitHub repository variables are set and the selected AWS role's trust policy and permissions authorize GitHub Actions to use OIDC and send SSM commands. An EC2 role that lets the instance register with SSM does not automatically grant GitHub Actions permission to call SSM.

### Job 4: `deploy-production` (intended automation)

The intended condition is a push to `main`, after image publishing succeeds. The job references the GitHub Environment `production`. If that environment has an enabled required-reviewer protection rule, GitHub pauses the job until an authorized reviewer approves the deployment.

The example production demo can use the same EC2 instance but a separate container and host port 5001. That is a **demonstration only**, not real production isolation: both environments share the same machine, network, resources, and failure domain. A separate production host/network should be used for a more realistic setup.

**Current limitation:** like staging, production requires approved AWS OIDC and SSM permissions, the correct repository variables, and production environment protection rules. Merely writing `environment: production` does not itself guarantee a manual approval.

## 9. GitHub repository configuration

Depending on which deployment jobs are enabled, the workflow expects these repository variables:

| Variable | Meaning |
|---|---|
| `AWS_REGION` | Region containing the EC2 instance |
| `STAGING_INSTANCE_ID` | Instance ID used for staging deployment |
| `PROD_INSTANCE_ID` | Instance ID used for production deployment; may equal staging only in the learning demo |
| `AWS_DEPLOY_ROLE_ARN` | ARN of the approved role GitHub Actions can assume using OIDC |

Add variables under **Repository → Settings → Secrets and variables → Actions → Variables**. The role ARN and region are identifiers/configuration, not credentials. Never store a private SSH key, AWS access key, GitHub token, or password in a plain repository variable.

The OIDC deployment role requires two separate configurations:

1. **Trust policy:** allows the intended GitHub repository/workflow subject to assume the role.
2. **Permissions policy:** grants only the required actions, such as sending the permitted SSM command to the approved instance and reading its result.

The organization's permission boundary and service control policies may still deny these actions. Do not try to bypass an organizational restriction; use an existing approved role or leave automated AWS deployment disabled.

## 10. Manual EC2 deployment (verified path)

The manual deployment used for initial validation was:

1. Connect to the Ubuntu EC2 instance.
2. Confirm Docker is installed and running.
3. Authenticate to GHCR if the package is private.
4. Pull the exact image tag published by a successful workflow run.
5. Start the container and check its health endpoint.

Example commands to run **on the EC2 instance**, replacing the tag with the actual full commit SHA:

```bash
sudo docker pull ghcr.io/anjali-sinha/deployment-pipeline:FULL_COMMIT_SHA
sudo docker run -d \
  --name deployment-demo \
  --restart unless-stopped \
  -p 5000:5000 \
  ghcr.io/anjali-sinha/deployment-pipeline:FULL_COMMIT_SHA

curl -fsS http://localhost:5000/health
```

If a container with the same name already exists, inspect it before replacing it. The simple initial deployment commands do not provide a rollback mechanism.

## 11. Security and cost notes

- Never commit `.pem` private keys, tokens, passwords, or `.env` secrets.
- Keep the EC2 SSH security-group rule limited to your own trusted IP when manual SSH is needed. Do not expose SSH to `0.0.0.0/0` simply to accommodate GitHub-hosted runners.
- For automated deployment, prefer OIDC plus SSM with narrowly scoped permissions over long-lived AWS access keys or a public SSH rule.
- Keep a private GHCR package authenticated on the server. Avoid printing tokens in logs.
- Port 5000 (and 5001 if used by the demo) is for learning. A public production application should use HTTPS, an appropriate reverse proxy or load balancer, and a reviewed security design.
- An EC2 instance, storage, public IPv4 address, data transfer, and other AWS resources may incur charges. Verify your organization's rules and pricing, and stop/terminate resources when no longer needed if permitted.
- The initial deploy script replaces the existing container before fully verifying the new version. Add health-based rollback before treating this as production-grade.

## 12. Troubleshooting guide

| Symptom | Things to check |
|---|---|
| Local tests fail | Check the failure output, confirm dependencies are installed, and ensure expected response values match `app.py`. |
| GitHub test job fails | Open the failed step in Actions; check Python version, dependency installation, file names, and test output. |
| Docker says repository name must be lowercase | Use a lowercase registry namespace and image name. Tags can contain the commit SHA. |
| GHCR push is denied | Check `packages: write`, the `GITHUB_TOKEN` login step, and package/repository settings. |
| EC2 cannot pull image | Check image name/tag, registry authentication if private, outbound internet access, and Docker logs. |
| Browser cannot reach port 5000 | Check the container is running, its port mapping, EC2 security-group inbound rule, public IP, and network route. Do not open access more broadly than necessary. |
| SSM target is Offline | Check instance role/SSM registration, SSM Agent status, AWS Region, and outbound HTTPS connectivity. |
| GitHub Actions cannot assume AWS role | Check OIDC provider, exact `sub` and `aud` trust conditions, repository/environment identity, role ARN, and organization permission boundaries. |
| SSM returns AccessDenied | The caller's effective IAM permissions may not allow `ssm:SendCommand` or reading command results; check the document and instance resource scope. |
| Deployment job is skipped | Check its event/branch condition and whether required repository variables are present. A skipped job is not a successful deployment. |

## 13. Glossary

- **CI (Continuous Integration):** automatically checks changes, usually by building and testing them.
- **CD (Continuous Delivery/Deployment):** automates preparing or releasing a change to an environment.
- **Workflow:** the YAML automation definition in GitHub Actions.
- **Runner:** the machine that executes a workflow job.
- **Artifact:** a build output that can be used later. Here, the Docker image is the deployable artifact.
- **Registry:** storage for packaged artifacts; here, GHCR stores Docker images.
- **Image tag:** a label identifying a particular image version. This project uses a commit SHA.
- **Environment:** a deployment target such as staging or production.
- **OIDC:** a federated identity mechanism that lets GitHub Actions obtain temporary AWS credentials without a stored long-lived AWS access key.
- **IAM role:** an AWS identity with a trust policy and permissions policies.
- **SSM Run Command:** a Systems Manager feature for running commands on managed instances.
- **Health check/smoke test:** a quick check used to verify that a service responds after deployment.
- **Rollback:** returning to a known-good version after a failed or unhealthy deployment.

## 14. Suggested future improvements

After the core flow works, improve the project in small steps:

1. Add branch protection and require CI checks before merging to `main`.
2. Add a health check and safer rollback that preserves the last working image/container.
3. Use a separate production host or isolated environment instead of sharing the staging EC2 instance.
4. Add HTTPS with a domain and reverse proxy/load balancer if you want a publicly accessible demo.
5. Add container scanning and dependency/security checks.
6. Add logs and basic monitoring/alerting.
7. Record screenshots of successful CI, GHCR publishing, staging deployment, and the production approval gate for the portfolio.

## 15. Useful references

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [GitHub Container Registry documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [GitHub Actions starter workflows](https://github.com/actions/starter-workflows/tree/main)
- [GitHub Actions OIDC with AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [AWS Systems Manager Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
- [Flask testing documentation](https://flask.palletsprojects.com/en/stable/testing/)
