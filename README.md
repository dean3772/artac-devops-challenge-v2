# Sentiment Analysis API

A small, production-oriented FastAPI service that serves a pre-trained scikit-learn
sentiment classifier. The service exposes liveness and readiness probes and a single
prediction endpoint, ships as a container image, and includes Terraform for provisioning
it on a single AWS EC2 instance. Build, test, security scanning, and image publishing are
automated with GitHub Actions.

---

## Overview

The service loads a serialized TF-IDF + Logistic Regression pipeline at startup and
serves sentiment predictions over HTTP. It is designed to run as a non-root container,
report readiness only once the model is loaded, and pass through a CI/CD pipeline that
validates the image before it is published.

| Endpoint   | Method | Description                                                                 |
| ---------- | ------ | --------------------------------------------------------------------------- |
| `/predict` | POST   | Accepts `{"text": "..."}`, returns `{"sentiment": "...", "confidence": ...}` |
| `/health`  | GET    | Liveness probe. Returns 200 if the server process is running.               |
| `/ready`   | GET    | Readiness probe. Returns 200 only after the model is loaded.                 |
| `/docs`    | GET    | Swagger UI for interactive exploration.                                     |

The distinction between `/health` and `/ready` matters for this service: the process
can be running before the model has finished loading, so readiness is the safer signal
for traffic and is used by the container healthcheck.

---

## Requirements

- Docker (and the Docker Compose plugin) for the containerized workflow.
- Python 3.12 for running the service or tests directly on the host.
- Terraform 1.5 or newer and AWS credentials if you want to run `terraform plan` or
  `terraform apply` against the configuration. See [Infrastructure](#infrastructure) for
  how the committed plan artifact was produced.

---

## Running locally

### With Docker Compose (recommended)

The simplest way to build and run the service locally:

```bash
docker compose up --build
```

Then open:

- Swagger UI: http://localhost:8080/docs
- Liveness: http://localhost:8080/health
- Readiness: http://localhost:8080/ready

Useful commands:

```bash
docker compose ps          # status, including container health
docker compose logs -f     # follow logs
docker compose down        # stop and remove
```

Compose builds the same production image defined in the `Dockerfile` and applies the
same readiness healthcheck used by the pipeline, so the local run mirrors the published
artifact.

### With Docker directly

```bash
docker build -t sentiment-api:local .
docker run -d --name sentiment-api -p 8080:8080 sentiment-api:local
```

Verify the endpoints:

```bash
curl -fsS http://localhost:8080/health
curl -fsS http://localhost:8080/ready
curl -fsS -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"This movie was fantastic!"}'
```

Clean up:

```bash
docker rm -f sentiment-api
```

### Without Docker

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8080
```

---

## Running tests

Tests use the development/test dependencies, which include the runtime dependencies plus
pytest and httpx:

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements-dev.txt
python -m pytest tests/ -v
```

The suite covers model loading, the three endpoints, and request validation behavior.

---

## Container image

The image is a multi-stage build on `python:3.12-slim`:

- A builder stage installs runtime dependencies into an isolated virtual environment.
- The runtime stage copies only that environment plus `app/` and `models/`, installs the
  one native library scikit-learn needs at runtime, and runs as a dedicated non-root user.
- `PYTHONUNBUFFERED` and `PYTHONDONTWRITEBYTECODE` are set for predictable container logging
  and a cleaner filesystem.
- The `HEALTHCHECK` validates `/ready`, so the container is only reported healthy once the
  model is serving.

A `.dockerignore` keeps the build context and final image limited to what the runtime
needs.

---

## CI/CD pipeline

The pipeline is defined in `.github/workflows/ci-cd.yml` and covers continuous integration
and continuous delivery. It runs on pushes and pull requests to `main` and is structured
around the lifecycle of a single image artifact rather than splitting that lifecycle across
jobs.

1. **Test.** Sets up Python, installs the dev/test dependencies, and runs the test suite.
   This is fast Python validation with no Docker involved.
2. **Build, smoke test, scan, and push.** Runs only after tests pass:
   - Builds the image once and loads it into the local Docker daemon.
   - Starts the container and smoke tests `/health`, `/ready`, and `/predict` against the
     real runtime image.
   - Scans that same image with Trivy (HIGH and CRITICAL, `ignore-unfixed`).
   - On pushes to `main`, tags and pushes the validated image to GitHub Container Registry.
   - The image is published only after it passes the smoke test and the scan, so the
     registry holds artifacts that cleared the quality gates.
3. **Deploy.** A documented dry-run template. Since no real EC2 host or deployment secrets
   are configured in this environment, the job does not execute a deployment. Instead it
   prints the exact flow a real deployment would follow: the required secrets, writing the
   SSH key with a non-interactive host-key policy, pulling the validated image, replacing the
   running container idempotently, and waiting on `/ready` before reporting success. This
   keeps the pipeline honest about what was actually run while still documenting the intended
   path to a live service. The reasoning for preferring AWS SSM Session Manager over inbound
   SSH in production is covered in `ASSESSMENT.md`.

### Verifying the published image

After a successful pipeline run on `main`, the image can be pulled and exercised locally:

```bash
docker pull ghcr.io/dean3772/artac-devops-challenge-v2:latest
docker run -d --name sentiment-check -p 8080:8080 \
  ghcr.io/dean3772/artac-devops-challenge-v2:latest
curl -fsS http://localhost:8080/ready
curl -fsS -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"This movie was fantastic!"}'
docker rm -f sentiment-check
```

---

## Infrastructure

The `terraform/` directory provisions a single EC2 instance and a security group. The
instance boots, installs Docker via user-data, pulls the published image, runs the
container, and waits for `/ready` before reporting completion. The user-data script is
idempotent: it removes any existing container of the same name before starting a new one.

### Terraform plan

The repository includes `terraform/plan-output.txt` as the reviewed plan artifact
(`Plan: 2 to add, 0 to change, 0 to destroy`).

The committed Terraform configuration uses the normal AWS provider configuration. In normal
usage, configure AWS credentials before running `terraform plan` or `terraform apply`:

```bash
cd terraform
terraform init
terraform plan \
  -var="docker_image=ghcr.io/dean3772/artac-devops-challenge-v2:latest" \
  -var="ssh_key_name=your-key-pair-name" \
  -out=tfplan
terraform show -no-color tfplan > plan-output.txt
```

For this review, the included `plan-output.txt` was generated without applying real
infrastructure, by temporarily adding dummy provider credentials and provider
skip-validation settings to generate the plan offline, then reverting the provider block
before commit. The committed provider block is the normal AWS configuration.

### Applying the configuration

A real apply requires:

- AWS credentials configured in the environment (for example via `aws configure` or
  environment variables).
- A real EC2 key pair name that exists in the target region, passed as `ssh_key_name`.
- The published image reference, passed as `docker_image`.

Copy `terraform/terraform.tfvars.example` to `terraform.tfvars`, fill in the values, then
run `terraform apply`. The outputs include the instance public IP, an application URL, and
an SSH command.

Two things to be aware of before applying:

- The AMI is pinned for reproducibility and is region-specific. If you change `aws_region`,
  update the AMI ID accordingly, or replace it with a controlled AMI lookup for the new
  region.
- The image must be reachable from the EC2 instance. If the GHCR package is private, either
  make it public for the demo or add registry authentication to the user-data or deploy
  flow, otherwise the `docker pull` on the instance will fail.

---

## Future production improvements

This setup is intentionally simple so it can be validated within free-tier limits. For a
production deployment, the main directions would be:

- **Network isolation.** Define a dedicated VPC, subnets, and route tables rather than
  relying on the account default VPC, and place the instance in a subnet with explicit
  ingress and egress control instead of broad defaults.
- **Ingress and TLS.** Place the service behind an Application Load Balancer, terminate TLS,
  attach a domain, and restrict the instance so only the load balancer reaches the app port.
- **Access.** Replace open inbound SSH with AWS Systems Manager Session Manager, or at least
  restrict SSH to a trusted CIDR or a bastion.
- **Compute model.** For a single container, a managed runtime such as AWS App Runner or
  ECS on Fargate would remove host patching and provide managed scaling and health checks,
  without the overhead of a full Kubernetes platform.
- **Observability.** Attach an IAM role with the CloudWatch agent, ship the user-data log and
  container logs to CloudWatch Logs, and add basic metric alarms.
- **State and supply chain.** Move Terraform to an S3 backend with locking for team use,
  refactor the configuration into reusable modules, and pin the base image by digest
  alongside an automated update process such as Renovate or Dependabot.
- **Cost and sizing.** Review expected traffic, instance size, image pull frequency, log
  retention, and managed-service pricing before choosing between EC2, App Runner, ECS, or
  another runtime.

These are described as direction rather than implemented here, to keep the committed
configuration to what could be validated without paid infrastructure.

---

## Technical review documents

This repository includes additional documents that capture the engineering review behind
the current state of the code:

- `ASSESSMENT.md` is a review of the inherited Docker, CI/CD, and Terraform setup. Each item
  is classified as a bug, an intentional trade-off, or a needs-improvement item, with the
  reasoning and the action taken.
- `DECISIONS.md` is the original decision log, preserved for context.
- `AI_WORKFLOW.md` is a set of notes on how AI assistance was used and validated during the
  work.

---

## Project structure

```
app/                        # FastAPI application (model loading, endpoints, schemas)
models/                     # Pre-trained serialized model
tests/                      # API and model tests
terraform/                  # EC2 + security group, user-data, plan output
.github/workflows/ci-cd.yml # CI/CD pipeline
Dockerfile                  # Multi-stage production image
docker-compose.yml          # Local development convenience
.dockerignore               # Build context exclusions
requirements.txt            # Runtime dependencies
requirements-dev.txt        # Runtime + test/dev dependencies
```