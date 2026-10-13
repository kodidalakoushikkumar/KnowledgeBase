
## 1. Architectural Concept & Data Flow

This project implements a **Monolithic Containerised Architecture** with an automated **CI/CD (Continuous Integration & Continuous Deployment)** pipeline.

### Why this architecture?

- **Monolith**: All application logic (static HTML/CSS) and the web server (Nginx) live inside a single container unit.
- **Containerisation**: Packaging the site into a Docker container ensures that the website runs identically on your local machine, inside GitHub Actions, or on AWS EC2.
- **Automated Deployment**: Pushing code to GitHub automatically triggers a build, packages the image, transfers it to AWS EC2 over SSH, and restarts the live website.

```
┌───────────────────────────┐         ┌───────────────────────────────┐
│     Developer Machine     │  Git    │      GitHub Repository        │
│ (Writes HTML + Dockerfile)├────────►│        (main branch)          │
└───────────────────────────┘  Push   └──────────────┬────────────────┘
                                                     │
                                                     │ GitHub Actions Workflow
                                                     ▼
┌───────────────────────────┐         ┌───────────────────────────────┐
│     AWS EC2 Instance      │         │     GitHub Actions Runner     │
│  (Docker Engine on Port 80)◄────────┤ 1. Check out code             │
└───────────────────────────┘ SSH/SCP │ 2. Build Docker image         │
                                      │ 3. Transfer & run on EC2      │
                                      └───────────────────────────────┘
```

---

## 2. Project Folder Structure

Organising files cleanly ensures GitHub Actions and Docker can locate application assets:

```
monolithic-devops-project/
├── website/
│   └── index.html         # Static web page content
├── Dockerfile             # Blueprint for packaging Nginx and website
└── .github/
    └── workflows/
        └── deploy.yml     # CI/CD automation script
```

---

## 3. Detailed Component Explanations

### Component 1: The Web Page (`website/index.html`)

This file contains the raw semantic HTML served to web visitors.

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My DevOps Project</title>
</head>
<body>
    <h1>Hello, DevOps!</h1>
    <p>This website is served by Nginx in Docker on AWS EC2.</p>
</body>
</html>
```

- **`<!DOCTYPE html>`**: Declares the document type as HTML5.
- **`<meta name="viewport" ...>`**: Ensures proper scaling on mobile and desktop browsers.

---

### Component 2: The Container Recipe (`Dockerfile`)

The `Dockerfile` instructs Docker how to build the application container.

```
FROM nginx:alpine

COPY website/ /usr/share/nginx/html/

EXPOSE 80
```

#### Line-by-Line Explanation:

- **`FROM nginx:alpine`**: Downloads the official Nginx web server image built on **Alpine Linux**. Alpine is chosen because it is extremely lightweight (~5 MB base size), fast to download, and secure.
- **`COPY website/ /usr/share/nginx/html/`**: Copies your local `website/` directory contents into Nginx's default root directory inside the container. Nginx automatically looks in `/usr/share/nginx/html/` to serve `index.html`.
- **`EXPOSE 80`**: Informs Docker that Nginx listens on network port 80 inside the container.

---

### Component 3: Provisioning AWS EC2 & Security Setup

An **Amazon EC2 (Elastic Compute Cloud)** virtual machine hosts the running container in the cloud.

#### Security Group Configuration

Security groups act as a virtual firewall controlling inbound network traffic to EC2:

|Rule Type|Port|Allowed Source|Explanation|
|---|---|---|---|
|**SSH**|`22`|`My IP`|Restricts remote command-line access strictly to your administrative machine for security.|
|**HTTP**|`80`|`0.0.0.0/0`|Opens port 80 to the entire internet so public visitors can load your website in a browser.|

#### Docker Setup Commands on EC2

Once connected to your EC2 instance via SSH, you prepare the environment:

```
# 1. Install Docker engine on the virtual machine
sudo dnf install -y docker

# 2. Start the Docker daemon process immediately
sudo systemctl start docker

# 3. Enable Docker to start automatically whenever the EC2 instance reboots
sudo systemctl enable docker

# 4. Add the default user ('ec2-user') to the 'docker' security group
sudo usermod -aG docker ec2-user
```

- **Why run `usermod -aG docker`?** By default, Docker requires `sudo` privileges. Adding `ec2-user` to the `docker` group allows GitHub Actions to execute `docker` commands remotely without password prompts.

---

### Component 4: GitHub Secrets Setup

Storing credentials directly in code files exposes private keys to public repositories. **GitHub Repository Secrets** encrypt sensitive values:

1. Navigate to **Settings > Secrets and variables > Actions** in GitHub.
2. Add the following secrets:
    - **`EC2_HOST`**: The public IP address of your EC2 instance (e.g., `[suspicious link removed]`).
    - **`EC2_USERNAME`**: The default SSH username (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu).
    - **`EC2_SSH_KEY`**: The complete text of your private SSH key (`.pem` file), including the opening `-----BEGIN RSA PRIVATE KEY-----` and closing header lines.

---

### Component 5: The CI/CD Automation Pipeline (`.github/workflows/deploy.yml`)

The GitHub Actions workflow automates the entire process whenever new code is pushed.

```
name: Build and Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Validate website assets
        run: test -s website/index.html

      - name: Build Docker image
        run: docker build -t devops-website:${{ github.sha }} .

      - name: Export Docker image to tar archive
        run: docker save devops-website:${{ github.sha }} -o devops-website.tar

      - name: Transfer Docker archive to EC2 via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "devops-website.tar"
          target: "~/"

      - name: Deploy and Run Container on EC2 via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker stop devops-website || true
            docker rm devops-website || true
            docker load -i ~/devops-website.tar
            docker run -d --name devops-website -p 80:80 devops-website:${{ github.sha }}
            rm ~/devops-website.tar
```

#### Detailed Step-by-Step Breakdown:

1. **`on: push: branches: [main]`**: Triggers the workflow automatically whenever changes are merged or pushed to `main`.
2. **`runs-on: ubuntu-latest`**: Requests an isolated, temporary Linux virtual machine hosted by GitHub to run the pipeline.
3. **`actions/checkout@v4`**: Clones your repository source code onto the GitHub runner.
4. **`test -s website/index.html`**: Checks that `index.html` exists and is non-empty before attempting to package it.
5. **`docker build -t devops-website:${{ github.sha }} .`**: Compiles the Docker image, assigning a unique tag equal to the Git commit ID (`github.sha`) to track version history.
6. **`docker save -o devops-website.tar`**: Packages the compiled Docker image into a portable `.tar` file archive.
7. **`appleboy/scp-action`**: Uses Secure Copy Protocol (SCP) to upload `devops-website.tar` over SSH directly to your EC2 instance home directory.
8. **`appleboy/ssh-action`**: Establishes an SSH connection to EC2 and executes a shell script remotely:
    - **`docker stop devops-website || true`**: Stops any previously running version of the container (`|| true` prevents pipeline failure if no container is running yet).
    - **`docker rm devops-website || true`**: Deletes the old container instance to free up the container name.
    - **`docker load -i ~/devops-website.tar`**: Unpacks the transferred `.tar` file into the EC2 Docker daemon.
    - **`docker run -d --name devops-website -p 80:80 devops-website:${{ github.sha }}`**: Launches the container in detached mode (`-d`), binding EC2 host port 80 to container port 80.
    - **`rm ~/devops-website.tar`**: Deletes the temporary tar file from EC2 to prevent disk space accumulation.

---

### Component 6: Version Control Commands (`Git Workflow`)

Execute these commands locally to send your code to GitHub and trigger the pipeline:

```
# 1. Initialize a new Git repository locally
git init

# 2. Stage all project files for tracking
git add .

# 3. Create a snapshot of the code with a descriptive commit message
git commit -m "feat: initial website setup with automated EC2 deployment"

# 4. Rename the default branch to 'main'
git branch -M main

# 5. Link local repository to your remote GitHub repository
git remote add origin https://github.com/YOUR_USERNAME/monolithic-devops-project.git

# 6. Push local code to GitHub and set main as the upstream branch
git push -u origin main
```

---

## 4. Verification & Next Steps

1. **Verification**: Open `http://<YOUR_EC2_PUBLIC_IP>` in your web browser. You will see your HTML page served directly from the Nginx container on AWS EC2.
2. **Key Concepts Mastered**:
    - **Monolithic Architecture**: Bundling web markup with Nginx.
    - **Containerisation**: Building minimal Docker containers.
    - **CI/CD Pipelines**: Automated linting, building, and SSH deployment.
    - **Cloud Infrastructure**: Cloud security groups and Docker container orchestration on EC2.



# Adding IaC

Here is the complete **Infrastructure as Code (IaC)** expansion using **Terraform**. This automates the provisioning of your AWS EC2 instance, Security Groups, and Docker installation so you never have to manually click around the AWS Console or manually install software on the server.

---

## 1. Updated Architecture & Folder Structure

With Terraform added, your repository now manages both **Infrastructure (IaC)** and **Application Code (CI/CD)**:

```
monolithic-devops-project/
├── website/
│   └── index.html             # Static website content
├── Dockerfile                 # Container image configuration
├── .github/
│   └── workflows/
│       └── deploy.yml         # CI/CD pipeline
└── terraform/                 # Infrastructure as Code
    ├── [suspicious link removed]            # AWS provider setup
    ├── [suspicious link removed]           # Input variables
    ├── [suspicious link removed]                # Security Group & EC2 Instance
    └── [suspicious link removed]             # Outputs (Public IP)
```

---

## 2. Terraform Code & Detailed Breakdown

Create a `terraform/` folder in your project root with the following four files:

### File 1: `terraform/[suspicious link removed]`

Defines the cloud provider plugin (AWS) and minimum required versions.

```
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = [suspicious link removed]_region
}
```

- **`required_providers`**: Downloads the official HashiCorp AWS provider plugin.
- **`provider "aws"`**: Configures AWS as the target cloud platform using the region defined in `[suspicious link removed]`.

---

### File 2: `terraform/[suspicious link removed]`

Parameterises your infrastructure configuration so values can be reused or overridden easily.

```
variable "aws_region" {
  type        = string
  default     = "us-east-1"
  description = "AWS Region to deploy resources into"
}

variable "instance_type" {
  type        = string
  default     = "t2.micro"
  description = "EC2 instance machine type"
}

variable "key_name" {
  type        = string
  description = "Name of your existing AWS SSH Key Pair"
}
```

---

### File 3: `terraform/[suspicious link removed]`

The core blueprint containing the **AMI Data Source**, **Security Group**, and **EC2 Instance with automated Docker installation**.

```
# 1. Dynamically locate the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

# 2. Security Group Firewall Rules
resource "aws_security_group" "web_sg" {
  name        = "monolithic-devops-sg"
  description = "Allow inbound HTTP and SSH access"

  # Allow public HTTP traffic on port 80
  ingress {
    description = "HTTP Public Access"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow SSH administration on port 22
  ingress {
    description = "SSH Admin Access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound traffic (so EC2 can update packages)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "monolithic-devops-sg"
  }
}

# 3. Compute Resource (EC2 Instance)
resource "aws_instance" "web_server" {
  ami                    = [suspicious link removed]
  instance_type          = var.instance_type
  key_name               = var.key_name
  vpc_security_group_ids = [[suspicious link removed]]

  # User Data script: runs automatically on initial server startup
  user_data = <<-EOF
              #!/bin/bash
              dnf update -y
              dnf install -y docker
              systemctl start docker
              systemctl enable docker
              usermod -aG docker ec2-user
              EOF

  tags = {
    Name = "monolithic-devops-server"
  }
}
```

#### Line-by-Line Highlights:

- **`data "aws_ami" "amazon_linux"`**: Automatically fetches the latest official Amazon Linux 2023 image ID without hardcoding region-specific AMI IDs.
- **`aws_security_group`**: Replaces manual firewall setup by declaring ingress rules for port `80` (HTTP web access) and port `22` (SSH deployment).
- **`user_data`**: A startup shell script passed to EC2. It automatically updates packages, installs Docker, enables the Docker system daemon, and grants `ec2-user` permission to manage containers upon boot.

---

### File 4: `terraform/[suspicious link removed]`

Outputs crucial values after deployment, such as the dynamically assigned public IP address.

```
output "ec2_public_ip" {
  value       = aws_instance.web_server.public_ip
  description = "Public IP address of the provisioned EC2 instance"
}
```

---

## 3. Provisioning Infrastructure with Terraform Commands

Run these commands inside the `terraform/` directory from your local terminal:

```
# 1. Initialize Terraform (downloads AWS provider plugins)
terraform init

# 2. Preview the resources Terraform will create
terraform plan -var="key_name=YOUR_AWS_KEY_PAIR_NAME"

# 3. Provision the infrastructure on AWS
terraform apply -var="key_name=YOUR_AWS_KEY_PAIR_NAME" -auto-approve
```

Once execution completes, Terraform displays the output:

```
Outputs:

ec2_public_ip = "[suspicious link removed]"
```

---

## 4. Connecting Terraform to GitHub Actions CI/CD

1. Copy the output IP (`[suspicious link removed]`) into your GitHub Repository Secret **`EC2_HOST`**.
2. Run `git push origin main`.
3. GitHub Actions connects to the EC2 instance provisioned by Terraform, loads your container, and deploys the live static site automatically!

---

🛠️ Would you like me to add a **Terraform destroy & cleanup command section** to avoid unnecessary AWS charges when you're done testing?


# Destroying IaC

Here is the **Terraform Destroy & Resource Cleanup Guide** to help you tear down your AWS infrastructure and avoid unexpected charges when you finish testing.

---

## 🧹 1. Tearing Down Infrastructure with Terraform

When you no longer need the EC2 instance and Security Groups, Terraform allows you to destroy all provisioned cloud resources in a single command.

### Cleanup Commands

Navigate to your `terraform/` directory in your local terminal and execute:

```
# 1. Preview which resources will be destroyed
terraform plan -destroy -var="key_name=YOUR_AWS_KEY_PAIR_NAME"

# 2. Destroy all resources provisioned by Terraform
terraform destroy -var="key_name=YOUR_AWS_KEY_PAIR_NAME" -auto-approve
```

---

## 🔍 2. What Happens During `terraform destroy`?

Terraform reads your local state file (`terraform.tfstate`) and systematically removes resources in reverse dependency order:

1. **Terminates the EC2 Instance:** Stops and permanently deletes the `monolithic-devops-server` virtual machine and its attached root EBS storage volume.
2. **Deletes the Security Group:** Removes `monolithic-devops-sg` firewall rules after the EC2 instance using it has been fully terminated.
3. **Updates State File:** Clears `terraform.tfstate` to reflect that zero resources remain active in your AWS account.

---

## 🛡️ 3. Verification & Best Practices

To ensure no orphan resources are left running in AWS:

1. **Check AWS Console:**
    - Go to **EC2 > Instances** and verify the instance status shows `Terminated`.
    - Go to **EC2 > Security Groups** and confirm `monolithic-devops-sg` is removed.
2. **Do Not Delete `terraform.tfstate` Manually Before Destroying:**
    - Terraform relies on `terraform.tfstate` to know what resources it created. If you delete this file manually before running `terraform destroy`, Terraform loses track of your AWS assets, and you will have to terminate the EC2 instance manually via the AWS Console.
3. **AWS Free Tier Awareness:**
    - **EC2 `t2.micro`:** Covered under 750 free hours/month for the first 12 months.
    - **Best Practice:** Running `terraform destroy` when you are done practicing guarantees you won't accidentally exceed monthly thresholds.

---

🎯 Would you like me to help summarize this complete workflow into a final project README or architecture document for your repository?