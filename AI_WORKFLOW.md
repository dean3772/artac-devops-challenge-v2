# AI Workflow

This document explains how AI assistance was used during the assignment, which prompts were useful, what AI suggestions were wrong or suboptimal, and how the final work was validated.

---

## AI tools used

I used AI assistance as a pair-programming, debugging, and review tool. I did not treat AI output as automatically correct. Suggestions were checked against repository files, local commands, runtime behavior, CI results, or Terraform output before being accepted.

The main AI tools used were:

* **ChatGPT** was used for repository analysis, debugging support, Docker and CI/CD design review, Terraform reasoning, and early drafting of the assessment and README.
* **Claude** was used to review and refine the deliverables before finalizing them: drafting and tightening the README and `ASSESSMENT.md`, checking the deploy dry-run template and user-data script for correctness, catching consistency issues between the documents and the actual repository state, and pushing the wording toward honesty rather than overstatement. Several of the final document versions were shaped in this review pass.

AI assistance was used for these tasks:

* Understanding the inherited repository and assignment requirements.
* Reviewing `README.md`, `DECISIONS.md`, `Dockerfile`, `requirements.txt`, GitHub Actions, Terraform, user-data, application files, tests, and the model artifact.
* Debugging why the container started successfully but `/predict` failed at runtime.
* Improving the Docker image while respecting the black-box constraint on `app/` and `models/`.
* Restructuring the CI/CD pipeline so tests, smoke tests, vulnerability scanning, and image publishing happen in the correct order.
* Reasoning about Trivy scan results and dependency updates.
* Improving the EC2 bootstrap script and the Terraform validation flow.
* Drafting and refining `ASSESSMENT.md`, `README.md`, and `AI_WORKFLOW.md`.

---

## Prompts that worked well

The examples below are shortened for readability, but they reflect the prompts and tasks that were most useful during the work.

### Prompt 1: Understanding the inherited repository before changing files

```text
I am working on a DevOps home assignment repository.
Repository: dean3772/artac-devops-challenge-v2

This is an inherited codebase assignment. A previous engineer built a small ML prediction API and initial DevOps infrastructure. Before changing files, help me understand the repository deeply.

Please review the README, DECISIONS.md, Dockerfile, requirements, GitHub Actions workflow, Terraform files, user-data script, app files, tests, and model artifact.

Do not suggest modifying app/ or models/, because the assignment treats them as black box constraints. Do not write the final ASSESSMENT.md or AI_WORKFLOW.md yet.

Explain how the application, Docker image, CI/CD pipeline, Terraform, and deployment pieces connect, and help me identify where the previous decisions may be correct, incomplete, or wrong.
```

This prompt worked well because it separated understanding from implementation. It helped create a mental model of the repository before making changes, and it kept the focus on engineering judgment instead of jumping directly to a final solution.

### Prompt 2: Debugging the prediction failure

```text
The Docker image builds and the FastAPI app starts. /health and /ready return 200, but /predict returns 500.

The logs show scikit-learn InconsistentVersionWarning. The model was serialized with scikit-learn 1.8.0, but requirements.txt installs scikit-learn 1.6.1. The runtime error is:

AttributeError: 'LogisticRegression' object has no attribute 'multi_class'

Given that app/ and models/ are black box constraints, what is the safest fix and how should I validate it?
```

This prompt was useful because it included concrete runtime evidence instead of asking for a generic Docker review. The result was to fix the dependency version to match the serialized model instead of modifying application code or regenerating the model.

### Prompt 3: Designing the CI/CD deploy stage honestly

```text
The assignment allows actual AWS deployment to be optional. I have a CI pipeline that tests, builds, smoke-tests, scans, and pushes the image.

I need the deploy job to be honest: it should not pretend to deploy to EC2 because no real host or secrets are configured. But it should still document the intended deployment flow clearly.

Help me design a dry-run deploy step that shows the real flow for deploying the validated image to an existing EC2 instance, including SSH key handling, container replacement, and a /ready check, without actually executing deployment.
```

This prompt helped turn a bare deploy placeholder into a documented dry-run template. It made the pipeline clearer while staying honest about what was actually executed in the assignment environment.

---

## AI mistakes or suboptimal suggestions that I caught

### 1. Terraform offline planning was initially oversimplified

One early AI suggestion treated `terraform plan -refresh=false` as if it would be enough to generate the required plan artifact without real AWS credentials. When I actually ran Terraform, the AWS provider still attempted credential and account validation, and the naive approach failed with a 403 from STS.

I caught this by testing the command locally instead of relying on the explanation. The working approach was to temporarily use dummy provider credentials together with provider skip-validation settings only for generating `terraform/plan-output.txt`. After the plan artifact was generated, I reverted the provider block back to the normal AWS provider configuration before committing.

This kept the submitted Terraform code clean while still producing the required plan output for review.

### 2. The deploy dry-run was initially too noisy

One AI draft for the deploy job included too many deployment alternatives directly inside the GitHub Actions workflow, including manual SSH, SSH helper actions, SSM, ECS, CodeDeploy, and other options. Those are valid production topics, but putting all of them inside the workflow made the deploy job harder to read and less focused.

I caught this during review and reduced the workflow to one clear dry-run path: SSH to an existing EC2 instance, pull the validated image, replace the running container idempotently, and check `/ready`. I kept only one short note that a ready-made GitHub Actions SSH helper action could replace the manual SSH setup, and moved the broader production alternatives to `ASSESSMENT.md` and the README.

This made the workflow more useful for the assignment: simple enough to review, honest that it does not execute, and still clear about the intended deployment flow.

---

## How I validated AI output

AI helped with reasoning and drafting, but I validated the important suggestions through actual commands and repository checks.

Validation included:

* Building and running the Docker image locally.
* Checking `/health`, `/ready`, `/docs`, and `/predict`.
* Reading container logs when `/predict` failed.
* Running the test suite with `pytest`.
* Pushing changes and checking the GitHub Actions pipeline.
* Confirming that the image was smoke-tested and scanned before publishing.
* Pulling the published image from the registry and exercising the endpoints again.
* Generating and reviewing `terraform/plan-output.txt`.
* Reviewing documentation against the final repository state before committing.

---

## Time saved

AI significantly reduced the time spent on repository orientation, debugging direction, CI/CD design review, and documentation drafting.

Estimated time saved compared with working fully manually: **approximately 8-10 hours**.

The largest time savings came from:

* Quickly mapping the inherited repository structure.
* Narrowing the `/predict` failure to a dependency and model serialization mismatch.
* Iterating on the Docker and CI/CD design.
* Drafting `ASSESSMENT.md` and the README in a consistent format.
* Catching documentation consistency issues before the final commit.

---

## Total time spent

Total time spent on the assignment, including repository review, local testing, Docker fixes, CI/CD fixes, Terraform validation, documentation, AI review, commits, and CI verification: **approximately 9-11 hours**.