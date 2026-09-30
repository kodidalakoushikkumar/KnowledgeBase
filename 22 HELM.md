### **Module 1: Introduction to Helm**

#### **1. What is Helm?**

**Helm** is the de facto package manager for Kubernetes, frequently described as the equivalent of `apt` or `yum` for container orchestration[1]. It dramatically simplifies deploying, upgrading, and managing complex Kubernetes applications by bundling all required resource definitions into a single, versioned, and reusable package called a **Chart**[1][2].

Before Helm was introduced, deploying a multi-component application to Kubernetes required writing and maintaining dozens of individual YAML manifests—such as Deployments, Services, ConfigMaps, Ingresses, and RBAC rules[3]. Every target environment (development, staging, production) required separate files with slightly different configurations[3]. Helm solves this friction using a **templating engine** combined with a **package management system**[2][3].

---

#### **2. Why Helm Matters in Production (Before vs. With Helm)**

In production environments, Helm addresses five core application lifecycle challenges[2]:

| Capability        | Without Helm                                         | With Helm                                                  |
| ----------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| **Packaging**     | Scattered, unorganized YAML files per application[2] | A single packaged `.tgz` chart containing all resources[2] |
| **Configuration** | Manual edits across dozens of separate files[2]      | A central `values.yaml` file with environment overrides[2] |
| **Upgrades**      | `kubectl apply` with manual tracking[2]              | `helm upgrade` with built-in atomic rollback protection[2] |
| **Rollback**      | Re-applying older YAML manifests manually[2]         | Instant execution via `helm rollback                       |

### **Module 2: Helm Architecture & Core Concepts**

#### **1. Helm v3 Architecture & Operating Model**

- **Client-Side Architecture:** Helm v3 operates entirely on the client side without any server-side dependencies or daemons[1]. It communicates directly with the target Kubernetes API server using the active context and authentication credentials configured in your local `kubeconfig` file[1][4].
- **Elimination of Tiller:** Helm v2 relied on a server-side daemon called **Tiller**, which required elevated, cluster-wide administrative privileges[2][5]. Helm v3 completely removed Tiller, shifting all rendering and execution logic to the client side to significantly improve cluster security and respect native Kubernetes RBAC[1].
- **Components Inside the Helm CLI:** All processing logic runs inside the `helm` executable[2]:
    - **Chart Engine:** Renders YAML templates by combining Go `text/template` logic with user-provided configuration values[2][6].
    - **Release Manager:** Tracks the installation, upgrade, and rollback history of releases as versioned revisions[2][7].
    - **Repository Client:** Pulls, caches, and manages chart packages from local directories, HTTP repositories, or OCI registries[2].
- **State Persistence via Secrets:** Release state and deployment history are persisted directly inside the target Kubernetes cluster using versioned Kubernetes Secrets (labeled under `helm.sh/release.v1` by default) within the release's target namespace[2][7].

```
  Developer / CI-CD Pipeline
              │
              ▼
         [ helm CLI ]
  ┌────────────────────────┐
  │  • Chart Engine        │
  │  • Release Manager     │
  │  • Repository Client   │
  └───────────┬────────────┘
              │ Direct API Calls (kubeconfig)
              ▼
   [ Kubernetes API Server ]
              │
      ┌───────┴───────────────────────┐
      ▼                               ▼
[ Kubernetes Resources ]     [ Release Secrets ]
(Deployments, Services...)   (helm.sh/release.v1)
```


---

#### 2. Deep Dive into the 7 Core Concepts

1. **Chart**
   A Chart is the primary packaging unit in Helm [7]. It is a structured directory tree (or compressed `.tgz` archive) containing Kubernetes resource templates, metadata (`Chart.yaml`), default variables (`values.yaml`), and optional sub-charts/dependencies [7, 9]. It serves as the Kubernetes equivalent of an `apt` or `yum` package [7].

2. **Release**
   A Release is an actual deployed instance of a Chart running inside a Kubernetes cluster [7]. A single Chart can be instantiated multiple times within the same cluster or across different namespaces, with each instance receiving a distinct Release name (e.g., `nginx-dev` vs. `nginx-prod`) [7].

3. **Revision**
   Every installation, upgrade, or configuration change executed on a Release creates an incremental, integer-based **Revision** (starting at Revision 1) [7, 10]. Helm maintains full revision history inside its release secrets, enabling instant inspection or rollbacks to any previous stable state [7, 11].

4. **Repository**
   A Repository is an HTTP server or OCI-compliant registry that hosts packaged chart archives (`.tgz`) [6, 8]. Examples include public directories like Artifact Hub, managed chart repositories like Bitnami, self-hosted servers like ChartMuseum, or cloud OCI registries (AWS ECR, Azure ACR, Google GCR) [6, 8, 12].

5. **Values**
   Values provide configuration parameters injected into chart templates at render time [6]. Default configurations are defined in the chart's `values.yaml` file, while environment-specific overrides are applied during deployment using external YAML value files (`-f` / `--values`) or inline command flags (`--set`) [6].

6. **Template**
   Templates are Go `text/template` files located in the chart's `templates/` directory [6, 9]. They generate Kubernetes manifests dynamically by evaluating static YAML syntax alongside built-in Helm objects, user values, named helper templates (`_helpers.tpl`), and Sprig utility functions [6, 9, 13, 14].

7. **Hook**
   Hooks are specialized Kubernetes manifests annotated to execute at specific lifecycle events during a deployment [6, 15]. Supported hook triggers include `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-rollback`, and `pre-delete`—commonly used for automated tasks like database schema migrations or health checks [6, 15].


### **Module 3: Installation & Setup**

#### **1. Overview & System Prerequisites**

Helm v3 is packaged as a single, lightweight binary with zero server-side dependencies[1]. Installation takes under two minutes on any operating system, and the client integrates directly with your existing `kubeconfig` context and cloud provider credentials[1]. Because there is no Tiller daemon, installing Helm simply involves placing the binary in your executable path and configuring repository pointers[1][2].

---

#### **2. Platform-Specific Installation Methods**

##### **Linux (Recommended for Automation & CI/CD)**

Linux environments offer two primary installation paths[1]:

- **Method 1: Official Installer Script** Automatically fetches, verifies, and installs the latest stable Helm v3 binary[1]:

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```

- **Method 2: Manual Binary Tarball (Specific Version Control)** Ideal for air-gapped environments or CI/CD pipelines requiring pinned versions[1]:

```
# Define desired version
HELM_VERSION=v3.14.0

# Download the release archive
curl -LO https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz

# Unpack archive
tar -zxvf helm-${HELM_VERSION}-linux-amd64.tar.gz

# Move binary to system path
sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version

```

##### **macOS**

Install via Homebrew package manager[3]:

```
# Install via Homebrew
brew install helm

# Verify version
helm version --short

```

##### **Windows**

Install using popular Windows package managers[3]:

```
# Via Chocolatey
choco install kubernetes-helm

# Via Scoop
scoop install helm

```

---

#### **3. Configuring Production Helm Repositories**

After installing the CLI binary, you must configure chart repositories to discover and download pre-packaged Kubernetes applications[3]. Repositories are indexed locally on your system and kept up to date using synchronization commands[3].

##### **Adding Core Production Repositories**

```
# Bitnami (Standard production-ready application charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Ingress-NGINX (Kubernetes ingress controllers)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Prometheus Community (Observability, metrics, and alerting stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Jetstack (Cert-Manager for automated TLS certificate management)
helm repo add jetstack https://charts.jetstack.io

```

##### **Repository Management Commands**

```
# Download and update local repository index caches
helm repo update

# Display all locally configured repositories
helm repo list

```

---

#### **4. Helm Environment Variables Reference**

Helm's execution behavior, file locations, and operational defaults can be customized using key environment variables[4][5]:

|Environment Variable|Description|Default Path / Value|
|---|---|---|
|**HELM_CACHE_HOME**|Local directory used to store cached chart packages and indexes[4].|`~/.cache/helm`[4]|
|**HELM_CONFIG_HOME**|Directory holding Helm configuration settings and repository definitions[4].|`~/.config/helm`[4]|
|**HELM_DATA_HOME**|Directory storing plugins and local chart data[4].|`~/.local/share/helm`[4]|
|**HELM_DEBUG**|Enables detailed verbose logging for all executed Helm commands when set[5].|`1` (or `true`)[5]|
|**HELM_NAMESPACE**|Sets default cluster namespace for all commands, overriding the `-n` flag[5].|Current `kubeconfig` namespace[5]|
|**HELM_NO_PLUGINS**|Disables all installed Helm plugins during execution when set[5].|`1`[5]|
|**KUBECONFIG**|Explicit file path to cluster credentials; respected identically to `kubectl`[5].|`~/.kube/config`[5]|

---

#### **5. Security & RBAC Considerations**

- **Context Inheritance:** Helm v3 relies strictly on your active `kubeconfig` context and inherits the exact Role-Based Access Control (RBAC) permissions of the executing user or account[5].
- **CI/CD Best Practices:** For automated deployment pipelines, **never** grant `cluster-admin` privileges to Helm[5]. Instead, create a dedicated Kubernetes `ServiceAccount` bound to Roles with minimal, namespace-scoped permissions required exclusively for the target application charts[5].

---

### **Module 4: Helm Charts Deep Dive**

A **Helm Chart** is a structured directory tree (or a compressed `.tgz` archive) containing all the definitions required to run an application, microservice, or piece of infrastructure on a Kubernetes cluster[1]. Understanding chart anatomy is fundamental both for using pre-built public charts and for authoring custom production charts[1].

---

#### **1. Complete Chart Directory Structure**

When a chart directory is initialized or expanded, it follows a standardized layout where each file and folder fulfills a distinct role in the application lifecycle[1]:

```
my-app/                    # Chart root directory
├── Chart.yaml             # Chart metadata (required)
├── values.yaml            # Default configuration values
├── values.schema.json     # Optional: JSON Schema for values validation
├── charts/                # Sub-chart dependencies (local or downloaded)
│   └── redis-18.0.0.tgz
├── crds/                  # Custom Resource Definitions (installed first)
├── templates/             # Kubernetes resource templates
│   ├── _helpers.tpl       # Named template definitions (not rendered directly)
│   ├── deployment.yaml    # Workload deployment manifest template
│   ├── service.yaml       # Networking service manifest template
│   ├── ingress.yaml       # Ingress rules manifest template
│   ├── configmap.yaml     # Environment configuration manifest template
│   ├── hpa.yaml           # Horizontal Pod Autoscaler manifest template
│   ├── serviceaccount.yaml# Pod identity &amp; RBAC manifest template
│   └── NOTES.txt          # Post-install help notes printed to stdout
└── .helmignore            # Files to exclude from packaging
```

##### **Directory &amp; File Components Breakdown**
* **`Chart.yaml`**: The required manifest holding high-level metadata, versioning information, and dependency declarations [1, 2].
* **`values.yaml`**: The default configuration parameters for the chart templates [1].
* **`values.schema.json`**: An optional JSON Schema used to validate input parameters during rendering or installation [1].
* **`charts/`**: Directory storing packaged sub-charts (`.tgz`) or unpacked sub-chart directories that act as application dependencies [1].
* **`crds/`**: Houses Custom Resource Definitions (CRDs); Helm installs CRDs in this folder prior to rendering and deploying template manifests [1].
* **`templates/`**: Folder containing Go `text/template` files that evaluate into Kubernetes YAML manifests [1, 3].
  * **`_helpers.tpl`**: Stores reusable named template partials and helpers; files prefixed with an underscore are **not** rendered directly into standalone Kubernetes objects [1].
  * **`NOTES.txt`**: A plain text template evaluated after installation/upgrade to output instructions, endpoint URLs, or credentials to the user's terminal [1].
* **`.helmignore`**: Specifies file patterns to ignore when packaging the chart (similar in syntax to `.gitignore`) [1].

---

#### 2. Deep Dive into `Chart.yaml`
The `Chart.yaml` file defines key metadata and operational directives for the package [1, 2].

##### **Sample `Chart.yaml` Configuration**
```yaml
apiVersion: v2                     # v2 for Helm 3 charts
name: my-app
description: A production-ready application chart
type: application                  # 'application' or 'library'
version: 1.3.0                     # SemVer: chart package version
appVersion: '2.1.0'                # Version of the underlying application
keywords:
  - nodejs
  - api
  - microservice
home: https://github.com/myorg/my-app
maintainers:
  - name: Aditya Jaiswal
    email: aditya@devopsshack.com
dependencies:
  - name: redis
    version: '&gt;=18.0.0 &lt;19.0.0'
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled          # Only install if redis.enabled=true
```

##### **Field Reference &amp; Specification**
* **`apiVersion`**: Set to `v2` for all Helm 3 charts (`v1` was used in Helm 2) [2].
* **`name`**: The official name of the chart package [2].
* **`type`**: Specifies whether the chart is an `application` (deployable workload) or a `library` (provides shared partials/helpers for other charts) [2].
* **`version`**: The **Semantic Version (SemVer)** of the Helm chart itself [2]. It must be bumped whenever the chart's templates or defaults change [2].
* **`appVersion`**: The version string of the actual application software or container image being deployed (e.g., `2.1.0`) [2].
* **`dependencies`**: Array listing external sub-charts [2]:
  * `name`: The name of the dependent chart [2].
  * `version`: SemVer constraint for the dependency [2].
  * `repository`: The URL or OCI repository hosting the sub-chart [2].
  * `condition`: A boolean expression path in `values.yaml` (e.g., `redis.enabled`) that controls whether the dependency is fetched and deployed [2].

---

#### 3. `values.yaml` Design &amp; Best Practices
The `values.yaml` file acts as the public API of the Helm chart [4]. Every configurable parameter should be exposed clearly with descriptive comments to facilitate multi-environment overrides [4].

##### **Production-Grade `values.yaml` Example**
```yaml
# -- Number of pod replicas
replicaCount: 2

image:
  # -- Container registry and repository
  repository: myapp
  # -- Image pull policy
  pullPolicy: IfNotPresent
  # -- Image tag (overrides chart appVersion)
  tag: ''

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: false
  className: nginx
  host: app.example.com
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

##### **Best Practices for Chart Values**
* **Document Every Key**: Use inline comments above keys to explain their function, type, and expected format [4].
* **Explicit Resource Allocations**: Always define baseline `requests` and `limits` for CPU and Memory within `values.yaml` [4, 5].
* **Feature Toggles**: Make optional components (like Ingress, Autoscaling, or Sub-charts) explicitly toggleable via boolean flags (e.g., `ingress.enabled: false`) [2, 4, 5].
* **Separation of Concerns**: Keep deployment-specific settings out of template files and drive all variability strictly through `values.yaml` [3, 4].

---

### Module 5: Helm Commands Reference

Module 5 provides a complete reference for the Helm commands used in day-to-day DevOps operations and automated production pipelines. These commands are categorized by lifecycle stage: **Installation & Upgrade**, **Inspection & Debugging**, **Chart Development**, and the **Production Deployment Pattern**.

---

#### 1. Installation & Upgrade Commands (Release Lifecycle)

These commands control the lifecycle of releases inside the target Kubernetes cluster:

- **`helm install <name> <chart>`**
    
    - **Purpose:** Installs a chart for the first time.
    - **Key Flags:**
        - `--namespace <ns>` (`-n`): Specifies the target namespace.
        - `--values <file>` (`-f`): Specifies external values override YAML files.
        - `--set key=value`: Overrides specific values inline from the CLI.
        - `--atomic`: Automatically purges/rolls back the release if the install fails within the timeout.
- **`helm upgrade <name> <chart>`**
    
    - **Purpose:** Updates an existing release to a new version or configuration.
    - **Key Flags:**
        - `--install`: Installs the chart if it doesn't already exist on the cluster.
        - `--atomic`: Rolls back to the previous stable revision if the upgrade fails.
        - `--cleanup-on-fail`: Deletes any newly created resources if the deployment fails.
- **`helm upgrade --install <name> <chart>`**
    
    - **Purpose:** Operates as an **idempotent deployment command**. If the release does not exist, it runs `install`; if it exists, it runs `upgrade`. This is the standard command used inside CI/CD automation.
- **`helm rollback <name> <revision>`**
    
    - **Purpose:** Instantaneously reverts a release back to a previous revision number.
    - **Key Flags:**
        - `--wait`: Blocks until all rolled-back pods pass readiness probes.
        - `--timeout <duration>`: Sets the maximum duration to wait for the rollback to succeed.
        - `--cleanup-on-fail`: Cleans up orphaned resources if the rollback fails.
- **`helm uninstall <name>`**
    
    - **Purpose:** Deletes a release and removes all associated Kubernetes manifests from the cluster.
    - **Key Flags:**
        - `--keep-history`: Retains historical release metadata secrets so the release name and history can still be inspected.
        - `--dry-run`: Simulates release removal without deleting actual resources.
        - `--wait`: Waits until all associated Kubernetes resources are fully terminated.

---

#### 2. Inspection & Debugging Commands

These commands allow operators to inspect deployed releases, view rendered manifests, and debug issues before or after deployment:

- **`helm list`**
    
    - `helm list -n <namespace>`: Lists all releases in the specified namespace.
    - `helm list -A`: Lists all active releases across **all namespaces** in the cluster.
- **`helm status <release-name> -n <namespace>`**
    
    - **Purpose:** Displays the overall release status, current revision, namespace, last deployed timestamp, and post-installation notes (`NOTES.txt`).
- **`helm get values <release-name> -n <namespace>`**
    
    - **Purpose:** Shows the user-supplied configuration values for a deployed release.
    - **Key Flag:** Add `--all` to view the complete computed values tree (merging default `values.yaml` with user overrides).
- **`helm get manifest <release-name> -n <namespace>`**
    
    - **Purpose:** Outputs the raw rendered Kubernetes YAML manifests currently running in the cluster for that release.
- **`helm history <release-name> -n <namespace>`**
    
    - **Purpose:** Displays the full revision history table for a release, showing revision numbers, update timestamps, release status (`deployed`, `superseded`, `failed`), chart version, and application version.
- **`helm upgrade --install myapp ./my-app -n prod --dry-run`**
    
    - **Purpose:** **Dry Run Mode.** Simulates an installation or upgrade by communicating with the Kubernetes API server to validate manifests without applying any real changes to the cluster.
- **`helm template myapp ./my-app --values prod.values.yaml`**
    
    - **Purpose:** **Local Template Rendering.** Renders chart templates locally on the client machine and prints the output YAML to standard output. **No connection to a live Kubernetes cluster is required**.

---

#### 3. Chart Development & Package Commands

Commands used when authoring, testing, and distributing custom Helm charts:

- **`helm create <chart-name>`**
    
    - **Purpose:** Scaffolds a new chart directory containing a standard file layout (`Chart.yaml`, `values.yaml`, `templates/`, `.helmignore`).
- **`helm lint <path-to-chart>`**
    
    - **Purpose:** Scans a chart directory for syntax errors, missing fields, structural issues, and best-practice violations.
- **`helm package <path-to-chart>`**
    
    - **Purpose:** Compresses a chart directory into a versioned `.tgz` archive file (e.g., `my-app-1.3.0.tgz`).
    - **Signing Flag:** `helm package --sign --key '<identity>' --keyring <path-to-keyring> <chart-dir>` cryptographically signs the archive using a GPG key.
- **`helm verify <chart-file.tgz>`**
    
    - **Purpose:** Verifies that a signed `.tgz` chart package matches its cryptographic provenance file and signature before deployment.
- **`helm dependency update <path-to-chart>`**
    
    - **Purpose:** Reads dependency declarations in `Chart.yaml` and downloads missing sub-chart packages into the `charts/` directory.
- **`helm dependency list <path-to-chart>`**
    
    - **Purpose:** Lists all declared sub-charts and checks their local availability and version status.

---

#### 4. The Recommended Production Install Pattern

In production pipelines, Helm deployments should always run with atomic safety and explicit timeout controls.

##### **Standard Production Command Example**

```
helm upgrade --install my-app ./charts/my-app \
  --namespace production \
  --create-namespace \
  --values values/base.yaml \
  --values values/production.yaml \
  --set image.tag=${IMAGE_TAG} \
  --atomic \
  --wait \
  --timeout 5m0s \
  --cleanup-on-fail
```

##### **Breakdown of Production Flags:**

1. **`--install`**: Ensures idempotency (creates the release if missing; upgrades if present).
2. **`--create-namespace`**: Automatically creates the destination namespace if it does not already exist.
3. **`--values` (`-f`)**: Stacks multiple configuration files. Later values files override parameters in preceding ones.
4. **`--set image.tag=${IMAGE_TAG}`**: Dynamically injects runtime variables (like Git commit SHAs or build numbers) from the CI runner.
5. **`--atomic`**: Guarantees atomic deployments. If pods fail readiness checks or hooks fail during deployment, Helm automatically rolls back the cluster to the previous working state.
6. **`--wait`**: Blocks the command execution until all Pods, Services, and PVCs are fully ready.
7. **`--timeout 5m0s`**: Sets an explicit ceiling on how long Helm should wait for readiness before triggering a rollback.
8. **`--cleanup-on-fail`**: Cleans up failed resources to prevent partial deployments from leaving orphaned objects in the cluster.



### **Module 6: Templating & Values**

Helm's templating engine is powered by Go's native `text/template` package, extended with the **Sprig** function library (over 100 utility functions) and Helm-specific built-in objects[1]. Mastering templates allows you to create flexible, dynamic, and production-safe Kubernetes manifests driven entirely by configuration values[1].

---

#### **1. Built-in Objects Reference**

During template evaluation, Helm provides a top-level scope containing several built-in objects[1]. These objects give templates access to release metadata, chart configuration, cluster capabilities, and user-supplied values[1][2].

##### **Release Object (** **.Release** **)**

- **.Release.Name**: The name assigned to the release (e.g., `"nginx-prod"`)[1].
- **.Release.Namespace**: The target Kubernetes namespace where the release is installed[1].
- **.Release.IsInstall**: A boolean set to `true` if the current operation is an initial installation (returns `false` during upgrades)[1].
- **.Release.Revision**: An integer representing the current revision number (starts at `1` on initial install and increments with each upgrade)[2].

##### **Chart Object (** **.Chart** **)**

- **.Chart.Name**: The chart name as declared in `Chart.yaml`[2].
- **.Chart.Version**: The chart package version specified in `Chart.yaml`[2].
- **.Chart.AppVersion**: The version string of the underlying application specified in `Chart.yaml`[2].

##### **Values & System Objects**

- **.Values.***: Contains all values defined in `values.yaml`, merged with any external override files (`-f`) or CLI flags (`--set`)[2].
- **.Files.***: Provides helper methods to access and read non-template files within the chart directory (such as configuration scripts or certificates)[2].
- **.Capabilities.***: Contains information about the target Kubernetes cluster, including supported API versions and the Kubernetes server version[2].

---

#### **2. Production-Grade Deployment Template Breakdown**

A production-ready Deployment template demonstrates how built-in objects, values, whitespace control, and helper functions combine to generate Kubernetes manifests[2]:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  annotations:
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
          {{- end }}
```

##### **Line-by-Line Templating Mechanics:**
1. **Named Template Invocations (`include`)**:
   `{{ include "my-app.fullname" . }}` evaluates a named template defined in `_helpers.tpl` using the current context `.` [2, 4].
2. **Whitespace Stripping (`{{-` and `-}}`)**:
   Adding a dash (`-`) strips all whitespace and newlines preceding or following the tag [2]. This prevents invalid YAML formatting when rendering blocks [2, 4].
3. **Pipelining and Indentation (`| nindent`)**:
   `{{- include "my-app.labels" . | nindent 4 }}` renders the labels block, adds a new line, and indents every line by exactly 4 spaces to align properly under `metadata.labels` [2].
4. **Fallback Handling (`| default`)**:
   `image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}` checks if `image.tag` is defined in values; if empty or unset, it automatically falls back to `.Chart.AppVersion` [3, 4].
5. **YAML Block Conversion (`toYaml`)**:
   `{{- toYaml .Values.resources | nindent 12 }}` converts the structured `.Values.resources` dictionary directly into formatted YAML and indents it by 12 spaces [3, 4].
6. **Conditional Blocks (`{{- if ... }}`)**:
   `{{- if .Values.livenessProbe.enabled }}` conditionally evaluates and renders the `livenessProbe` block only when `livenessProbe.enabled` evaluates to `true` [3, 4].

---

#### 3. Core Template Functions Reference

Helm includes several essential functions for data manipulation, formatting, and safety checks [4, 5]:

| Function | Description | Example Syntax | Output / Action |
| :--- | :--- | :--- | :--- |
| **`default`** | Provides a fallback value if the target parameter is empty or unset [4]. | `{{ .Values.tag \| default .Chart.AppVersion }}` [4] | Falls back to app version if `tag` is omitted [4]. |
| **`quote` / `squote`** | Wraps a string value in double or single quotes [4]. | `{{ .Values.name \| quote }}` [4] | `"my-app"` [4] |
| **`upper` / `lower`** | Converts string values to upper or lower case [4]. | `{{ .Values.env \| upper }}` [4] | `"PRODUCTION"` [4] |
| **`toYaml`** | Serializes a Go/Helm object into YAML format [4]. | `{{- toYaml .Values.resources \| nindent 12 }}` [4] | Formatted YAML block indented 12 spaces [4]. |
| **`required`** | Halts rendering with an error message if a required value is missing [4, 6]. | `{{ required "DB_HOST required" .Values.db.host }}` [4] | Fails build if `.Values.db.host` is undefined [4]. |
| **`include`** | Renders a named helper template and returns the resulting string [4]. | `{{ include "chart.labels" . }}` [4] | Evaluates named helper template [4]. |
| **`tpl`** | Renders a string containing template syntax dynamically [4]. | `{{ tpl .Values.config . }}` [4] | Evaluates nested template expressions in values [4]. |
| **`lookup`** | Queries a live Kubernetes cluster for existing resources [5]. | `{{ lookup "v1" "Secret" "ns" "name" }}` [5] | Returns target resource object from cluster [5]. |

### Module 7: Production Deployment Strategies

Module 7 details the operational strategies required to manage production-grade Helm deployments. While Helm handles applying resource manifests and checking basic readiness, production systems require structured multi-environment configuration, lifecycle hooks for database migrations, robust secret management, and deterministic rollback workflows.

---

#### 1. Multi-Environment Management (Layered Values Structure)

The standard production pattern uses a **layered values override structure**. Instead of maintaining separate charts for development, staging, and production, a single chart is shared across environments while environmental differences are managed by stacking value files.

##### **Directory Layout**

A clean directory layout separates the immutable chart template logic from environment configurations:

```
deployment/
├── charts/
│   └── my-app/           # Shared chart templates and defaults
├── values/
│   ├── base.yaml         # Baseline settings shared across all environments
│   ├── dev.yaml          # Development overrides (1 replica, low CPU/memory)
│   ├── staging.yaml      # Staging overrides (mirrors production structure)
│   └── production.yaml   # Production overrides (high replicas, strict resource limits)
└── Makefile              # Shortcut deployment targets
```

##### **Layering Mechanics**

1. **`base.yaml`**: Contains common settings applicable to every tier (e.g., container ports, application labels, logging levels).
2. **`production.yaml`**: Overrides only environment-specific parameters—such as scaling `replicaCount: 5`, enabling ingress TLS, assigning strict CPU/memory requests and limits, and enabling autoscaling.
3. **Command Execution**: Stacking multiple `-f` / `--values` flags processes files from left to right, where rightmost files override preceding settings.

```
# Production Deployment Command
helm upgrade --install my-app ./charts/my-app \
  -f values/base.yaml \
  -f values/production.yaml \
  --set image.tag=$(git rev-parse --short HEAD) \
  -n production \
  --atomic
```

---

#### 2. Helm Hooks for Production Safety

Helm **Hooks** allow operators to intervene at specific points in a release lifecycle to execute automated jobs—such as running database migrations, executing pre-upgrade smoke tests, or cleaning up temporary resources.

##### **Supported Lifecycle Hook Triggers**

- **`pre-install`**: Executes after templates are rendered, but before any resources are created in the cluster.
- **`post-install`**: Executes after all resources in a chart have been installed.
- **`pre-upgrade`**: Executes on an upgrade request before any existing resources are modified.
- **`post-upgrade`**: Executes after all resources have been upgraded.
- **`pre-rollback`**: Executes before a rollback operation begins.
- **`pre-delete`**: Executes before any resources are removed during uninstallation.

##### **Production Pre-Upgrade Database Migration Example**

To prevent new pods from starting against an outdated database schema, a pre-upgrade hook runs a migration `Job` before the application deployment updates:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ['npm', 'run', 'migrate']
```

##### **Hook Annotations Reference:**

- **`helm.sh/hook`**: Declares the target lifecycle event (e.g., `pre-upgrade`).
- **`helm.sh/hook-weight`**: Sets execution order when multiple hooks are defined. Lower integer weights (e.g., `-5`) execute before higher weights.
- **`helm.sh/hook-delete-policy`**: Controls when the hook resource is removed. Values include `hook-succeeded` (deletes the job on success), `hook-failed`, or `before-hook-creation`.

---

#### 3. Secret Management with Helm

Storing unencrypted credentials, tokens, or private keys directly in `values.yaml` or committing them to Git creates critical security vulnerabilities. Production setups enforce distinct secret management strategies depending on infrastructure requirements:

|Approach|Underlying Tool|Operational Mechanism|
|:--|:--|:--|
|**External Secrets**|External Secrets Operator (ESO)|The Helm chart deploys an `ExternalSecret` Custom Resource; the cluster operator fetches the actual credentials from AWS Secrets Manager, HashiCorp Vault, or GCP Secret Manager and generates a native Kubernetes `Secret`.|
|**Sealed Secrets**|Bitnami Sealed Secrets|Encrypts raw secrets offline using a cluster public key into a `SealedSecret` CRD; safe to store directly in Git because only the cluster controller holds the private decryption key.|
|**Helm Secrets Plugin**|`helm-secrets` + Mozilla SOPS|Encrypts whole values files (e.g., `secrets.production.yaml`) using Cloud KMS or PGP; automatically decrypts files in-memory during `helm upgrade` commands.|
|**CSI Secret Store**|Secrets Store CSI Driver|Mounts secrets directly from external providers (Vault, AWS Secrets Manager) into pod volume filesystems without writing persistent secrets to Kubernetes ETCD.|
|**CI/CD Injections**|Pipeline `--set` / ENV Flags|Injects secrets dynamically during deployment pipelines using `--set db.password=$DB_PASS`; prevents saving secret values inside chart directories.|

---

#### 4. Production Rollback Strategy & Recovery Procedure

When an upgrade fails—due to application panics, failed readiness probes, or database incompatibility—a structured recovery procedure restores availability.

##### **Step-by-Step Rollback Workflow**

1. **Identify the Release State & Revisions:** Inspect current and historic release states across the namespace:
    
    ```
    helm history my-app -n production
    ```
    
    _Sample Output:_
    
    ```
    REVISION  UPDATED                   STATUS     CHART         APP VERSION
    1         Mon Jun 10 08:00:00 2025  superseded my-app-1.2.0  2.0.0
    2         Mon Jun 10 10:00:00 2025  superseded my-app-1.3.0  2.1.0
    3         Mon Jun 10 14:00:00 2025  failed     my-app-1.4.0  2.2.0
    ```
    
2. **Inspect the Target Stable Revision:** Verify the parameters of the last known healthy revision (e.g., Revision 2) before initiating the rollback:
    
    ```
    helm get values my-app --revision 2 -n production
    ```
    
3. **Execute Atomic Rollback:** Trigger the rollback command targeting the healthy revision, enforcing `--wait` and `--timeout` parameters to ensure pod readiness:
    
    ```
    helm rollback my-app 2 -n production --wait --timeout 3m0s
    ```
    
4. **Verify Application Health:** Confirm release status and check pod status in the target namespace:
    
    ```
    helm status my-app -n production
    ```
    

---
### Module 8: Helm with CI/CD Pipelines

Module 8 explores how Helm integrates into continuous integration, continuous delivery, and GitOps workflows. Helm serves as the industry-standard deployment engine for Kubernetes automation because of its **idempotent upgrade pattern (`helm upgrade --install`)** and **atomic rollback capabilities (`--atomic`)**, which guarantee that failed builds or unready pods do not leave clusters in a broken state.

---

#### 1. Helm's Role across DevSecOps Workflows

In modern automated pipelines, Helm acts as the unifying contract between application development and platform deployment:

- **Development Phase:** Developers define chart templates alongside application source code in version control.
- **CI Phase (Linting & Testing):** Automated pipelines run `helm lint` and **Chart Testing (`ct`)** on every pull request to catch syntax errors, missing values, or deprecated API versions before merging. Manifests are also scanned using security tools like Checkov, Trivy, or Polaris.
- **CD Phase (Automated Deployment):** Deployment jobs execute `helm upgrade --install` with environment value overrides and dynamic image tag injection.
- **GitOps Phase (Continuous Reconciliation):** GitOps controllers monitor chart repositories or Git branches to continuously sync cluster state to match the desired repository state.

---

#### 2. GitHub Actions Deployment Pipeline

The guide presents a complete, production-ready GitHub Actions workflow for deploying to an Amazon EKS cluster using keyless **OIDC authentication**.

##### **Pipeline Workflow Definition**

```
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # Required for AWS OIDC authentication
      contents: read
    steps:
      - uses: actions/checkout@v4

      # 1. Authenticate to AWS via OIDC (No long-lived access keys)
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRoleQA
          aws-region: ap-south-1

      # 2. Update cluster credentials in local context
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name my-cluster --region ap-south-1

      # 3. Setup Helm CLI binary
      - name: Install Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.14.0

      # 4. Idempotent Atomic Deployment
      - name: Helm Deploy
        run: |
          helm upgrade --install my-app ./charts/my-app \
            -n production \
            -f values/base.yaml \
            -f values/production.yaml \
            --set image.tag=${{ github.sha }} \
            --atomic --wait --timeout 5m
```

##### **Key Pipeline Highlights:**

- **OIDC Authentication:** Uses short-lived IAM credentials via OpenID Connect instead of hardcoded long-lived AWS secret keys.
- **Dynamic Commit SHAs:** Replaces image tags at runtime with `${{ github.sha }}` so every commit generates a uniquely versioned immutable release.
- **Production Guardrails:** Employs `--atomic --wait --timeout 5m` to ensure the job blocks until all new pods pass readiness probes; if pods fail within 5 minutes, Helm automatically triggers an immediate rollback.

---

#### 3. GitLab CI Pipeline Integration

For teams using GitLab CI/CD, deployment jobs execute inside dedicated, lightweight Helm docker containers.

##### **GitLab CI Configuration (`.gitlab-ci.yml`)**

```
deploy-production:
  stage: deploy
  image: alpine/helm:3.14.0
  environment:
    name: production
  script:
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
    - helm upgrade --install $APP_NAME ./charts/$APP_NAME \
        --namespace production \
        --values values/production.yaml \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --atomic --wait --timeout 5m0s
  only:
    - main
```

##### **Key Pipeline Highlights:**

- **Pre-built Runner Image:** Uses official lightweight runner images (`alpine/helm:3.14.0`) to minimize job setup time.
- **Built-in CI Variables:** Injects `$APP_NAME` and short commit hashes (`$CI_COMMIT_SHORT_SHA`) directly into deployment parameters.
- **Scoped Environments:** Leverages GitLab's native `environment: production` tracking to link release revisions directly to deployment dashboards.

---

#### 4. GitOps Workflow with ArgoCD & Flux CD

In a **GitOps continuous delivery model**, cluster deployments shift from "pushing" manifests from a CI runner to "pulling" and synchronizing changes directly from Git using an in-cluster operator like **ArgoCD** or **Flux CD (Helm Operator)**.

ArgoCD natively understands Helm charts and eliminates the need to execute Helm commands manually inside CI pipelines.

##### **ArgoCD `Application` CRD Manifest**

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  destination:
    namespace: production
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: HEAD
    path: charts/my-app
    helm:
      valueFiles:
        - values/production.yaml
      parameters:
        - name: image.tag
          value: v2.1.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

##### **GitOps Engine Capabilities:**

- **Declarative Source Pointers:** Points directly to the chart path (`charts/my-app`), target branch (`HEAD`), and environment values file (`values/production.yaml`).
- **Automated Pruning (`prune: true`):** Automatically deletes Kubernetes resources from the cluster if their manifest templates are removed from the Git repository.
- **Self-Healing (`selfHeal: true`):** Continuously monitors cluster state against Git. If a cluster administrator manually edits or deletes a resource (causing cluster drift), ArgoCD immediately overwrites the change to match Git.

---

### Module 9: Helm Repositories & ChartMuseum

For organizations authoring custom applications and microservices, hosting a private **Helm Chart Repository** is an essential piece of internal delivery infrastructure. While public charts provide standard off-the-shelf software, internal repositories allow engineering teams to version, control, and securely distribute proprietary charts across development, staging, and production environments.

---

#### 1. Hosting ChartMuseum on Kubernetes

**ChartMuseum** is an open-source, lightweight Helm chart repository server that provides an API for uploading, managing, and serving charts stored directly in cloud object storage backends (such as AWS S3, Google Cloud Storage, or Azure Blob Storage).

##### **Deploying ChartMuseum via Helm**

You can deploy ChartMuseum directly into your Kubernetes cluster using its official Helm chart:

```
# Add and update the ChartMuseum repository
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm repo update

# Install ChartMuseum with an Amazon S3 storage backend
helm upgrade --install chartmuseum chartmuseum/chartmuseum \
  --namespace helm-system \
  --create-namespace \
  --set env.open.STORAGE=amazon \
  --set env.open.STORAGE_AMAZON_BUCKET=my-helm-charts \
  --set env.open.STORAGE_AMAZON_REGION=ap-south-1 \
  --set env.open.DISABLE_API=false \
  --set serviceAccount.annotations.'eks\.amazonaws\.com/role-arn'=arn:aws:iam::551322107788:role/ChartMuseumS3Role
```

##### **Configuration Flags Breakdown:**

- **`STORAGE=amazon`**: Configures ChartMuseum to persist `.tgz` chart archives directly to an Amazon S3 bucket (`my-helm-charts`) in the specified region (`ap-south-1`).
- **`DISABLE_API=false`**: Enables HTTP API endpoints so automated CI/CD pipelines can upload (`POST`) or delete charts programmatically.
- **`serviceAccount.annotations`**: Attaches an IAM Role for Service Accounts (**IRSA**) to grant ChartMuseum granular read/write permissions to the S3 bucket without managing static, long-lived AWS secret keys.

---

#### 2. Native OCI Registry Support (Helm v3.8+)

Starting with **Helm v3.8+**, Helm natively supports the **Open Container Initiative (OCI)** specification. This capability allows you to store, version, package, and distribute Helm charts alongside application container images inside standard enterprise container registries—such as **Amazon ECR**, **Azure Container Registry (ACR)**, or **Google Artifact Registry (GAR)**—without operating a dedicated chart server like ChartMuseum.

##### **OCI Workflow Commands**

1. **Authenticate to the OCI Registry:** Log in to your registry using the `helm registry login` command:
    
    ```
    aws ecr get-login-password --region ap-south-1 | \
      helm registry login --username AWS --password-stdin \
      551322107788.dkr.ecr.ap-south-1.amazonaws.com
    ```
    
2. **Package & Push Chart Archive:** Package the chart directory into a `.tgz` file and push it using the `oci://` protocol scheme:
    
    ```
    # Package chart directory into tarball
    helm package ./my-app
    
    # Push to ECR repository
    helm push my-app-1.3.0.tgz \
      oci://551322107788.dkr.ecr.ap-south-1.amazonaws.com/helm-charts
    ```
    
3. **Install Directly from OCI Endpoint:** Deploy the release straight from the OCI registry path using `--version`:
    
    ```
    helm upgrade --install my-app \
      oci://551322107788.dkr.ecr.ap-south-1.amazonaws.com/helm-charts/my-app \
      --version 1.3.0 \
      -n production --atomic
    ```
    

---

#### 3. Essential Public Chart Repositories Directory

When consuming third-party applications or cluster add-ons, always rely on official, community-verified chart repositories:

|Repository Name|Repository URL|Key Notable Charts Provided|
|:--|:--|:--|
|**Artifact Hub**|`artifacthub.io`|Central public indexing hub for over 10,000+ community charts.|
|**Bitnami**|`charts.bitnami.com/bitnami`|Production-ready stateful apps & middleware (PostgreSQL, Redis, Kafka, NGINX).|
|**Prometheus Community**|`prometheus-community.github.io/helm-charts`|Complete observability stack (`kube-prometheus-stack`, Prometheus, Alertmanager).|
|**Ingress-NGINX**|`kubernetes.github.io/ingress-nginx`|Official NGINX Ingress Controller for Kubernetes traffic routing.|
|**Jetstack**|`charts.jetstack.io`|`cert-manager` for automated TLS/SSL certificate issuance and renewal.|
|**Grafana**|`grafana.github.io/helm-charts`|Visualization & telemetry logging tools (Grafana, Loki, Tempo, Alloy).|
|**AWS**|`aws.github.io/eks-charts`|EKS cluster infrastructure drivers (`aws-load-balancer-controller`, EBS/EFS CSI drivers).|

---

