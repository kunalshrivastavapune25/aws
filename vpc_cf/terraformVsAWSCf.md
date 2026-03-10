
# AWS Infrastructure Strategy: CloudFormation vs. Terraform

This document outlines the architectural patterns for managing dependencies, hierarchies, and external variables.

---

## 1. Architectural Concepts Mapping

| Concept | AWS CloudFormation | Terraform / OpenTofu |
| --- | --- | --- |
| **Declaration** | `Parameters:` section in `.yaml` | `variables.tf` / `vars.tf` |
| **Value Assignment** | JSON Parameter File | `terraform.tfvars` |
| **Cross-Stack** | `Export` & `Fn::ImportValue` | `terraform_remote_state` |
| **Nested Stack** | `AWS::CloudFormation::Stack` | `module { source = "..." }` |

---

## 2. Variable Configuration Samples

### **Terraform: `01-networks/vars.tf**`

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

```

### **CloudFormation: `01-networks/params.json` (The `.tfvars` equivalent)**

Unlike Terraform, CloudFormation typically uses a JSON format for external parameter files when using the CLI.

```json
[
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.1.0.0/16"
  }
]

```

---

## 3. Deployment Commands with Variable Files

### **Phase 1: Deploy Networking**

**Terraform (using `.tfvars`):**

```bash
cd 01-networks
# Terraform automatically loads terraform.tfvars. 
# For custom files use:
terraform apply -var-file="prod.tfvars" -auto-approve

```

**CloudFormation (using `params.json`):**

```bash
cd 01-networks
aws cloudformation deploy \
  --stack-name prod-network \
  --template-file network.yml \
  --parameter-overrides file://params.json

```

---

### **Phase 2: Deploy Application**

**Terraform:**

```bash
cd ../02-applications
terraform init
terraform apply -auto-approve

```

**CloudFormation:**

```bash
cd ../02-applications
aws cloudformation deploy \
  --stack-name prod-app \
  --template-file app.yml \
  --parameter-overrides file://app-params.json

```

---

## 4. Full Directory Midmap (Final)

```text
infrastructure/
├── 01-networks/                
│   ├── network.yml             
│   ├── params.json             <-- CFN Variable Values
│   ├── main.tf                 
│   ├── vars.tf                 <-- TF Variable Declaration
│   ├── terraform.tfvars        <-- TF Variable Values
│   └── backend.tf              
│
└── 02-applications/            
    ├── app.yml                 
    ├── app-params.json         <-- CFN Variable Values
    ├── main.tf                 
    ├── vars.tf                 
    └── modules/                
        └── web-server/         
            ├── main.tf         
            └── vars.tf         

```

---

## 5. Operations & Lifecycle Summary

| Action | CloudFormation CLI | Terraform CLI |
| --- | --- | --- |
| **Apply with Vars** | `aws cloudformation deploy ... --parameter-overrides file://x.json` | `terraform apply -var-file="x.tfvars"` |
| **Check Drift** | `aws cloudformation detect-stack-drift` | `terraform plan` |
| **Rollback** | Automatic (Check `ROLLBACK_COMPLETE`) | Manual (`git revert` + `apply`) |

---

### **Pro-Tip on Naming:**

In Terraform, if you name your file `terraform.tfvars`, it loads **automatically**. For CloudFormation, you must **always** explicitly point to your JSON parameter file in the `aws cloudformation deploy` command using `file://`.

Would you like me to help you create a **Python automation script** that detects which environment you are in and automatically picks the right `.json` or `.tfvars` file for you?
