
# AWS Infrastructure Strategy: CloudFormation vs. Terraform

This document provides a complete reference for managing a 2-layer decoupled infrastructure (Networking & Applications) using both AWS Native and Terraform patterns.

---

## 1. Folder Structure (The Midmap)

```text
infrastructure/
├── 01-networks/                
│   ├── network.yml             
│   ├── params.json             <-- Network Parameter Values
│   ├── provider.tf             
│   ├── main.tf                 
│   ├── vars.tf                 
│   ├── terraform.tfvars        
│   └── backend.tf              
│
└── 02-applications/            
    ├── app.yml                 <-- Parent Template
    ├── params.json             <-- NEW: App Parameter Values
    ├── provider.tf             
    ├── main.tf                 
    ├── vars.tf                 
    ├── backend.tf              
    └── modules/                
        └── web-server/         
            ├── web-server.yml  <-- Child Template
            ├── main.tf         
            └── vars.tf

```

---

## 2. Layer 01: Networking (The Producer)

### **CloudFormation: `network.yml**`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
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
  ExportedVpcId:
    Value: !Ref MyVPC
    Export:
      Name: NetStack-VpcId

```

### **CloudFormation: `params.json**`

```json
[
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.0.0.0/16"
  }
]

```

---

### **Terraform: `main.tf**`

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "PrimaryVPC" }
}

output "vpc_id" {
  value = aws_vpc.main.id
}

```

### **Terraform: `vars.tf**`

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

```

### **Terraform: `terraform.tfvars**`

```hcl
vpc_cidr = "10.0.0.0/16"

```

### **Terraform: `backend.tf**`

```hcl
terraform {
  backend "s3" {
    bucket         = "my-company-tf-state"
    key            = "prod/network.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock" # Prevents concurrent applies
  }
}

```

### The Provider (01-networks/provider.tf)
You need this in every root folder to tell Terraform which version of the AWS plugin to use.

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

---

## 3. Layer 02: Applications (The Consumer)

### **CloudFormation: `app.yml**`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # Calling the Nested Stack
  MyNestedWebServer:
    Type: AWS::CloudFormation::Stack
    Properties:
      # NOTE: In AWS, this URL must point to where you uploaded web-server.yml in S3
      TemplateURL: https://my-bucket.s3.amazonaws.com/modules/web-server.yml
      Parameters:
        VpcId: !ImportValue NetStack-VpcId
        InstanceType: t3.micro 

```

---

### **Terraform: `main.tf` (Root)**

```hcl
# CROSS-STACK: Fetch Networking outputs
data "terraform_remote_state" "net" {
  backend = "s3"
  config = {
    bucket = "my-company-tf-state"
    key    = "prod/network.tfstate"
    region = "us-east-1"
  }
}

# NESTED STACK: Call Local Module
module "web_app" {
  source    = "./modules/web-server"
  vpc_id    = data.terraform_remote_state.net.outputs.vpc_id
  inst_type = var.instance_type
}

```

### **Terraform: `modules/web-server/main.tf` (Child)**

```hcl
resource "aws_instance" "server" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.inst_type
  # In a real scenario, you'd use a Subnet ID here
  tags = {
    VPC_Parent = var.vpc_id
  }
}

```

### **Terraform: `modules/web-server/vars.tf**`

```hcl
variable "vpc_id" { type = string }
variable "inst_type" { type = string }

```

###  02-applications/modules/web-server/web-server.yml
This is the reusable component that defines the actual EC2 instance.

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

## 4. Operational Commands

### **Deploy Sequence**

1. **Network Layer:**
* `cd 01-networks`
* **TF:** `terraform init && terraform apply -auto-approve`
* **CFN:** `aws cloudformation deploy --stack-name net --template-file network.yml --parameter-overrides file://params.json`


2. **App Layer:**
* `cd ../02-applications`
* **TF:** `terraform init && terraform apply -auto-approve`
* **CFN:** `aws cloudformation deploy --stack-name app --template-file app.yml`



### **Teardown Sequence**

*Must be done in reverse order:*

1. `cd 02-applications && terraform destroy`
2. `cd 01-networks && terraform destroy`


Here is the final addition to your `README.md`. This **GitHub Actions** workflow handles the logic we discussed: it navigates to the folders in the correct sequence, initializes the backend, and applies the changes.

I've also included a **Safety Check**—the pipeline for the `02-applications` layer won't run unless the `01-networks` layer has successfully completed.

---

## 5. Automation: GitHub Actions Workflow (`.github/workflows/deploy.yml`)

Add this file to your repository to automate the "Command Sequence" we defined.

```yaml
# This command uploads local nested files to S3 and updates the Parent template
aws cloudformation package \
  --template-file app.yml \
  --s3-bucket my-templates-bucket \
  --output-template-file packaged-app.yml

# Then deploy the generated file
aws cloudformation deploy \
  --template-file packaged-app.yml \
  --stack-name prod-app-stack \
  --capabilities CAPABILITY_IAM
```

```yaml
name: 'Infrastructure Deployment'

on:
  push:
    branches: [ "main" ]
  pull_request:

jobs:
  # STEP 1: Deploy Networking (The Foundation)
  deploy-network:
    name: '01-Networks Layer'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init & Apply
        run: |
          cd 01-networks
          terraform init
          terraform apply -auto-approve
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  # STEP 2: Deploy Applications (Depends on Networking)
  deploy-app:
    name: '02-Applications Layer'
    needs: deploy-network  # Ensures Network is ready first
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init & Apply
        run: |
          cd 02-applications
          terraform init
          terraform apply -auto-approve
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

```

---

## 6. Summary of Your Stack Strategy

### **Why this works for your AI Agent Project:**

1. **Modularity:** Your Python agent can generate code for the `web-server` module independently without breaking the VPC.
2. **Traceability:** Every change is recorded in the S3 bucket defined in your `backend.tf`.
3. **Safety:** If the `01-networks` layer fails, GitHub Actions stops immediately, preventing "Orphaned" applications that have no network to attach to.

### **Final Checklist before Copy-Pasting:**

* [ ] Create an S3 bucket named `my-company-tf-state` (or update the `backend.tf` files with your bucket name).
* [ ] Add your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to your **GitHub Repo Settings > Secrets**.
* [ ] Ensure your `params.json` and `terraform.tfvars` have the correct CIDR blocks for your specific AWS Region.

---

This completes your comprehensive technical guide! Is there any specific **Python automation script** or **CloudWatch monitoring** logic you'd like to add to this README before you finalize it?
---

Would you like me to add a **GitHub Actions Workflow (.yml)** that automatically runs these commands whenever you push code to your repo?
