Here is a clean, professional-looking GitHub-flavored Markdown (.md) version of your notes. You can copy-paste this into a file called something like `AWS-DevOps-Demos-Reference.md` and keep it in your personal repo or notes folder.

```markdown
# AWS DevOps – Demo Commands & Reference Flows

Personal quick-reference notes for AWS DevOps workflows (VPC, CodeDeploy, CodePipeline, SAM, ECS/Fargate).  
Maintained for interview discussions, screen-sharing, and future personal projects.

Current region used in most examples: **ap-south-1** (Mumbai)

---

## 1. AWS CLI Authentication (Best Practices)

```bash
# Option 1: AWS SSO (recommended for orgs with SSO)
aws sso login --profile kunalshrivastava

# Option 2: aws-vault (good for local machines)
aws-vault exec kunalshrivastava --duration=8h

# Always verify who you are
aws sts get-caller-identity
```

**Talking point**: "I prefer AWS SSO or aws-vault over long-lived access keys for security. I always run `get-caller-identity` before starting any critical operation."

---

## 2. VPC – CloudFormation Deployment

```bash
cd "C:\kunal\cloudthat training devops\scope\vpc"

# Validate template first (very important!)
aws cloudformation validate-template --template-body file://Enterprise_VPC_Complete_working.yaml

# Create stack (example with parameter)
aws cloudformation create-stack \
  --stack-name LabVPC-Demo \
  --parameters ParameterKey=InstanceType,ParameterValue=t2.micro \
  --template-body file://Enterprise_VPC_Complete_working.yaml \
  --disable-rollback

# Monitor status (run repeatedly)
aws cloudformation describe-stacks \
  --stack-name LabVPC-Demo \
  --query "Stacks[0].StackStatus" \
  --output text

# Drift detection (very useful in interviews)
aws cloudformation describe-stack-resource-drifts --stack-name LabVPC-Demo

# Safe update workflow – ChangeSet
aws cloudformation create-change-set \
  --stack-name LabVPC-Demo \
  --change-set-name ReviewChanges \
  --template-body file://Enterprise_VPC_Complete_working.yaml

aws cloudformation describe-change-set \
  --stack-name LabVPC-Demo \
  --change-set-name ReviewChanges

# Execute only after review
# aws cloudformation execute-change-set --stack-name LabVPC-Demo --change-set-name ReviewChanges

# Cleanup (when done)
# aws cloudformation delete-stack --stack-name LabVPC-Demo
```

**Talking point**: "I always validate templates and use ChangeSets for production-like safety. Drift detection helps catch manual changes."

---

## 3. CodeDeploy (EC2 + Blue/Green potential)

### Pre-requisites
- EC2 tagged: `Name = my-web-server`
- IAM role attached: `AmazonSSMManagedInstanceCore`

```bash
cd "C:\kunal\cloudthat training devops\scope\codedeploy"

# Install CodeDeploy agent via SSM (tag-based targeting)
aws ssm send-command \
  --region ap-south-1 \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=my-web-server" \
  --parameters 'commands=[
    "yum install -y ruby wget",
    "cd /tmp",
    "wget https://aws-codedeploy-ap-south-1.s3.ap-south-1.amazonaws.com/latest/install",
    "chmod +x install",
    "./install auto",
    "systemctl enable codedeploy-agent || true",
    "systemctl start codedeploy-agent || true"
  ]' \
  --comment "Install CodeDeploy Agent"

# Check status (use CommandId from previous output)
aws ssm list-command-invocations \
  --command-id <CommandId> \
  --details \
  --region ap-south-1

# Prepare artifact bucket & push
aws s3 mb s3://web-cd-artifacts-ks-demo --region ap-south-1
aws deploy push \
  --application-name my-web-server-app \
  --source web \
  --s3-location s3://web-cd-artifacts-ks-demo/web.zip \
  --region ap-south-1

# Create deployment
aws deploy create-deployment \
  --application-name my-web-server-app \
  --s3-location bucket=web-cd-artifacts-ks-demo,key=web.zip,bundleType=zip \
  --deployment-group-name my-web-server-app-dg \
  --region ap-south-1

# Monitor
aws deploy get-deployment \
  --deployment-id <DeploymentId> \
  --region ap-south-1
```

**Talking point**: "For production I recommend blue/green deployments with traffic shifting. Common troubleshooting: SSM role missing, no VPC endpoint for S3, or yum repo issues."

---

## 4. CodePipeline – Basic S3 Artifact Flow

```bash
cd "C:\kunal\cloudthat training devops\scope\codepipeline"

aws s3 mb s3://web-codepipeline-artifacts-ks-demo --region ap-south-1
aws s3 cp web.zip s3://web-codepipeline-artifacts-ks-demo/web.zip
```

→ Then verify pipeline execution in AWS Console (Source → Build → Deploy)

---

## 5. AWS SAM (Serverless)

```bash
cd "C:\kunal\cloudthat training devops\scope\aws sam\sam-app"

# Normal development flow
git status
git add .
git commit -m "Update Lambda function - added feature X"
git push

# SAM local testing (great for interviews)
# sam local invoke "FunctionName" -e event.json
```

**Talking point**: "SAM is CloudFormation + extras for serverless. I like `sam local` for fast iteration before pushing to pipeline."

---

## 6. ECS / Fargate + Pipeline

```bash
cd "C:\kunal\cloudthat training devops\scope\ecs ecr fargate\MY_WEB_APP_DEMO"

# Typical flow
git add .
git commit -m "Update container image version v1.2.3"
git push

# Pipeline should:
# → Build & push → ECR
# → Update ECS service (blue/green or rolling)
```

**Final check-points**
- ECR image pushed?
- CodePipeline stages green?
- ALB DNS: `my-webapp-alb-531517662.ap-northeast-1.elb.amazonaws.com`

---

**General Interview Talking Points**
- Prefer infrastructure-as-code → CloudFormation / SAM / CDK
- Tag strategy is key for automation (SSM, deployments, cost allocation)
- Always verify identity (`sts get-caller-identity`)
- Use ChangeSets for CloudFormation safety
- Prefer blue/green for zero-downtime deployments
- SSM > direct SSH for production EC2 management

Feel free to expand sections with diagrams (draw.io) or more advanced patterns later.
```

This format looks structured, uses proper headings, code blocks, bullet points, and talking points — perfect for screen-sharing during interviews. It shows organization, best practices, and depth without being overwhelming.

Good luck with your interviews! 🚀
