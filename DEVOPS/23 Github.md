## Section 2

Section 2 (**Overview of GitHub Actions**) provides the structural foundation of the platform, detailing its event-driven architecture, core building blocks, key operational terminology, and a side-by-side comparative analysis against other industry CI/CD platforms.

### **Overview & Execution Paradigm**

GitHub Actions is an **event-driven automation platform** embedded directly inside GitHub repositories. It enables teams to construct automated pipelines that execute on either GitHub-managed or self-hosted virtual infrastructure.

Workflows are triggered by repository events—such as code pushes, pull requests, scheduled crons, or external webhooks—and execute a sequence of jobs, where each job runs inside an isolated, fresh virtual machine or container environment.

### **2.1 Core Concepts & Architecture**

#### **1. The Five Core Primitives**

The platform relies on five fundamental building blocks:

- **Workflow**: A YAML configuration file located in the repository at `.github/workflows/` that defines the automation logic.
- **Event**: A specific trigger (e.g., `push`, `pull_request`, `schedule`, `workflow_dispatch`) that initiates a workflow run.
- **Job**: A collection of steps that execute sequentially on the same runner machine. By default, multiple jobs within a single workflow execute in parallel.
- **Step**: An individual operational task within a job—either executing a raw shell command (`run`) or invoking a prebuilt reusable component (`uses`).
- **Action**: A standalone, reusable unit of code published to the GitHub Marketplace or defined locally inside the repository.

#### **2. Architectural Workflow Execution Flow**

When an event occurs in the repository, the execution lifecycle follows six discrete stages:

1. **Event Triggering**: A developer action (e.g., git push) or API call fires an event on GitHub.
2. **Workflow Evaluation**: The GitHub Actions engine scans `.github/workflows/` and evaluates conditional rules to determine eligible workflows.
3. **Queueing**: Triggered workflows are placed into the GitHub Actions service queue.
4. **Runner Allocation**: The service allocates an isolated virtual machine (hosted or self-hosted runner) for each job.
5. **Sequential Step Execution**: The runner clones the repository code, downloads required Actions, and executes job steps sequentially.
6. **Reporting**: Output logs, generated build artefacts, and final exit status codes are sent back to the GitHub UI.

### **2.2 Key Terminology**

Understanding these ten fundamental terms is necessary for navigating the platform's advanced features:

1. **Runner**: The underlying virtual or physical host (GitHub-hosted or self-hosted) executing the workflow jobs.
2. **Artefact**: Persistent files generated during workflow execution (e.g., compiled binaries, test execution reports, code coverage metrics).
3. **Secret**: Encrypted environment variables stored securely at the repository, organization, or environment level.
4. **Environment**: A named deployment target (e.g., `staging`, `production`) configured with optional protection rules, required human reviewers, and wait timers.
5. **Expression**: Dynamically evaluated strings written in `${{ <expression> }}` syntax used to access context data or evaluate runtime conditions.
6. **Context**: Structured JSON data objects (such as `github`, `env`, `secrets`, `needs`, `steps`) accessible during pipeline execution.
7. **Composite Action**: A custom Action that bundles multiple shell commands and other Actions into a single reusable unit.
8. **Reusable Workflow**: A complete workflow YAML file that can be called and executed by other parent workflows.
9. **Matrix Strategy**: An automated mechanism that executes a single job across multiple parameter combinations simultaneously (e.g., testing across OS versions or runtime versions).
10. **OIDC Token**: A short-lived OpenID Connect token used for keyless, secure authentication with external cloud providers.

### **2.3 GitHub Actions vs. Alternatives**

GitHub Actions differs from traditional CI/CD engines like Jenkins, GitLab CI, and CircleCI across several technical dimensions:

| Feature              | **GitHub Actions**                    | **Jenkins**                       | **GitLab CI**              | **CircleCI**       |
| :------------------- | :------------------------------------ | :-------------------------------- | :------------------------- | :----------------- |
| **Setup Effort**     | **Zero** (built natively into GitHub) | High (requires server management) | Low (built-in)             | Low (cloud hosted) |
| **Configuration**    | **YAML-based**                        | Groovy Pipeline / YAML            | YAML-based                 | YAML-based         |
| **Free Tier Quotas** | **2,000 min/month** (private repos)   | Self-hosted infrastructure costs  | 400 min/month              | 6,000 min/month    |
| **Ecosystem**        | **16,000+ Marketplace Actions**       | 2,000+ plugins                    | Moderate built-in features | Orbs library       |
| **OIDC Support**     | **Native keyless integration**        | Requires external plugin          | Native                     | Native             |
| **Matrix Builds**    | **Native**                            | Parameterized builds              | Native                     | Native             |
| **Self-Hosted**      | **Supported**                         | Supported                         | Supported                  | Supported          |

#### **Plan Quotas & Pricing Nuances**

- **Public Repositories**: Completely **free with unlimited build minutes**.
- **Private Repositories**:
    - **Free Tier**: 2,000 minutes/month.
    - **Pro Tier**: 3,000 minutes/month.
    - **Enterprise Tier**: 50,000 minutes/month.

## Section 3

Here is a complete, step-by-step breakdown of **Section 3: Getting Started** from the documentation.

### **3.1 Prerequisites**

Before authoring your first GitHub Actions workflow, four foundational prerequisites must be met:

1. **GitHub Repository**: You need access to either a public or private GitHub repository, as GitHub Actions is available across all account plans.
2. **Repository Permissions**: You must possess **write access** to the repository to create or modify files inside the `.github/workflows/` directory.
3. **Actions Enabled**: Ensure GitHub Actions is turned on for your repository by navigating to **Settings > Actions > General** and confirming that Actions execution is enabled.
4. **Basic YAML Knowledge**: Workflows are authored in **YAML 1.2**. Indentation **must strictly use spaces**, as tabs are invalid syntax in YAML parsing.

---

### **3.2 Creating Your First Workflow**

Workflows are defined as individual YAML files located in the `.github/workflows/` directory at the root of your repository. While the filename serves as the default identifier, you can override the displayed workflow title using the top-level `name:` key.

#### **Step 1 — Create the Directory Structure**

Run the following shell commands in your repository root to create the workflows directory and an initial pipeline file:

```
mkdir -p .github/workflows
touch .github/workflows/ci.yml
```

#### **Step 2 — Write the Workflow Definition**

Populate `.github/workflows/ci.yml` with the following baseline Continuous Integration pipeline:

```
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build application
        run: npm run build
```

##### **Line-by-Line Code Breakdown**:

- **`name: CI Pipeline`**: Sets the human-readable display name in the GitHub UI.
- **`on:`**: Defines event triggers. This workflow runs when code is pushed to `main` or `develop`, or when a pull request targets `main`.
- **`jobs:`**: Contains job declarations.
- **`build-and-test:`**: Defines a job running on a fresh `ubuntu-latest` virtual machine.
- **`actions/checkout@v4`**: Fetches and clones the repository code into the runner workspace.
- **`actions/setup-node@v4`**: Installs Node.js version 20 onto the runner host.
- **`npm ci` / `npm test` / `npm run build`**: Executes shell commands sequentially to install exact dependencies, run test suites, and compile the application.

#### **Step 3 — Commit, Push, and Verify**

Stage, commit, and push the workflow configuration to your remote GitHub repository:

```
git add .github/workflows/ci.yml
git commit -m 'ci: add initial GitHub Actions workflow'
git push origin main
```

Once pushed, navigate to your repository on GitHub and click the **Actions** tab. Your workflow run will appear and start executing within seconds.

### **3.3 Simple CI/CD Pipeline Example**

The following production-grade pipeline demonstrates a full Continuous Integration and Continuous Deployment (CI/CD) workflow. It automates dependency installation, unit testing, code coverage artifact archiving, Docker image compilation, and keyless authentication push to **Amazon ECR**.

```
# .github/workflows/cicd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-app
  IMAGE_TAG: ${{ github.sha }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm test -- --coverage

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::585593375370:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push Docker image
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
```

#### **Key Pipeline Mechanics**:

1. **Environment Block (`env`)**: Standardizes values for the AWS region (`ap-south-1`), ECR repository name (`my-app`), and dynamic image tag using the unique commit SHA (`${{ github.sha }}`).
2. **`test` Job**:
    - Uses `actions/setup-node@v4` with built-in dependency caching (`cache: 'npm'`) to speed up workflow executions.
    - Generates code coverage metrics via `npm test -- --coverage`.
    - Archives the resulting `coverage/` directory as a downloadable workflow artifact using `actions/upload-artifact@v4`.
3. **`build-and-push` Job**:
    - **`permissions:` Block**: Grants `id-token: write` permission required for keyless **OpenID Connect (OIDC)** authentication with AWS.
    - **AWS Credential Assumption**: Exchanges a short-lived OIDC token for temporary AWS IAM role credentials (`GitHubActionsRole`) using `aws-actions/configure-aws-credentials@v4`.
    - **Amazon ECR Login & Push**: Logins to ECR via `aws-actions/amazon-ecr-login@v2`, captures the registry endpoint from step outputs (`${{ steps.login-ecr.outputs.registry }}`), and executes native `docker build` and `docker push` commands.

#### **Key Architectural Rule**

The **`needs: test`** directive forms a strict dependency gate between jobs. It enforces that the `build-and-push` job **only executes if the `test` job passes successfully**. This prevents broken or unverified code from ever being packaged and pushed to your production container registry.

---

🚀 **Next Step**: Would you like to proceed to **Section 4: Building Workflows — Step-by-Step** to analyze workflow syntax, event triggers, and inter-job data passing in detail?

## Section 4

Here is the complete, exhaustive technical breakdown of **Section 4: Building Workflows — Step-by-Step** from the documentation.

---

### **4. Building Workflows — Step-by-Step**

Section 4 transitions from fundamental concepts to hands-on workflow engineering. It breaks down the YAML syntax schema, event triggers, job/step structures, virtual machine runner specifications, Marketplace integration, and data-passing mechanisms across jobs.

---

### **4.1 Workflow Syntax Deep Dive**

A GitHub Actions workflow is defined in YAML and relies on seven top-level keys:

|Key|Required?|Technical Description|
|:--|:-:|:--|
|**`name`**|No|Human-readable name displayed in the GitHub Actions UI.|
|**`on`**|**Yes**|Defines the event or set of events that trigger workflow execution.|
|**`env`**|No|Workflow-level environment variables inherited by all jobs and steps.|
|**`defaults`**|No|Default execution settings for all `run:` steps (such as default shell or working directory).|
|**`concurrency`**|No|Prevents duplicate workflow runs by cancelling or queueing concurrent executions.|
|**`permissions`**|No|Overrides default `GITHUB_TOKEN` permissions across the workflow.|
|**`jobs`**|**Yes**|Map of job IDs and their corresponding execution definitions.|

---

### **4.2 Triggers (Events)**

The `on:` key configures the events that initiate execution. It accepts a single event string, an array of events, or an event map with fine-grained filters.

#### **1. Push & Pull Request Triggers**

Supports glob patterns for branch matching, tag filtering, and file path exclusions:

```
on:
  push:
    branches:
      - main
      - 'release/**'   # Glob pattern matching release/v1, release/v2, etc.
    paths:
      - 'src/**'       # Triggers only when files under src/ change
      - '!src/**/*.md' # Excludes changes to markdown files
    tags:
      - 'v+.+.+' # Semantic version tag filter
  pull_request:
    branches: [ main ]
    types: [ opened, synchronize, reopened ] # Executes on PR open, new commit push, or reopen
```

#### **2. Schedule Trigger (Cron)**

Automates recurring workflows using standard UTC cron syntax:

```
on:
  schedule:
    - cron: '0 2 * * 1-5'   # Runs every weekday at 02:00 AM UTC
    - cron: '0 8 * * 0'     # Runs every Sunday at 08:00 AM UTC
```

#### **3. Manual Trigger with Inputs (`workflow_dispatch`)**

Allows users to trigger workflows manually via the GitHub UI or API with typed parameters:

```
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target deployment environment'
        required: true
        type: choice
        options: [ dev, staging, production ]
      debug_enabled:
        description: 'Enable debug logging'
        required: false
        type: boolean
        default: false
      version:
        description: 'Version tag to deploy'
        required: true
        type: string
```

#### **4. Repository Dispatch (External Trigger)**

Enables external systems (such as external CI, webhooks, or microservices) to trigger workflows via the GitHub REST API:

```
# Workflow YAML definition
on:
  repository_dispatch:
    types: [ deploy-triggered ]
```

_Triggering via API Call_:

```
curl -X POST \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/OWNER/REPO/dispatches \
  -d '{"event_type":"deploy-triggered","client_payload":{"ref":"v1.2.3"}}'
```

#### **5. Complete Event Reference**

Below is the full catalog of workflow trigger events:

| Event                     | Description                                                                                                                   |
| :------------------------ | :---------------------------------------------------------------------------------------------------------------------------- |
| **`push`**                | Fired when commits or tags are pushed to a branch.                                                                            |
| **`pull_request`**        | Fired when a pull request is opened, updated, or closed.                                                                      |
| **`pull_request_target`** | Similar to `pull_request`, but executes in the context of the base branch and has write access to secrets (use with caution). |
| **`workflow_dispatch`**   | Manual trigger accepting interactive user inputs.                                                                             |
| **`schedule`**            | Cron-based time trigger.                                                                                                      |
| **`release`**             | Fired when a GitHub Release is created, published, or edited.                                                                 |
| **`workflow_call`**       | Enables a workflow to function as a reusable template called by other workflows.                                              |
| **`repository_dispatch`** | Webhook-driven API trigger from external systems.                                                                             |
| **`workflow_run`**        | Triggered upon the completion of another specified workflow.                                                                  |
| **`create` / `delete`**   | Fired when a git branch or tag is created or deleted.                                                                         |
| **`deployment`**          | Fired when a GitHub deployment object is created.                                                                             |
| **`issue_comment`**       | Fired when a comment is added to an issue or pull request.                                                                    |

---

### **4.3 Jobs & Steps**

A **job** is a set of steps executed sequentially on a assigned runner host. By default, multiple jobs run concurrently. Dependency chains are established using the `needs:` directive.

```
jobs:
  # Job 1 – executes immediately
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  # Job 2 – executes concurrently with lint
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  # Job 3 – waits for both lint AND unit-tests to succeed
  build:
    needs: [ lint, unit-tests ]
    runs-on: ubuntu-latest
    if: ${{ always() && !contains(needs.*.result, 'failure') }}
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

#### **Step Types**

Steps fall into two categories:

1. **`uses`**: Executes a pre-built Action (from the Marketplace, a local directory `./.github/actions/`, or a Docker container `docker://`).
2. **`run`**: Executes shell commands (`bash` by default on Linux/macOS, `pwsh` on Windows).

```
steps:
  # 1. Marketplace Action invocation
  - name: Checkout code
    uses: actions/checkout@v4

  # 2. Single-line shell command
  - name: Print git log
    run: git log --oneline -10

  # 3. Multi-line shell script block
  - name: Multi-line script
    run: |
      echo 'Starting build...'
      npm ci
      npm run build
      echo 'Build complete!'

  # 4. Step output assignment
  - name: Set output variable
    id: version
    run: echo "tag=$(git describe --tags)" >> $GITHUB_OUTPUT

  # 5. Accessing step output variable
  - name: Use the output
    run: echo 'Tag is ${{ steps.version.outputs.tag }}'
```

---

### **4.4 Runners**

GitHub provides hosted virtual machine runners across Linux, macOS, and Windows operating systems. Self-hosted infrastructure can also be attached.

#### **GitHub-Hosted Runner Hardware Specs**

|Runner Label|Operating System|vCPUs|RAM|Storage|
|:--|:--|:-:|:-:|:-:|
|**`ubuntu-latest` / `ubuntu-24.04`**|Ubuntu 24.04 LTS|4|16 GB|14 GB SSD|
|**`ubuntu-22.04`**|Ubuntu 22.04 LTS|4|16 GB|14 GB SSD|
|**`windows-latest` / `windows-2022`**|Windows Server 2022|4|16 GB|14 GB SSD|
|**`macos-latest` / `macos-14`**|macOS 14 (Sonoma)|3|7 GB|14 GB SSD|
|**`macos-13`**|macOS 13 (Ventura)|4|14 GB|14 GB SSD|

> 💡 **Cost Optimization Tip**: macOS runners bill at **10x the rate of Linux runners**. Use `ubuntu-latest` for all platform-agnostic build tasks.

---

### **4.5 Using Actions from the Marketplace**

Actions are reusable workflow building blocks referenced via `owner/repo@ref` syntax.

#### **Action Pinning Strategies**

```
steps:
  # 1. Pinned to major version (Recommended for trusted actions; updates automatically within major release)
  - uses: actions/checkout@v4

  # 2. Pinned to exact version tag (Most stable against minor release changes)
  - uses: actions/setup-node@v4.0.2

  # 3. Pinned to full 40-character commit SHA (Most secure — immune to tag mutation attacks)
  - uses: actions/cache@0c45773b623bea8c8e75f6c82b208c3cf94ea4f

  # 4. Docker container action
  - uses: docker://alpine:3.19

  # 5. Local composite action in the same repo
  - uses: ./.github/actions/my-custom-action
```

#### **Essential Marketplace Actions Catalog**

|Action Reference|Core Purpose|
|:--|:--|
|**`actions/checkout@v4`**|Clones the repository code into the runner workspace.|
|**`actions/setup-node@v4`**|Installs Node.js with built-in dependency caching support.|
|**`actions/setup-python@v5`**|Installs Python environments.|
|**`actions/setup-java@v4`**|Installs Java JDK distributions (Temurin, Zulu, etc.).|
|**`actions/cache@v4`**|Caches directory paths between workflow executions.|
|**`actions/upload-artifact@v4`**|Saves workflow files as downloadable build artifacts.|
|**`actions/download-artifact@v4`**|Downloads saved artifacts within downstream jobs.|
|**`aws-actions/configure-aws-credentials@v4`**|Configures AWS credentials using OIDC or IAM keys.|
|**`docker/build-push-action@v5`**|Builds and pushes Docker images using BuildKit.|
|**`hashicorp/setup-terraform@v3`**|Installs and configures HashiCorp Terraform CLI.|
|**`github/codeql-action/analyze@v3`**|Runs CodeQL static security analysis.|

---

### **4.6 Passing Data Between Jobs**

Because every job runs in an isolated runner environment, sharing data between jobs requires structured mechanisms.

#### **Method 1 — Job Outputs (Lightweight Strings / Variables)**

Step outputs are exposed as job-level outputs, which downstream jobs access via the `needs` context:

```
jobs:
  generate-version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.set-version.outputs.version }}
    steps:
      - id: set-version
        run: echo "version=1.$(date +%Y%m%d).${GITHUB_RUN_NUMBER}" >> $GITHUB_OUTPUT

  deploy:
    needs: generate-version
    runs-on: ubuntu-latest
    steps:
      - run: echo 'Deploying version ${{ needs.generate-version.outputs.version }}'
```

#### **Method 2 — Artifacts (Files, Build Directories, Binaries)**

For binary outputs, coverage reports, or compiled code, use artifact archiving:

```
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build output
        uses: actions/download-artifact@v4
        with:
          name: dist-files
          path: dist/
      - run: ls -la dist/
```

---

🎯 **Next Step**: Would you like to proceed directly to **Section 5: Advanced Features** (covering Matrix Builds, Dependency Caching, Reusable Workflows, Composite Actions, Environments, OIDC Auth, ARC Self-Hosted Runners, and Concurrency Controls)?

## Section 5

Section 5 (**Advanced Features**) covers strategies and tools for optimizing, scaling, securing, and maintaining production-grade GitHub Actions pipelines.

### **5.1 Matrix Builds**

**Matrix builds** allow a single job definition to run concurrently across multiple variable combinations. This eliminates the need to duplicate workflow code when testing across multiple operating systems, language runtime versions, or database engines.

#### **1. Basic Matrix Strategy**

A matrix grid is declared under `strategy.matrix`. GitHub Actions automatically generates a job for every unique combination:

```
jobs:
  test-matrix:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ ubuntu-latest, windows-latest, macos-latest ]
        node: [ 18, 20, 22 ]
      # Spawns 3 × 3 = 9 parallel jobs
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

#### **2. Matrix Controls, Exclusions, and Inclusions**

- **`fail-fast: false`**: Prevents GitHub from cancelling all remaining matrix jobs if a single job fails.
- **`max-parallel`**: Limits the number of matrix jobs running simultaneously.
- **`exclude`**: Skips specific unwanted combinations.
- **`include`**: Adds custom parameters to specific combinations or injects entirely new jobs into the matrix.

```
strategy:
  fail-fast: false      # Continue other matrix jobs even if one fails
  max-parallel: 4       # Run at most 4 jobs at once
  matrix:
    os: [ ubuntu-latest, windows-latest ]
    python: [ '3.10', '3.11', '3.12' ]
    exclude:
      - os: windows-latest
        python: '3.10'   # Skip this specific combination
    include:
      - os: ubuntu-latest
        python: '3.12'
        experimental: true  # Inject extra variable into this combination
```

#### **3. Dynamic Matrix Generation**

Matrix values can be computed dynamically at runtime using scripts and passed to downstream jobs via `fromJson()`:

```
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          SERVICES=$(ls services/ | jq -R -s -c 'split(" ")[:-1]')
          echo "matrix={\"service\":${SERVICES}}" >> $GITHUB_OUTPUT

  build-service:
    needs: generate-matrix
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
    steps:
      - run: echo 'Building ${{ matrix.service }}'
```

---

### **5.2 Caching Dependencies**

Dependency caching is one of the most effective ways to accelerate workflow execution. The `actions/cache@v4` action persists specified directories across workflow runs based on a unique cache key.

#### **1. Package Manager Caching Implementations**

- **npm Cache**:
    
    ```
    - name: Cache npm dependencies
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-npm-
    - run: npm ci
    ```
    
- **Python pip Cache**:
    
    ```
    # Built-in shorthand in setup-python
    - uses: actions/setup-python@v5
      with:
        python-version: '3.12'
        cache: 'pip'
    
    # Or manually with actions/cache:
    - uses: actions/cache@v4
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
        restore-keys: ${{ runner.os }}-pip-
    ```
    
- **Docker Layer Cache (Buildx)**:
    
    ```
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Build with cache
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: my-app:latest
        cache-from: type=gha          # Restore from GitHub Actions cache
        cache-to: type=gha,mode=max   # Store all layers in cache
    ```
    
- **Gradle and Maven Cache**:
    
    ```
    # Gradle
    - uses: actions/cache@v4
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    
    # Maven
    - uses: actions/cache@v4
      with:
        path: ~/.m2
        key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    ```
    

> 💡 **Cache Invalidation Principle**: Using a lockfile hash (e.g., `package-lock.json`, `pom.xml`, `requirements.txt`) ensures the cache automatically invalidates when dependencies change. Providing `restore-keys` serves as a fallback mechanism to fetch partial cache hits on unmatched keys.

---

### **5.3 Reusable Workflows**

**Reusable workflows** allow organizations to define centralized pipeline templates that can be called by multiple repositories or pipelines.

#### **1. Defining a Reusable Workflow (`workflow_call`)**

A workflow is made reusable by adding the `workflow_call:` event trigger along with explicit input, secret, and output schemas:

```
# .github/workflows/deploy-template.yml
name: Deploy Template

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
    secrets:
      KUBE_CONFIG:
        required: true
    outputs:
      deployment_url:
        description: 'URL of the deployed application'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.get-url.outputs.url }}
    steps:
      - name: Deploy to Kubernetes
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBE_CONFIG }}
        run: |
          echo $KUBECONFIG_DATA | base64 -d > kubeconfig.yml
          kubectl --kubeconfig kubeconfig.yml set image \
            deployment/app app=myrepo/app:${{ inputs.image_tag }}
```

#### **2. Calling a Reusable Workflow**

Parent workflows invoke reusable templates using the `uses:` key and pass parameters via `with:` and `secrets:`:

```
# .github/workflows/production-deploy.yml
name: Production Deploy

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.tag.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: tag
        run: echo "tag=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT
      - run: docker build -t myrepo/app:${{ steps.tag.outputs.tag }} .

  deploy-production:
    needs: build
    uses: myorg/shared-workflows/.github/workflows/deploy-template.yml@main
    with:
      environment: production
      image_tag: ${{ needs.build.outputs.image_tag }}
    secrets:
      KUBE_CONFIG: ${{ secrets.PROD_KUBE_CONFIG }}
```

---

### **5.4 Composite Actions**

**Composite Actions** allow you to package multiple steps into a custom reusable action stored inside your repository. Unlike reusable workflows, composite actions run directly inside the caller's existing job without provisioning a new runner host.

#### **1. Action Definition (`.github/actions/setup-and-test/action.yml`)**

Composite actions require `runs.using: 'composite'` and explicit `shell:` settings on every `run:` step:

```
# .github/actions/setup-and-test/action.yml
name: Setup and Test
description: Install dependencies and run tests

inputs:
  node-version:
    description: 'Node.js version to use'
    default: '20'
  test-command:
    description: 'Custom test command'
    default: 'npm test'

outputs:
  test-exit-code:
    description: 'Exit code from test run'
    value: ${{ steps.test.outputs.exit_code }}

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Run tests
      id: test
      shell: bash
      run: |
        ${{ inputs.test-command }} || true
        echo "exit_code=$?" >> $GITHUB_OUTPUT
```

#### **2. Invoking a Local Composite Action**

```
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-and-test
    with:
      node-version: '22'
      test-command: 'npm run test:ci'
```

---

### **5.5 Environments & Deployment Protection Rules**

Environments represent target infrastructure stages (e.g., `dev`, `staging`, `production`) and enforce security controls before job execution.

#### **1. Configurable Environment Controls**

- **Required Reviewers**: Specific users or teams must manually approve job execution.
- **Wait Timers**: Delays job execution for a specified duration (0–30 days).
- **Deployment Branches**: Restricts deployments to specific branches or tag patterns.
- **Environment Secrets**: Credentials scoped exclusively to a target deployment stage.

#### **2. Environment Workflow Usage**

```
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: echo 'Deploying to staging...'

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com   # URL link displayed directly in GitHub UI
    steps:
      - run: echo 'Deploying to production...'
```

---

### **5.6 OIDC & Keyless Cloud Authentication**

**OpenID Connect (OIDC)** enables GitHub Actions to authenticate securely with cloud providers (AWS, GCP, Azure) without storing long-lived static API keys or credentials as secrets. The runner exchanges a short-lived JSON Web Token (JWT) for short-lived cloud credentials at runtime.

#### **1. AWS OIDC Configuration**

1. **AWS Identity Provider & IAM Role Setup (Terraform Example)**:
    
    ```
    resource "aws_iam_openid_connect_provider" "github" {
      url             = "https://token.actions.githubusercontent.com"
      client_id_list  = ["sts.amazonaws.com"]
      thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
    }
    
    resource "aws_iam_role" "github_actions" {
      name = "GitHubActionsRole"
      assume_role_policy = jsonencode({
        Version = "2012-10-17"
        Statement = [{
          Effect    = "Allow"
          Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
          Action    = "sts:AssumeRoleWithWebIdentity"
          Condition = {
            StringEquals = {
              "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
            }
            StringLike = {
              "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:*"
            }
          }
        }]
      })
    }
    ```
    
2. **Workflow Authentication**:
    
    ```
    jobs:
      deploy:
        runs-on: ubuntu-latest
        permissions:
          id-token: write   # Required to request the OIDC JWT token
          contents: read
        steps:
          - uses: aws-actions/configure-aws-credentials@v4
            with:
              role-to-assume: arn:aws:iam::585593375370:role/GitHubActionsRole
              aws-region: ap-south-1
          - run: aws s3 ls   # Authenticated via temporary IAM session keys
    ```
    

#### **2. Google Cloud Platform (Workload Identity Federation)**

```
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/my-pool/providers/github
          service_account: github-actions@my-project.iam.gserviceaccount.com
      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud run deploy my-app --image gcr.io/my-project/app:latest
```

---

### **5.7 Self-Hosted Runners**

Self-hosted runners allow jobs to execute on private servers, local VMs, cloud instances (EC2/GKE), or hardware platforms (like Raspberry Pis). They provide access to internal networks, custom hardware (GPUs), and help reduce runner minute costs.

#### **1. Manual Runner Setup**

```
# Download runner tarball
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.315.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.315.0/actions-runner-linux-x64-2.315.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.315.0.tar.gz

# Configure runner connection
./config.sh --url https://github.com/myorg/myrepo --token <REGISTRATION_TOKEN>

# Install and start background service
sudo ./svc.sh install
sudo ./svc.sh start
```

#### **2. Targeting Self-Hosted Runners in Workflows**

```
jobs:
  build-on-prem:
    runs-on: [ self-hosted, linux, gpu ] # Targets runners with matching labels
    steps:
      - uses: actions/checkout@v4
      - run: nvidia-smi                   # Utilizes host GPU resources
```

#### **3. Ephemeral Scaling via Actions Runner Controller (ARC)**

For Kubernetes clusters, **Actions Runner Controller (ARC)** deploys operator controllers that auto-scale ephemeral runner pods dynamically based on job queue volume:

```
# Install ARC Helm Controller
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --namespace arc-systems --create-namespace

# Deploy Runner Scale Set
helm install arc-runner-set \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --namespace arc-runners --create-namespace \
  --set githubConfigUrl=https://github.com/myorg/myrepo \
  --set githubConfigSecret.github_token=<PAT>
```

---

### **5.8 Concurrency Control**

The `concurrency` property controls parallel execution and prevents redundant or conflicting workflow runs.

#### **1. Cancel In-Progress Runs (Feature/Topic Branches)**

Automatically cancels older running builds when a new commit is pushed to the same branch:

```
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

#### **2. Sequential Queueing (Deployments)**

Enforces strict sequential execution for deployment jobs without cancelling active deployments:

```
concurrency:
  group: deploy-production
  cancel-in-progress: false   # Queues new deployments until current run finishes
```

---

🛠️ **Next Step**: Would you like to proceed to **Section 6: Best Practices** (covering security, performance, workflow organization, cost optimization, and DevSecOps scanning) or **Section 7: Troubleshooting**?


