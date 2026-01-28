# AWS DevOps Demo – Commands, Flows & Interview Notes
> **Purpose**: Personal reference notes maintained for interview discussions, demos, and real project execution.  
> **Style**: CLI-first, production-aware, and security-focused.
---
## 1. AWS CLI Authentication (Best Practices)
### Why this matters
* Secure, auditable access
* Avoids static access keys
* Preferred in enterprise environments
### Recommended Methods
```bash
# Using AWS SSO (Enterprise preferred)
aws sso login --profile kunalshrivastava
# OR using aws-vault (temporary credentials)
aws-vault exec kunalshrivastava --duration=8h
# Always verify identity
aws sts get-caller-identity
```
> "Verify identity using STS before executing any infra or deployment command."
---

## 2. VPC Creation Using CloudFormation

### Objective

* Infrastructure as Code (IaC)
* Repeatable and reviewable network setup

### Steps

```bash
cd "C:\kunal\cloudthat training devops\scope\vpc"

aws --version
aws cloudformation validate-template \
  --template-body file://Enterprise_VPC_Complete_working.yaml
```

### Create Stack

```bash
aws cloudformation create-stack \
  --stack-name LabVPC-Demo \
  --parameters ParameterKey=InstanceType,ParameterValue=t2.micro \
  --template-body file://Enterprise_VPC_Complete_working.yaml \
  --disable-rollback
```

### Monitor Stack Status

```bash
aws cloudformation describe-stacks \
  --stack-name LabVPC-Demo \
  --query "Stacks[0].StackStatus" \
  --output text
```

### Drift Detection (Very Important in Prod)

```bash
aws cloudformation describe-stack-resource-drifts \
  --stack-name LabVPC-Demo
```

### Safe Updates using ChangeSets

```bash
aws cloudformation create-change-set \
  --stack-name LabVPC-Demo \
  --change-set-name ReviewChanges \
  --template-body file://Enterprise_VPC_Complete_working.yaml

aws cloudformation describe-change-set \
  --stack-name LabVPC-Demo \
  --change-set-name ReviewChanges
```

> Execute only after review approval.

### Cleanup

```bash
aws cloudformation delete-stack --stack-name LabVPC-Demo
```


> "I prefer ChangeSets in production to avoid accidental resource replacement."

---

## 3. Common Prerequisites (Reusable Across Projects)

* Tag EC2 instances (used by SSM, CodeDeploy, automation)
* Attach **AmazonSSMManagedInstanceCore** role
* Deploy agents using SSM (no SSH)
* Create:

  * CodeDeploy Application & Deployment Group
  * S3 bucket for artifacts
  * Security Groups for ECS
  * Target Groups (Blue/Green ready)
  * Application Load Balancer (ALB)
  * ECS Cluster + ECR Repository
  * IAM Roles for ECS Tasks & Services

---

## 4. Practice 2 – AWS CodeDeploy (EC2 + Agent)

### Scenario

* EC2 based deployment
* Foundation for Blue/Green strategy

### EC2 Requirements

* Tag: `Name = my-web-server`
* IAM Role: `AmazonSSMManagedInstanceCore`

### Install CodeDeploy Agent via SSM

```bash
cd "C:\kunal\cloudthat training devops\scope\codedeploy"

aws ssm send-command \
  --region ap-south-1 \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=my-web-server" \
  --parameters "commands=yum install -y ruby wget,cd /tmp,wget https://aws-codedeploy-ap-south-1.s3.ap-south-1.amazonaws.com/latest/install,chmod +x install,./install auto,systemctl enable codedeploy-agent || true,systemctl start codedeploy-agent || true" \
  --comment "Install CodeDeploy Agent" \
  --output text
```

### Verify Command Execution

```bash
aws ssm list-command-invocations \
  --command-id <COMMAND_ID> \
  --details \
  --region ap-south-1
```

### Prepare Deployment Artifact

```bash
aws s3 mb s3://web-cd-artifacts-ks-demo --region ap-south-1

aws deploy push \
  --application-name my-web-server-app \
  --source web \
  --s3-location s3://web-cd-artifacts-ks-demo/web.zip \
  --region ap-south-1
```

### Deploy Application

```bash
aws deploy create-deployment \
  --application-name my-web-server-app \
  --deployment-group-name my-web-server-app-dg \
  --s3-location bucket=web-cd-artifacts-ks-demo,key=web.zip,bundleType=zip \
  --region ap-south-1
```

### Monitor Deployment

```bash
aws deploy get-deployment \
  --deployment-id <DEPLOYMENT_ID> \
  --region ap-south-1
```


> "If agent fails, first check SSM role, VPC endpoints, and yum repo access. In production, prefer Blue/Green."

---

## 5. Practice 3 – AWS CodePipeline (CI/CD)

### Flow

`Source → Build → Deploy`

```bash
cd "C:\kunal\cloudthat training devops\scope\codepipeline"

aws s3 mb s3://web-codepipeline-artifacts-ks-demo --region ap-south-1
aws s3 cp web.zip s3://web-codepipeline-artifacts-ks-demo/web.zip
```

### Validation

* Open AWS Console → CodePipeline
* Verify stages:

  * Source (S3 / CodeCommit)
  * Build (CodeBuild)
  * Deploy (CodeDeploy / ECS)

---

## 6. Practice 4 – AWS SAM (Serverless CI/CD)

### Why SAM

* Simplifies Lambda + API Gateway
* Built on top of CloudFormation

```bash
cd "C:\kunal\cloudthat training devops\scope\aws sam\sam-app"

git status
git commit -m "Update Lambda function - added feature X"
git push
```

### Verification

* Pipeline trigger in CodePipeline / SAM Pipeline
* CloudFormation Outputs
* Lambda / API Gateway endpoint


> "SAM makes serverless faster with local testing using `sam local invoke`."

---

## 7. Practice 5 – ECS / Fargate Pipeline

### Flow

`Git → CodePipeline → ECR → ECS → ALB`

```bash
cd "C:\kunal\cloudthat training devops\scope\ecs ecr fargate\MY_WEB_APP_DEMO"

git add .
git commit -m "Update container image version"
git push
```

### Verification

* ECR: New image pushed
* CodePipeline: All stages green
* ECS Service: New task revision
* ALB DNS reachable

```text
ALB DNS Example:
my-webapp-alb-531517662.ap-northeast-1.elb.amazonaws.com
```

> "This setup supports zero-downtime deployments using ALB and rolling updates on Fargate."

---
