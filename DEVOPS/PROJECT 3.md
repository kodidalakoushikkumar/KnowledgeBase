# Monolithic DevOps Project: GitHub Actions + Terraform + AWS EC2

For your basic DevOps portfolio project, you can build a pipeline where GitHub Actions triggers Terraform, Terraform creates an EC2 instance, and the instance downloads and launches your static website hosted on GitHub. You can trigger the workflow automatically or manually.

## 1. Architecture

```
Developer
Updates website code and pushes to GitHub
		↓
GitHub Repository
Website code + Terraform files + workflow
		↓
GitHub Actions
Automatic push trigger or manual Run workflow
Configure AWS credentials and execute Terraform
		↓
Terraform
Creates AWS infrastructure
		↓
AWS EC2
Installs Nginx → downloads website → serves it over HTTP
```

## 2. How the process works

1. You create a static website using HTML, CSS and JavaScript.
2. You push the code to GitHub.
3. GitHub Actions starts when you push to your selected branch, or you manually start the workflow.
4. The workflow configures AWS access and runs `terraform init`, `terraform plan` and `terraform apply`.
5. Terraform provisions an EC2 instance and configures its security group.
6. EC2 user data installs Nginx, downloads your website and starts the web server.
7. You access the website using the EC2 public IP address.

This is a monolithic deployment because the website is deployed as one application on a single EC2 instance. You do not need Docker, Kubernetes or a separate backend for this initial version.

## 3. GitHub repository structure

Create a repository with the following structure:

```
monolithic-devops/

	site/
		Website files
		index.html
		style.css
	
	terraform/AWS infrastructure
		main.tf
		variables.tf
		outputs.tf
	
	.github/workflows/
		deploy.yml
```

## 4. Basic website code

Create `site/index.html`:

```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My DevOps Portfolio</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <main>
        <h1>My DevOps Portfolio</h1>
        <p>Deployed on AWS EC2 using Terraform and GitHub Actions.</p>
        <p>Infrastructure as Code | CI/CD | AWS</p>
    </main>
</body>
</html>
```

Create `site/style.css`:

```

body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: #f1f5f9;
    color: #1e293b;
}

main {
    max-width: 700px;
    margin: 100px auto;
    padding: 30px;
    text-align: center;
    background: white;
    border-radius: 12px;
}
```

## 5. Terraform code

This example uses Ubuntu on EC2, installs Nginx through EC2 user data, and downloads the website from your public GitHub repository.

### `terraform/main.tf`

```

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Find the default VPC
data "aws_vpc" "default" {
  default = true
}

# Find a subnet in the default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Find the latest official Ubuntu 24.04 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Allow public HTTP traffic
resource "aws_security_group" "website" {
  name        = "portfolio-website-sg"
  description = "Allow HTTP traffic"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "website" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = var.instance_type
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.website.id]
  associate_public_ip_address = true

  # Replace the instance when a new deployment version is supplied.
  user_data_replace_on_change = true

  user_data = templatefile("${path.module}/user_data.sh", {
    repo_url        = var.repo_url
    deploy_version  = var.deploy_version
  })

  tags = {
    Name        = "Monolithic-Portfolio"
    Environment = "Dev"
  }
}
```

### `terraform/variables.tf`

```

variable "aws_region" {
  description = "AWS region for deployment"
  type        = string
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "repo_url" {
  description = "Public GitHub repository URL"
  type        = string
}

variable "deploy_version" {
  description = "Git commit SHA used to trigger a fresh deployment"
  type        = string
  default     = "initial"
}
```

### `terraform/outputs.tf`

```

output "website_url" {
  description = "Public URL of the website"
  value       = "http://${aws_instance.website.public_ip}"
}

output "instance_id" {
  value = aws_instance.website.id
}

output "public_ip" {
  value = aws_instance.website.public_ip
}
```

### `terraform/user_data.sh`

```

#!/bin/bash
set -euxo pipefail

# Deployment marker supplied by Terraform
DEPLOY_VERSION="${deploy_version}"

export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get install -y nginx git

# Download the website repository
rm -rf /tmp/monolithic-devops
git clone --depth 1 "${repo_url}" /tmp/monolithic-devops

# Publish the website files
mkdir -p /var/www/html
rm -rf /var/www/html/*
cp -r /tmp/monolithic-devops/site/. /var/www/html/

# Ensure Nginx starts automatically
systemctl enable nginx
systemctl restart nginx
```

Important: Replace the repository URL with your own public GitHub repository URL, such as `https://github.com/YOUR_USERNAME/monolithic-devops.git`. The repository must contain the `site/` directory.

This example deliberately replaces the EC2 instance whenever the deployment version changes. That makes a basic push-triggered deployment easy to demonstrate, but the public IP can change and each replacement may incur instance charges. Later, you can improve this by updating the website in place using AWS Systems Manager instead.

## 6. GitHub Actions workflow

Create `.github/workflows/deploy.yml`.

This workflow supports both automatic deployments on pushes to `main` and manual execution through the GitHub Actions interface.

```

name: Deploy Website to AWS EC2

on:
  push:
    branches:
      - main
    paths:
      - "site/**"
      - "terraform/**"
      - ".github/workflows/deploy.yml"

  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: terraform

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-south-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Initialize
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: |
          terraform plan \
            -var="repo_url=https://github.com/${{ github.repository }}.git" \
            -var="deploy_version=${{ github.sha }}" \
            -out=tfplan

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan

      - name: Show Website URL
        run: terraform output website_url
```

The workflow uses GitHub's OIDC authentication to assume an AWS IAM role, rather than storing long-lived AWS access keys in GitHub secrets.

## 7. One-time AWS and GitHub configuration

Before running the workflow, complete these prerequisites.

1. Create an S3 bucket for Terraform state. GitHub-hosted runners are temporary, so local Terraform state disappears between runs. Store state in S3 and configure an S3 backend in `main.tf` before running the workflow. Use a unique bucket name and enable versioning.
2. Create an AWS IAM OIDC identity provider for GitHub Actions. Configure an IAM role that trusts your GitHub repository and the `main` branch, with permissions to manage the EC2, security group and related resources used by this project.
3. Configure the GitHub repository variable or secret. Add `AWS_ROLE_ARN` under your repository's `Settings → Secrets and variables → Actions → Secrets`. Set its value to the IAM role ARN. The role also needs appropriate access to the Terraform state bucket.
4. Check your AWS network setup. The code assumes a default VPC with a subnet and internet access. Ensure your AWS account has the required resources and quotas.

For the S3 backend, add the following inside the top-level `terraform` block in `main.tf`, replacing the bucket name with your own:

```

backend "s3" {
  bucket = "YOUR-UNIQUE-TERRAFORM-STATE-BUCKET"
  key    = "monolithic-devops/terraform.tfstate"
  region = "ap-south-1"
}
```

Create and configure that bucket before running `terraform init`. For production-quality state management, configure state locking and tightly restrict bucket access. The AWS role's permissions should follow least privilege.

## 8. How to run the project

**Automatic deployment**
Edit the website → commit changes → push to `main` → GitHub Actions runs Terraform → EC2 is deployed → the workflow prints the website URL.

**Manual deployment**
Open GitHub → Actions → Deploy Website to AWS EC2 → Run workflow → select `main` → run it.

**Manual Terraform execution**
From the `terraform/` directory on your computer, configure AWS credentials and run Terraform commands directly.

```
terraform init

terraform plan \
  -var="repo_url=https://github.com/YOUR_USERNAME/monolithic-devops.git"

terraform apply \
  -var="repo_url=https://github.com/YOUR_USERNAME/monolithic-devops.git"
```

## 9. Important behavior to understand

- The EC2 instance's user-data script runs during instance initialization, not on every Terraform apply.
- In this example, passing a new Git commit SHA changes the deployment version and replaces the instance. This is simple to learn, but it is not the most efficient deployment strategy.
- The website is public over HTTP. Do not use this setup for sensitive applications; HTTPS, restricted access, monitoring and a more robust deployment method should be added for production.
- AWS charges may apply while the EC2 instance and other resources are running. Run `terraform destroy` when finished experimenting, after confirming the resources and state you intend to remove.


## You only need to replace or configure these values in the files.

|File|Value to change|What to enter|
|---|---|---|
|`terraform/variables.tf`|`aws_region`|Your AWS region, e.g. `ap-south-1` for Mumbai|
|`terraform/variables.tf`|`repo_url`|Your GitHub repository URL|
|`terraform/main.tf`|S3 backend `bucket`|Your unique S3 bucket name for Terraform state|
|GitHub Secrets|`AWS_ROLE_ARN`|ARN of the IAM role configured for GitHub Actions OIDC|

Example values:

```
AWS Region:
ap-south-1

GitHub Repository:
https://github.com/YOUR_USERNAME/monolithic-devops.git

S3 Bucket:
my-unique-terraform-state-2026

GitHub Secret AWS_ROLE_ARN:
arn:aws:iam::123456789012:role/GitHubActionsTerraformRole
```

The account ID, username, bucket name and role ARN above are placeholders. Replace them with your actual values.

Before running: create the S3 bucket, configure AWS GitHub OIDC and IAM permissions, and ensure your repository contains the `site/` and `terraform/` directories. You do not need to change the EC2 AMI lookup or the basic HTML/CSS files.





# 👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾







# Understanding `terraform/main.tf`

The easiest way to understand `main.tf` is to think of it as a set of instructions telling AWS what infrastructure to create for your website.
In your project, Terraform needs to:

1. Connect to AWS.
2. Find a suitable Ubuntu operating system image.
3. Find a network where the EC2 instance can run.
4. Allow visitors to access the website.
5. Create the EC2 instance and run your website setup script.

Let's break down the code you were given, section by section.

## 1. Tell Terraform which version and provider to use

```
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

What does this mean?

- `terraform` — contains Terraform's own configuration.
- `required_version` — specifies the Terraform versions your project accepts.
- `required_providers` — tells Terraform which provider plugin it needs.
- `hashicorp/aws` — the plugin Terraform uses to communicate with AWS.
- `version = "~> 5.0"` — selects a compatible AWS provider version in the 5.x series.

Think of the AWS provider as a translator between Terraform and AWS.
Without it, Terraform would not know how to create AWS resources using the configuration you wrote.

## 2. Connect Terraform to your AWS region

```
provider "aws" {
  region = var.aws_region
}
```

This tells Terraform to use the AWS provider in the region you configured.
Here, `var.aws_region` means the value comes from a variable defined in `variables.tf`.
For example:

```
variable "aws_region" {
  default = "ap-south-1"
}
```

`ap-south-1` is the AWS Mumbai region.
You could also write `region = "ap-south-1"` directly, but using a variable makes it easier to change the region later.
Important: This block selects the region; it does not authenticate you. AWS credentials must be configured separately, such as through GitHub Actions OIDC.

## 3. Find the default VPC

```
data "aws_vpc" "default" {
  default = true
}
```

First, understand what a VPC is.
A VPC (Virtual Private Cloud) is a logically isolated network inside AWS where you can launch resources such as EC2 instances.
This block tells Terraform to look up the default VPC that already exists in your AWS account and selected region.
Let's understand the syntax: `data "aws_vpc" "default"`

- `data` means read information about an existing resource.
- `aws_vpc` tells Terraform what type of information to retrieve.
- `default` is a name you choose to refer to this lookup elsewhere in your code.
- `default = true` asks for the default VPC.

Notice the difference:

- `resource` — creates or manages infrastructure.
- `data` — looks up existing information.

This block does not create a VPC.

## 4. Find the subnets inside that VPC

```
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```

A subnet is a smaller network inside a VPC. An EC2 instance is launched into a subnet.
Let's examine the important line: `values = [data.aws_vpc.default.id]`
This means: take the ID of the VPC we found in the previous step and use it to find the subnets belonging to that VPC.
For example, imagine AWS returns:

```
VPC ID:     vpc-0123456789
Subnet IDs: subnet-111111, subnet-222222
```

Terraform looks up the subnets associated with that VPC.
Later, the EC2 resource uses: `subnet_id = data.aws_subnets.default.ids[0]`
The `[0]` means select the first subnet ID from the returned list.
You don't need to type the actual subnet ID yourself.

## 5. Find an Ubuntu AMI

```
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = [
      "ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"
    ]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

Before creating an EC2 instance, AWS needs to know which operating system to install.
An AMI (Amazon Machine Image) is a template used to launch an EC2 instance. It contains the operating system and associated image configuration.
This block searches for an appropriate Ubuntu image.

- `aws_ami` — looks up an Amazon Machine Image.
- `most_recent = true` — chooses the newest matching image.
- `owners` — restricts the search to the specified image publisher, in this case Canonical, Ubuntu's publisher.
- The first `filter` searches for Ubuntu 24.04 LTS 64-bit server images.
- The second `filter` selects the HVM virtualization type.

The `*` at the end of the image name is a wildcard, allowing the AMI version-specific suffix to vary.
Terraform retrieves the matching AMI ID automatically. You don't need to hardcode an ID that might differ between AWS regions.

## 6. Create a security group

```
resource "aws_security_group" "website" {
  name        = "portfolio-website-sg"
  description = "Allow HTTP traffic"
  vpc_id      = data.aws_vpc.default.id
```

A security group acts like a virtual firewall for your EC2 instance.
Unlike the previous `data` blocks, this uses `resource` because Terraform will create a new security group.
The `vpc_id` connects that security group to the VPC found earlier.

### Allow incoming HTTP traffic

```
ingress {
  description = "HTTP"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

This rule allows incoming web traffic over HTTP.

|Setting|Meaning|
|---|---|
|`ingress`|Incoming network traffic|
|`from_port = 80`|Start of allowed port range|
|`to_port = 80`|End of allowed port range|
|`protocol = "tcp"`|Use TCP|
|`0.0.0.0/0`|Allow connections from any IPv4 address|

Why port 80? Because the Nginx web server will serve your website over HTTP on port 80.

### Allow outgoing traffic

```
egress {
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```

This allows all outbound traffic from the instance to any IPv4 destination. Your instance needs outbound connectivity to download packages and retrieve your website from GitHub.

Security note: Port 80 is publicly accessible in this example. For a real production website, you would generally use HTTPS and restrict access where appropriate.

## 7. Create the EC2 instance

This is the most important part of your `main.tf`.

```
resource "aws_instance" "website" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = var.instance_type
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.website.id]
  associate_public_ip_address = true
```

The `resource "aws_instance" "website"` block tells Terraform to create an EC2 instance.
Each setting controls part of that instance.

| Setting                       | Meaning                                     |
| ----------------------------- | ------------------------------------------- |
| `ami`                         | Which operating system image to use         |
| `instance_type`               | The instance's CPU and memory configuration |
| `subnet_id`                   | Which subnet to launch it in                |
| `vpc_security_group_ids`      | Which security group to attach              |
| `associate_public_ip_address` | Whether to assign a public IPv4 address     |

Let's look at how the references work. `ami = data.aws_ami.ubuntu.id`
Terraform gets the AMI ID from section 5. `instance_type = var.instance_type`
Terraform gets the instance type from `variables.tf`, for example `t3.micro`. `vpc_security_group_ids = [aws_security_group.website.id]`
Terraform attaches the security group created in section 6.
The `aws_security_group.website.id` expression references a resource managed by Terraform. You don't need to look up or manually enter its ID.

## 8. Tell Terraform when to replace the instance

```
user_data_replace_on_change = true
```

This setting matters for your deployment process.
It tells Terraform to replace the EC2 instance if the configured user data changes.
User data is the script that runs when the instance starts for its initial setup.
Your GitHub Actions workflow also supplies a deployment version, so a changed commit SHA changes the user data configuration and triggers replacement.
The practical result is that a new instance is created for a new deployment version. This is easy to demonstrate in a beginner project, but it can change your public IP address.

## 9. Run the website setup script

```
user_data = templatefile("${path.module}/user_data.sh", {
  repo_url       = var.repo_url
  deploy_version = var.deploy_version
})
```

This connects Terraform to your `user_data.sh` file.
What is user data?
User data is a script AWS runs when an EC2 instance is initially launched. In your project, that script installs Nginx and downloads the website from GitHub.
Let's break down this expression:

- `user_data` — passes a startup script to EC2.
- `templatefile()` — reads a file and substitutes variables into it.
- `${path.module}` — refers to the directory containing this Terraform module, which is your `terraform/` directory.
- `user_data.sh` — the script file to load.
- `repo_url` — the GitHub repository URL supplied by Terraform.
- `deploy_version` — the deployment version supplied by Terraform.

For example, Terraform takes: `repo_url = var.repo_url`
and makes the value available inside the script as: `${repo_url}`
The deployment version is also supplied to the script, although the current script only assigns it to a variable.
This is how your Terraform configuration connects infrastructure creation to website deployment.

## 10. Add tags to the EC2 instance

```
tags = {
  Name        = "Monolithic-Portfolio"
  Environment = "Dev"
}
```

Tags are labels used to identify and organize AWS resources.
For example, in the AWS console, you can find the instance by its `Name` tag, `Monolithic-Portfolio`.
The `Environment` tag tells you that it belongs to your development environment.
These tags are useful when you have multiple EC2 instances and need to identify which project they belong to.

## How the entire `main.tf` connects

1. Provider -> Select AWS region and provider
2. Data sources -> Find VPC, subnets and Ubuntu AMI
3. Security group -> Configure network access on port 80
4. EC2 instance -> Launch Ubuntu and run user_data.sh
5. Website is running -> Nginx serves your GitHub website files

The key distinction to remember is:

- `provider` tells Terraform where and how to manage resources.
- `data` looks up existing information.
- `resource` creates or manages infrastructure.
- `var` retrieves a configurable value.
- `user_data` passes the startup script to EC2.

Once you understand these five concepts, most of the `main.tf` file becomes much easier to read.