Finding 1: scikit-learn runtime version does not match the serialized ML model

What I found:
The Docker image built successfully and the FastAPI service started. /health, /ready, and /docs were reachable and returned 200 OK. However, /predict initially returned 500 Internal Server Error even with a valid request body:
{
  "text": "This movie was fantastic!"
}

The Docker logs showed that the model file was loaded, but scikit-learn emitted InconsistentVersionWarning messages. The model was serialized with scikit-learn 1.8.0, while the container installed scikit-learn 1.6.1 from requirements.txt. The actual failure happened during predict_proba:
AttributeError: 'LogisticRegression' object has no attribute 'multi_class'

Classification:
Bug.

Contractor's reasoning:
DECISIONS.md mentions that scikit-learn was pinned to 1.6.1 for stability, because newer versions may introduce breaking API changes. I disagree with this decision in the current codebase, because the provided model was already generated with scikit-learn 1.8.0. Pinning an older runtime version made the application start successfully but fail during actual prediction.

What I did:
I updated requirements.txt to use the scikit-learn version compatible with the provided model, then rebuilt and reran the Docker image. I did not modify app/ or models/, because the assignment says to treat the application as a black box. After rebuilding, the model loaded without version warnings and /health, /ready, and /predict all returned 200 OK.

Why:
For pickled ML models, the runtime library version should match the version used when the model was serialized. Fixing the dependency pin is safer and cleaner than changing application code or regenerating the model.

Finding 2: Dockerfile uses a single-stage build

What I found:
The inherited Dockerfile uses a single-stage build. The image builds and the service can run, but build-time and runtime concerns are not separated. For a small proof-of-concept this is acceptable, but for a production container it is better to keep the final runtime image as small and focused as possible.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that a multi-stage build was tried, but the model did not load correctly in the final slim stage because some shared libraries for scikit-learn were missing. The contractor reverted to a single-stage build to keep the application working. I agree this was a reasonable short-term trade-off during the initial setup, but I do not think it should remain the final production approach without being revalidated.

Action taken:
I replaced the single-stage Dockerfile with a multi-stage build. The build stage installs the Python runtime dependencies into an isolated virtual environment, and the final stage copies only that environment plus the application files required to serve the API. After the change, I rebuilt the image and verified that /health, /ready, and /predict all return 200 OK.

Why:
A multi-stage Dockerfile creates a cleaner separation between build-time and runtime concerns. It helps reduce the final image size, avoids carrying unnecessary files or build-time artifacts into production, and provides a better reusable pattern for future services.

Finding 3: Docker image uses the full python:3.12 base image

What I found:
The inherited Dockerfile uses python:3.12 as the base image. This works, but it produces a larger runtime image than necessary. For production, a smaller runtime image is usually preferable when it can still support the required native dependencies.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that python:3.12-slim caused issues with native dependencies during pip installation or model loading, so the contractor used the full Python image because it “just works”. I agree with the goal of keeping the service working, especially because scikit-learn can require native libraries, but I disagree that the full image should be kept without another validation attempt.

Action taken:
I changed the runtime image to python:3.12-slim and added the required native runtime library for scikit-learn. After rebuilding the image, the model loaded successfully and /health, /ready, and /predict returned 200 OK.

Why:
Using a slimmer runtime image reduces image size, pull time, storage usage, and unnecessary operating-system surface area. Since this service depends on a serialized scikit-learn model, the change was validated with real inference through /predict, not only by checking that the container starts.

Finding 4: Missing .dockerignore file

What I found:
The repository did not include a .dockerignore file. Without it, Docker sends unnecessary files from the repository into the build context. This can include files such as .git, local notes, caches, test artifacts, editor metadata, Terraform files, CI configuration, and other files that are not required for the application runtime.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md does not mention this.

Action taken:
I added a .dockerignore file to reduce the Docker build context and prevent unnecessary repository files from being sent to the Docker daemon.

Why:
A .dockerignore file improves build performance, reduces accidental leakage of local or repository metadata, and makes the image build process more predictable. It is a basic production Docker hygiene practice.

Finding 5: Dockerfile copies the entire repository into the image

What I found:
The inherited Dockerfile uses COPY . . after installing dependencies. This copies the full repository into the container image, even though the runtime only needs the application code, model files, and runtime dependency definitions. This can accidentally include tests, Terraform code, CI/CD files, documentation, local notes, Git metadata if not ignored, and other non-runtime files.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md does not mention this. It was likely done because it is simple and common during early development.

Action taken:
I changed the Dockerfile to copy only the files required by the runtime: app/, models/, and the runtime requirements file. Non-runtime files are not copied into the final production image.

Why:
A production image should contain only what it needs to run. Copying the whole repository makes the image less predictable, increases size, and can accidentally expose files that are irrelevant or sensitive. Explicit COPY instructions also make the Dockerfile easier to review.

Finding 6: Runtime and development/test dependencies are not separated

What I found:
The inherited repository uses a single requirements.txt file for both runtime dependencies and test/development dependencies. For example, pytest and httpx are useful for tests, but they are not required inside the final production container that serves the API.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md does not mention dependency separation. The single requirements file was likely chosen for simplicity.

Action taken:
I kept requirements.txt as the production/runtime dependency file and added requirements-dev.txt for local development and CI test tooling. The production Docker image installs only requirements.txt. CI and local test workflows can install requirements-dev.txt when they need pytest and test-related packages.

Why:
Separating runtime and development dependencies keeps the production image smaller and cleaner. It also makes the dependency intent clearer: requirements.txt is used to run the service, while requirements-dev.txt is used for local validation and CI. If the project grows, this can be split further into separate development and test requirement files, but for this assignment a runtime file plus a dev/test file is a good balance.

Finding 7: Docker healthcheck uses /health instead of readiness validation

What I found:
The inherited Dockerfile uses the /health endpoint for Docker HEALTHCHECK. /health confirms that the FastAPI process is running, but it does not prove that the ML model is loaded and the service is ready to handle prediction traffic. The application also exposes /ready, which is specifically intended to return 200 only after the model has loaded.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that /health was used for the Docker HEALTHCHECK and CI smoke test because both /health and /ready return 200 and /health is the standard name. I agree that /health is a reasonable liveness check, but I disagree that it is equivalent to /ready for this application. The main business function depends on the model being loaded.

Action taken:
I changed the Docker HEALTHCHECK to call /ready instead of /health. In platforms that support separate probe types, I would use /health for liveness and /ready for readiness. Since Docker HEALTHCHECK provides only one container health signal, /ready gives a safer production signal for this ML API.

Why:
A service can be alive but not ready. In this application, the process may be running while the model is not yet available. Using /ready helps avoid marking the container healthy before it can actually serve prediction traffic.

Finding 8: Docker base image is not pinned by digest

What I found:
The Dockerfile uses a readable Python base image tag, but it does not pin the base image by digest. Tags are convenient, but they can move over time. This means the same Dockerfile may not always build from the exact same base image in the future.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md does not mention base image digest pinning. The contractor focused on choosing a Python image that worked with the ML dependencies.

Action taken:
I documented digest pinning as a recommended production hardening step, but did not pin the digest in this assignment. In a real production pipeline, I would pin the base image by digest together with an automated update process such as Renovate or Dependabot.

Why:
Digest pinning improves reproducibility and makes future debugging easier when base image changes introduce unexpected behavior. The trade-off is that digest pinning can also freeze old base images and delay security updates unless an automated update workflow exists. For production, the stronger pattern is digest pinning plus automated updates.

Finding 9: Python runtime environment is not explicitly configured for containers

What I found:
The inherited Dockerfile does not set common Python container runtime environment variables such as PYTHONUNBUFFERED and PYTHONDONTWRITEBYTECODE.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md does not mention Python runtime environment variables.

Action taken:
I added PYTHONUNBUFFERED=1 and PYTHONDONTWRITEBYTECODE=1 to the Dockerfile.

Why:
PYTHONUNBUFFERED=1 ensures logs are written immediately, which is useful for Docker logs and production observability. PYTHONDONTWRITEBYTECODE=1 prevents Python from writing .pyc files at runtime, keeping the container filesystem cleaner.

Finding 10: Pipeline can publish the Docker image before tests and security validation pass

What I found:
The inherited pipeline starts with a Docker image build job. On push to main, this job can also push the image to GitHub Container Registry before the test and security scan jobs complete. The test and security scan jobs depend on the build job, which means validation happens after the image has already been built and potentially published.

Classification:
Bug.

Contractor's reasoning:
DECISIONS.md mentions that the pipeline was split into separate jobs for build, test, security scan, and deploy. I agree with the idea of separating pipeline responsibilities, but I disagree with the current validation order. A production pipeline should not publish an image before the required quality gates pass.

Action taken:
I changed the pipeline so unit tests run first. The Docker image is built only after tests pass, and it is pushed to the registry only after the image passes smoke testing and vulnerability scanning.

Why:
CI/CD should prevent bad artifacts from being published. The registry should contain artifacts that passed the expected quality gates. Publishing before validation creates a risk where a broken or vulnerable image is available even if the pipeline later fails.

Finding 11: Pipeline job structure does not match the Docker artifact lifecycle

What I found:
The inherited pipeline splits build, test, security scan, and deploy into separate jobs. Separate jobs are useful when work can run in parallel, needs different runners, permissions, environments, or approval gates. However, in this pipeline the Docker image is a single artifact that should move through a sequential lifecycle: build, run, smoke test, scan, and then push. Splitting this lifecycle incorrectly caused the image to be pushed too early and forced the security scan to rebuild a separate image.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that the pipeline was split into separate jobs for build, test, security scan, and deploy. I agree with separating jobs by purpose, but I disagree with the way the Docker image lifecycle was split. The split looked organized, but it did not preserve artifact consistency.

Action taken:
I kept a separate test job because it has a clear purpose: fast Python validation without Docker. I moved the Docker image lifecycle into a single image job with sequential steps: build the image, run the container, smoke test the endpoints, run the vulnerability scan, and push only after validation passes. Another valid production approach would be to keep more separate jobs and pass the built image between them using workflow artifacts, but for this small service the single image job is simpler, avoids duplicate builds, and keeps the same local image available across all image validation steps.

Why:
Jobs should represent meaningful boundaries. For this repository, testing Python code and validating the Docker image are different purposes, so two jobs make sense. But the Docker build, smoke test, scan, and push operate on the same image artifact, so keeping them as sequential steps in the same job gives clearer artifact flow, simpler logs, and less unnecessary CI complexity.

Finding 12: Pipeline does not smoke test the built Docker image before publishing

What I found:
The inherited pipeline builds a Docker image but does not run the resulting container and verify the API endpoints before publishing. Unit tests are useful, but they do not prove that the Docker image itself works with the final runtime filesystem, installed native libraries, non-root user, exposed port, healthcheck, and CMD.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that /health was used for the CI smoke test, but the current pipeline does not actually run the built container and test the service endpoints before publishing.

Action taken:
I added a Docker smoke test after the image build. The pipeline starts the built container and validates /health, /ready, and /predict before the image is published.

Why:
The Docker image is the production artifact. Since this application depends on a serialized ML model and native scikit-learn dependencies, it is important to validate the final container, not only the Python code. This catches issues such as missing shared libraries, wrong COPY instructions, broken healthchecks, wrong ports, and model loading failures.

Finding 13: Security scan rebuilds a separate image instead of scanning the validated image

What I found:
The inherited security scan job builds a separate image tagged for scanning. This means the scan does not necessarily run against the exact same image that was built and potentially published. It also duplicates Docker build work.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that a Trivy scan was added, but does not mention artifact consistency between build, scan, and publish steps.

Action taken:
I changed the pipeline so the image is built once and loaded into the local Docker daemon. The same image is then smoke-tested, scanned by Trivy, tagged for GitHub Container Registry, and pushed only after validation succeeds.

Why:
A CI pipeline should avoid testing or scanning one artifact while publishing another. Using the same image for smoke testing, vulnerability scanning, and publishing improves confidence and avoids unnecessary duplicate builds.

Finding 14: Trivy severity policy only fails on CRITICAL vulnerabilities

What I found:
The inherited Trivy configuration only fails the pipeline on CRITICAL vulnerabilities. This makes the pipeline easier to pass, but it may allow HIGH severity vulnerabilities to pass without review.

Classification:
Intentional Trade-off.

Contractor's reasoning:
DECISIONS.md mentions that the Trivy configuration was relaxed because vulnerability findings in the base image were blocking the pipeline. I agree that ignore-unfixed can be reasonable, because failing on vulnerabilities with no available fix can create noise. However, I would not keep CRITICAL-only enforcement as the final production policy without review.

Action taken:
I made the scan stricter by checking both HIGH and CRITICAL vulnerabilities while keeping ignore-unfixed enabled. If this becomes too noisy in a real production environment, I would add a documented ignore policy for accepted findings instead of silently allowing important vulnerabilities.

Why:
Security scanning should be useful rather than noisy. A pipeline that fails on every unactionable vulnerability creates alert fatigue, but a pipeline that only gates CRITICAL issues may miss meaningful production risk. A better production approach is to scan for HIGH and CRITICAL vulnerabilities, keep exceptions documented, and review the vulnerability policy regularly.

Finding 15: Deploy job was an inherited placeholder with no real deployment logic

What I found:
The inherited deploy job only echoed TODO messages and did not deploy the image anywhere, so the pipeline was not end-to-end from source to a running service.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md notes the deploy job is stubbed out and still needs wiring to real infrastructure. I agree a stub is acceptable during early setup, especially since the assignment does not require an actual AWS deployment, but a bare TODO does not show the intended deployment design.

What I did:
Since I do not have AWS credentials or a real EC2 target, and the assignment does not require a live deploy, I replaced the bare TODO with a documented dry-run deployment template. The job runs and prints the intended real deployment flow without executing it: the required secrets, SSH key handling with a non-interactive host-key policy, an idempotent container replacement, and a post-deploy readiness check against /ready. The SSH key approach could also be replaced with a ready-made GitHub Actions SSH helper. In production I would prefer AWS SSM Session Manager over inbound SSH, as explained in Finding 17.

Why:
A bare TODO shows nothing about whether the engineer understood deployment. A non-executing documented template demonstrates the full intended flow and the production considerations, while staying honest that no real deployment was performed and no money was spent.


Finding 16: Stricter vulnerability scanning surfaced a HIGH advisory in the Starlette dependency chain

What I found:
After I tightened the Trivy policy to fail on both HIGH and CRITICAL severities, the pipeline failed during the image security scan. Trivy reported a HIGH severity advisory affecting Starlette, which is pulled in transitively through FastAPI rather than pinned directly in requirements.txt. The advisory had a fixed version available upstream, so this was an actionable finding rather than an unfixable base-image warning.

Classification:
Bug.

Contractor's reasoning:
DECISIONS.md does not mention this dependency advisory. The contractor only mentioned relaxing Trivy to CRITICAL-only because scans were noisy and blocking the pipeline. Relaxing the policy is what hid this HIGH finding in the first place, so the contractor's trade-off had a real cost.

What I did:
I updated the FastAPI pin in requirements.txt so the dependency resolver installs a patched Starlette version that is no longer flagged by the scan. After the change I recreated the local environment, installed requirements-dev.txt, ran the test suite, and rebuilt and smoke-tested the Docker image locally before pushing. As a further hardening step, the transitive dependency could also be pinned explicitly to make the resolved version auditable directly from the requirements file.

Why:
Application dependencies are part of the production runtime surface. A HIGH severity advisory in a web framework dependency should not be ignored when a compatible fixed version exists. This also demonstrates the value of scanning for HIGH and CRITICAL rather than CRITICAL only, since the stricter policy is what surfaced it.

Finding 17: SSH access is open to the internet

What I found:
The Terraform security group allows inbound SSH access on port 22 from 0.0.0.0/0. This means any public IP address can attempt to connect to the EC2 instance over SSH.

Classification:
Needs Improvement.

Contractor's reasoning:
DECISIONS.md mentions that SSH was opened to 0.0.0.0/0 for initial setup and for the CI pipeline to deploy via SSH. I understand this as a temporary setup decision, but I would not keep it as a production-ready security group rule.

Action taken:
I kept the current SSH rule for this assignment because it matches the inherited simple EC2 deployment model and actual cloud deployment is optional. I documented it as a production hardening item. In production, I would restrict SSH to a trusted CIDR, use a VPN or bastion, or preferably use AWS Systems Manager Session Manager instead of direct inbound SSH.

Why:
Open SSH increases the attack surface of the instance. Even with key-based authentication, production infrastructure should avoid exposing SSH to the entire internet unless there is a temporary and clearly controlled reason.

Finding 18: Application port is exposed directly to the internet

What I found:
The Terraform security group allows inbound access to the application port from 0.0.0.0/0. This makes the API directly reachable on the EC2 public IP and app port.

Classification:
Intentional Trade-off.

Contractor's reasoning:
DECISIONS.md does not explicitly discuss the application port exposure, but the Terraform outputs expose an app URL using the EC2 public IP and application port, so the setup appears designed as a simple direct-access deployment.

Action taken:
I kept the direct app port exposure for the assignment because it keeps the deployment simple and avoids requiring paid infrastructure. For production, I would put the service behind an Application Load Balancer, terminate TLS, attach a proper domain name, and restrict the instance security group so only the load balancer can reach the application port.

Why:
Direct EC2 exposure is acceptable for a simple demo or free-tier validation, but it is not the preferred production pattern. A load balancer gives better security boundaries, health checks, TLS support, and future scaling options.

Finding 19: AMI is pinned directly in the EC2 resource

What I found:
The Terraform configuration hardcodes a specific AMI ID directly in the aws_instance resource. This improves reproducibility, but it also ties the configuration to a specific region and can become stale over time.

Classification:
Intentional Trade-off.

Contractor's reasoning:
DECISIONS.md explains that the AMI was pinned because a dynamic latest Ubuntu lookup previously broke the Docker installation script. I agree with the goal of reproducibility, especially for an assignment or a simple single-instance deployment.

Action taken:
I kept the pinned AMI approach for now. For production, I would either expose the AMI as an explicit ami_id variable with clear region-specific documentation, or use a controlled AMI selection process such as a tested golden image or a carefully filtered data source. I would avoid blindly tracking latest.

Why:
Pinning prevents unexpected infrastructure changes, but hardcoding AMI IDs directly inside the resource makes updates and region changes harder. The production goal should be controlled updates, not automatic latest and not permanently stale infrastructure.

Finding 20: Terraform uses local state

What I found:
The Terraform configuration currently uses local state. There is no remote backend configured for shared state storage or locking.

Classification:
Intentional Trade-off.

Contractor's reasoning:
DECISIONS.md mentions that local state was chosen because this is a single-operator deployment, and S3 plus DynamoDB locking was considered overkill at this stage. I agree that local state is acceptable for a small assignment or one-person validation workflow.

Action taken:
I kept local state for this assignment. I did not configure an S3 backend because actual AWS deployment is optional and I do not want to require cloud resources just to review or validate the repository. For a real production/team deployment, I would use an S3 backend with DynamoDB locking, encryption, and restricted IAM access.

Why:
Remote state and locking are important when more than one person or automation process can run Terraform. They prevent state loss, state drift, and concurrent apply conflicts. Local state is fine for isolated local validation, but not for production teamwork.

Finding 21: EC2 bootstrap script is not idempotent and does not verify application readiness

What I found:
The user-data script pulls the Docker image and runs a container named sentiment-api. However, it does not remove an existing container with the same name before running docker run. If the bootstrap or deployment logic is executed again, Docker will fail because a container named sentiment-api already exists. The script also starts the container but does not verify that the application actually becomes ready.

Classification:
Bug.

Contractor's reasoning:
DECISIONS.md does not mention this behavior. The script appears to be written for first EC2 boot only, where the container name conflict would not happen.

Action taken:
I updated user-data.sh to remove any existing sentiment-api container before starting a new one. I also added a readiness loop that checks the /ready endpoint after container startup and writes useful logs if the service does not become ready.

Why:
Deployment and bootstrap scripts should be safe to rerun. A script that fails because an old container exists can break redeployments and make recovery harder. Verifying /ready after startup also catches cases where Docker starts successfully but the application cannot serve traffic.

Additional note: Terraform plan validation evidence

Purpose:
The assignment asks for Terraform validation and a plan output, while actual AWS deployment is optional.

What I did:
I generated terraform/plan-output.txt without applying any real AWS infrastructure. Because real AWS credentials were not available in this local environment, I temporarily used dummy AWS provider credentials together with provider skip-validation settings only for generating the local plan.

Important:
This temporary provider configuration was not kept in the final Terraform code. After generating the plan output, I reverted the provider block back to the normal AWS provider configuration.

Result:
terraform plan successfully generated a plan showing:

Plan: 2 to add, 0 to change, 0 to destroy.

The generated plan output is included as terraform/plan-output.txt. The binary tfplan file was treated as a local generated artifact and was not committed.

Why:
This keeps the submitted Terraform code clean for real AWS usage while still providing a reproducible plan artifact for the assignment review.

