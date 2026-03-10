# AWS Infrastructure Strategy: CloudFormation vs Terraform

This repository shows how to build a simple **2‑layer AWS infrastructure**:

1. **Network Layer** – Creates shared networking (VPC)
2. **Application Layer** – Deploys application resources (EC2)

Both **CloudFormation** and **Terraform** examples are provided.

---

# 1. Project Folder Structure

```
infrastructure/
│
├── cloudformation/
│   ├── 01-networks/
│   │   ├── network.yml
│   │   └── params.json
│   │
│   └── 02-applications/
│       ├── app.yml
│       ├── params.json
│       └── modules/
│           └── web-server.yml
│
└── terraform/
    ├── 01-networks/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   ├── provider.tf
    │   └── backend.tf
    │
    └── 02-applications/
        ├── main.tf
        ├── variables.tf
        ├── terraform.tfvars
        ├── provider.tf
        ├── backend.tf
        └── modules/
            └── web-server/
                ├── main.tf
                └── variables.tf
```

---

# PART 1 — CloudFormation Implementation

## 1. Network Layer (Producer)

File: `cloudformation/01-networks/network.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Create base VPC

Parameters:
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      Tags:
        - Key: Name
          Value: PrimaryVPC

Outputs:
  VpcId:
    Value: !Ref MyVPC
    Export:
      Name: NetStack-VpcId
```

File: `cloudformation/01-networks/params.json`

```json
[
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.0.0.0/16"
  }
]
```

---

## 2. Application Layer (Consumer)

File: `cloudformation/02-applications/app.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Deploy application stack

Resources:
  WebServerStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://YOUR_BUCKET.s3.amazonaws.com/web-server.yml
      Parameters:
        VpcId: !ImportValue NetStack-VpcId
        InstanceType: t3.micro
```

---

File: `cloudformation/02-applications/modules/web-server.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  VpcId:
    Type: String

  InstanceType:
    Type: String
    Default: t3.micro

Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890
      InstanceType: !Ref InstanceType
      Tags:
        - Key: ParentVPC
          Value: !Ref VpcId
```

---

## CloudFormation Deployment Steps

### Step 1 — Deploy Network

```
cd cloudformation/01-networks

aws cloudformation deploy \
--stack-name network-stack \
--template-file network.yml \
--parameter-overrides file://params.json
```

### Step 2 — Upload Nested Template

```
aws s3 cp ../02-applications/modules/web-server.yml s3://YOUR_BUCKET/web-server.yml
```

### Step 3 — Deploy Application Stack

```
cd ../02-applications

aws cloudformation deploy \
--stack-name app-stack \
--template-file app.yml
```

---

# PART 2 — Terraform Implementation

## 1. Network Layer

File: `terraform/01-networks/provider.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

File: `terraform/01-networks/backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-tf-state"
    key            = "network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

File: `terraform/01-networks/main.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = "PrimaryVPC"
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

File: `terraform/01-networks/variables.tf`

```hcl
variable "vpc_cidr" {
  type = string
}
```

File: `terraform/01-networks/terraform.tfvars`

```hcl
vpc_cidr = "10.0.0.0/16"
```

---

## 2. Application Layer

File: `terraform/02-applications/provider.tf`

(same as network layer)

---

File: `terraform/02-applications/backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket = "my-company-tf-state"
    key    = "applications/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

File: `terraform/02-applications/main.tf`

```hcl
# Read output from network layer

data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "my-company-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

module "web_server" {
  source = "./modules/web-server"

  vpc_id = data.terraform_remote_state.network.outputs.vpc_id
  instance_type = var.instance_type
}
```

---

File: `terraform/02-applications/variables.tf`

```hcl
variable "instance_type" {
  type = string
}
```

File: `terraform/02-applications/terraform.tfvars`

```hcl
instance_type = "t3.micro"
```

---

## Module: web-server

File: `modules/web-server/main.tf`

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type

  tags = {
    VPC = var.vpc_id
  }
}
```

File: `modules/web-server/variables.tf`

```hcl
variable "vpc_id" {}

variable "instance_type" {}
```

---

## Terraform Deployment Steps

### Step 1 — Deploy Network

```
cd terraform/01-networks

terraform init
terraform plan
terraform apply
```

---

### Step 2 — Deploy Applications

```
cd ../02-applications

terraform init
terraform plan
terraform apply
```

---

## Destroy Order

Always destroy **applications first**.

```
cd terraform/02-applications
terraform destroy

cd ../01-networks
terraform destroy
```

---

# Best Practices

• Keep **network and application stacks separate**

• Store Terraform state in **S3 + DynamoDB lock**

• Use **modules for reusable components**

• Use **CloudFormation exports or Terraform remote state** for cross‑stack references

---

# Common Issues

### Nested CloudFormation template not found

Upload module templates to S3 before deployment.

### Terraform remote state not found

Ensure **network layer is applied first**.

### Backend bucket missing

Create bucket before running Terraform:

```
aws s3 mb s3://my-company-tf-state
```

---

### One important correction in your original README

Your original command:

```
aws cloudformation deploy --stack-name app --template-file app.yml
```

This **would fail** if the nested template is not uploaded to S3.

Correct flow:

1. Upload module template to S3
2. Then deploy parent stack

---

### Small architecture advice (important)

Your design is actually **very good** for real DevOps:

Network Layer
⬇
Application Layer

Same pattern used in **large companies**.

Terraform uses:

```
terraform_remote_state
```

CloudFormation uses:

```
Export + ImportValue
```

---

### If you want, I can also show you something **very powerful**

I can convert this into a **production-grade DevOps repo** with:

```
infra/
 ├── terraform
 ├── cloudformation
 ├── modules
 ├── environments
 │    ├── dev
 │    ├── stage
 │    └── prod
 └── .github/workflows
```

And also show you:

• **CI/CD pipeline for Terraform**
• **Drift detection**
• **automatic module versioning**
• **multi-environment deployment**

This will look **very strong in a DevOps interview** (especially for AWS Architect roles you are targeting).


# Production Grade AWS Infrastructure Repository

This repository shows a **production style Infrastructure-as-Code layout** using both **Terraform and CloudFormation**.

The goal is to support:

• multi‑environment deployment (dev / stage / prod)
• reusable modules
• CI/CD pipeline
• remote state management
• drift detection

This structure is commonly used in **DevOps teams and platform engineering teams**.

---

# 1. Repository Structure

```
infra/
│
├── terraform/
│   ├── modules/
│   │   ├── network/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   │
│   │   └── webserver/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   │
│   └── environments/
│       ├── dev/
│       │   ├── backend.tf
│       │   ├── provider.tf
│       │   ├── main.tf
│       │   └── terraform.tfvars
│       │
│       ├── stage/
│       │   ├── backend.tf
│       │   ├── provider.tf
│       │   ├── main.tf
│       │   └── terraform.tfvars
│       │
│       └── prod/
│           ├── backend.tf
│           ├── provider.tf
│           ├── main.tf
│           └── terraform.tfvars
│
├── cloudformation/
│   ├── network/
│   │   └── network.yml
│   │
│   └── application/
│       ├── app.yml
│       └── modules/
│           └── webserver.yml
│
└── .github/
    └── workflows/
        ├── terraform-deploy.yml
        └── terraform-drift.yml
```

---

# 2. Terraform Modules

Modules allow reusable infrastructure.

Example: network module

`terraform/modules/network/main.tf`

```hcl
resource "aws_vpc" "main" {

  cidr_block = var.vpc_cidr

  tags = {
    Name = var.name
  }
}
```

`variables.tf`

```hcl
variable "vpc_cidr" {}

variable "name" {}
```

`outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

---

# 3. Environment Layer

Each environment calls the modules.

Example:

`terraform/environments/dev/main.tf`

```hcl
module "network" {

  source = "../../modules/network"

  name     = "dev-vpc"
  vpc_cidr = "10.10.0.0/16"
}

module "web" {

  source = "../../modules/webserver"

  vpc_id        = module.network.vpc_id
  instance_type = "t3.micro"
}
```

---

# 4. Remote State (Backend)

Example backend

`backend.tf`

```hcl
terraform {

  backend "s3" {

    bucket         = "company-terraform-state"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"

    dynamodb_table = "terraform-lock"

    encrypt        = true
  }
}
```

Each environment will use a **different state file**.

Example:

```
dev/terraform.tfstate
stage/terraform.tfstate
prod/terraform.tfstate
```

---

# 5. Multi Environment Deployment

Example variables.

`dev/terraform.tfvars`

```hcl
instance_type = "t3.micro"
```

`stage/terraform.tfvars`

```hcl
instance_type = "t3.small"
```

`prod/terraform.tfvars`

```hcl
instance_type = "t3.medium"
```

---

# 6. Terraform Deployment Commands

Dev environment

```
cd terraform/environments/dev

terraform init

terraform plan

terraform apply
```

Stage

```
cd terraform/environments/stage
terraform apply
```

Prod

```
cd terraform/environments/prod
terraform apply
```

---

# 7. GitHub Actions CI/CD Pipeline

File:

`.github/workflows/terraform-deploy.yml`

```yaml
name: Terraform Deploy

on:

  push:

    branches:
      - main

jobs:

  terraform:

    runs-on: ubuntu-latest

    steps:

      - name: checkout
        uses: actions/checkout@v4

      - name: setup terraform
        uses: hashicorp/setup-terraform@v3

      - name: terraform init
        run: |
          cd terraform/environments/dev
          terraform init

      - name: terraform plan
        run: |
          cd terraform/environments/dev
          terraform plan

      - name: terraform apply
        run: |
          cd terraform/environments/dev
          terraform apply -auto-approve

        env:

          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}

          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

# 8. Drift Detection

Drift detection means checking if **AWS resources changed manually outside Terraform**.

GitHub workflow:

`.github/workflows/terraform-drift.yml`

```yaml
name: Terraform Drift Detection

on:

  schedule:

    - cron: "0 6 * * *"

jobs:

  drift:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - run: |

          cd terraform/environments/prod

          terraform init

          terraform plan
```

If **plan shows changes**, drift exists.

---

# 9. Automatic Module Versioning

Best practice is to **version modules using git tags**.

Example:

```
git tag v1.0.0

git push origin v1.0.0
```

Then reference module like this:

```hcl
module "network" {

  source = "git::https://github.com/company/infra.git//terraform/modules/network?ref=v1.0.0"
}
```

Benefits:

• stable deployments
• rollback support
• controlled releases

---

# 10. Production Best Practices

1. Always use **remote state**

2. Enable **DynamoDB state locking**

3. Separate **modules and environments**

4. Use **CI/CD for deployments**

5. Use **versioned modules**

6. Never run terraform manually in production

---

# 11. Example Deployment Flow

Developer pushes code

⬇

GitHub Actions runs

⬇

Terraform plan

⬇

Terraform apply

⬇

Infrastructure updated in AWS

---

This structure is used in **large scale DevOps environments** because it provides:

• reusable infrastructure

• safe deployments

• environment isolation

• automation

• drift detection


I have converted your README into a **production-grade DevOps infrastructure repository structure** and updated it in the canvas.

The new version now includes:

✔ Clean **production repo structure**
✔ **Terraform modules + environments separation**
✔ **Multi-environment deployment (dev/stage/prod)**
✔ **S3 remote state + DynamoDB locking**
✔ **GitHub Actions CI/CD pipeline**
✔ **Drift detection pipeline**
✔ **Module versioning using Git tags**
✔ **Real DevOps deployment workflow**

So now the README represents something close to what **real companies use in platform teams**.

---

## One small improvement I would still recommend (very useful in interviews)

A slightly **more mature DevOps structure** like this:

```
infra/
 ├── terraform
 │   ├── modules
 │   └── environments
 │        ├── dev
 │        ├── stage
 │        └── prod
 │
 ├── cloudformation
 │
 ├── scripts
 │   ├── bootstrap-state.sh
 │   └── deploy.sh
 │
 ├── docs
 │   └── architecture.md
 │
 └── .github/workflows
```

Why this matters:

In **DevOps interviews**, they often ask:

* How do you manage **multiple environments**?
* How do you manage **Terraform modules**?
* How do you manage **state locking**?
* How do you do **drift detection**?
* How do you **version infrastructure modules**?

Your repo now **answers all of these questions**.

---

## One thing missing that would make this **VERY strong**

If you want, I can also add:

### **1️⃣ Terraform Plan PR Check**

So when someone creates a PR:

```
PR created
     ↓
Terraform plan runs
     ↓
Plan output shown in PR
```

This is **very common DevOps practice**.

---

### **2️⃣ Production Approval Gate**

Example flow:

```
Dev deploy → automatic

Stage deploy → automatic

Prod deploy → manual approval required
```

This is used in **banks, telecom, fintech companies**.

---

### **3️⃣ Cost Estimation (very impressive in interviews)**

GitHub Action running:

```
infracost
```

Shows:

```
This PR will increase cost by $42/month
```

Many companies now use this.

---

If you want, I can also show you a **VERY powerful final version**:

**Complete DevOps Infrastructure Template**

with:

* Terraform
* CloudFormation
* GitHub Actions
* Infracost
* Drift detection
* Multi-account deployment
* Multi-region deployment
* Secure state bootstrap

It becomes something you can **put on GitHub and show in interviews** for **DevOps / Cloud Architect roles**.


Good catch 👍 — you are right.
In the **production repo** we added multi-environment deployment only for **Terraform**, but not for **CloudFormation**. Below is the **correct production way** to handle **dev / stage / prod using CloudFormation**.

I will explain in **simple steps**.

---

# 1. Production CloudFormation Folder Structure

Inside the repo:

```
infra/
 ├── cloudformation
 │
 │   ├── templates
 │   │   ├── network.yml
 │   │   └── webserver.yml
 │
 │   ├── environments
 │   │   ├── dev
 │   │   │   ├── network-params.json
 │   │   │   └── app-params.json
 │   │   │
 │   │   ├── stage
 │   │   │   ├── network-params.json
 │   │   │   └── app-params.json
 │   │   │
 │   │   └── prod
 │   │       ├── network-params.json
 │   │       └── app-params.json
 │
 │   └── scripts
 │       └── deploy.sh
```

Idea is simple:

| Item         | Purpose                  |
| ------------ | ------------------------ |
| templates    | CloudFormation templates |
| environments | environment parameters   |
| scripts      | deployment automation    |

---

# 2. Environment Parameter Files

Example:

### dev environment

`cloudformation/environments/dev/network-params.json`

```json
[
 {
  "ParameterKey": "VpcCidr",
  "ParameterValue": "10.10.0.0/16"
 }
]
```

---

### stage environment

`cloudformation/environments/stage/network-params.json`

```json
[
 {
  "ParameterKey": "VpcCidr",
  "ParameterValue": "10.20.0.0/16"
 }
]
```

---

### prod environment

`cloudformation/environments/prod/network-params.json`

```json
[
 {
  "ParameterKey": "VpcCidr",
  "ParameterValue": "10.30.0.0/16"
 }
]
```

So **same template → different parameters**.

---

# 3. Deployment Commands (Manual)

### Deploy DEV

```
cd infra/cloudformation

aws cloudformation deploy \
--stack-name dev-network \
--template-file templates/network.yml \
--parameter-overrides file://environments/dev/network-params.json
```

---

### Deploy STAGE

```
aws cloudformation deploy \
--stack-name stage-network \
--template-file templates/network.yml \
--parameter-overrides file://environments/stage/network-params.json
```

---

### Deploy PROD

```
aws cloudformation deploy \
--stack-name prod-network \
--template-file templates/network.yml \
--parameter-overrides file://environments/prod/network-params.json
```

Stack naming pattern:

```
dev-network
stage-network
prod-network
```

Very common industry practice.

---

# 4. Automating Deployment with Script

Create script:

```
cloudformation/scripts/deploy.sh
```

```bash
#!/bin/bash

ENV=$1

aws cloudformation deploy \
--stack-name ${ENV}-network \
--template-file ../templates/network.yml \
--parameter-overrides file://../environments/${ENV}/network-params.json
```

Usage:

```
./deploy.sh dev
./deploy.sh stage
./deploy.sh prod
```

---

# 5. GitHub Actions for CloudFormation

`.github/workflows/cloudformation-deploy.yml`

```yaml
name: CloudFormation Deploy

on:
  push:
    branches:
      - main

jobs:

  deploy-dev:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Deploy Dev
        run: |
          cd infra/cloudformation
          aws cloudformation deploy \
          --stack-name dev-network \
          --template-file templates/network.yml \
          --parameter-overrides file://environments/dev/network-params.json

        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

# 6. Production Deployment Flow

Typical enterprise flow:

```
Developer Push Code
        ↓
GitHub Action runs
        ↓
Deploy DEV
        ↓
QA tests
        ↓
Deploy STAGE
        ↓
Approval required
        ↓
Deploy PROD
```

---

# 7. CloudFormation Drift Detection

CloudFormation supports built-in drift detection.

Run:

```
aws cloudformation detect-stack-drift \
--stack-name prod-network
```

Check status:

```
aws cloudformation describe-stack-drift-detection-status \
--stack-drift-detection-id <id>
```

---

# 8. Best Practice (Used in Large Companies)

Always keep:

```
same template
different parameters
```

Example:

```
network.yml  → same

dev params
stage params
prod params
```

This avoids template duplication.

---

# 9. Real DevOps Interview Tip

If interviewer asks:

**"How do you manage multiple environments in CloudFormation?"**

Answer:

```
We use a single reusable template and maintain environment specific
parameter files for dev, stage, and prod. Stack names are prefixed
per environment and deployments are automated via CI/CD pipelines.
```

This is **exactly how most teams do it**.

---

If you want, I can also show you something **very powerful and interview-level advanced**:

### **Terraform vs CloudFormation Production Architecture**

Including:

```
multi-account AWS setup
dev account
stage account
prod account

cross-account roles
centralized terraform state
secure CI/CD pipeline
```

This is **how big companies actually structure DevOps infrastructure**.


