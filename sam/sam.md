# AWS SAM CI/CD Pipeline – End-to-End Deployment Guide

## Overview

This project demonstrates how to deploy a **serverless Node.js / Python application using AWS SAM with CI/CD**.

The system uses:

* AWS SAM (Serverless Application Model)
* AWS Lambda
* Amazon API Gateway
* AWS CodePipeline
* AWS CodeBuild
* AWS CloudFormation
* AWS CodeDeploy (for traffic shifting)

The application returns a simple **HTML page through Lambda**, deployed using a **SAM template and automated pipeline**.

---

# End-to-End Story

Initially, the **DevOps / Platform team** sets up the infrastructure and CI/CD pipeline.

After the platform is ready:

* The **SAM template becomes part of the application repository**
* Developers modify application code
* Every commit triggers the CI/CD pipeline
* The pipeline builds and deploys the application automatically

### Responsibilities

**Developers**

* Write Lambda code
* Update HTML / Node.js / Python logic
* Commit code to repository

**DevOps**

* Maintain SAM templates
* Maintain CI/CD pipeline
* Manage IAM permissions
* Ensure safe deployments and rollbacks

**Simple rule**

> Developers write code. DevOps builds the road.

---

# Step 1 – Initial Infrastructure Setup

DevOps team creates:

* SAM template (`template.yaml`)
* CI/CD pipeline (CodePipeline + CodeBuild + CloudFormation)

### Initialize SAM project

```bash
cd ~/environment/
sam init --runtime python3.12
cd sam-app
```

### Build the project

```bash
sam build
```

### Test Lambda locally

```bash
sam local invoke HelloWorldFunction --event events/event.json
```

### Start local API

```bash
sam local start-api -p 8000
```

Test locally:

```
http://127.0.0.1:8000/
```

### Create S3 bucket for deployment

```bash
labBucket=lab4-sam-[YOUR-INITIALS]-[YOUR-POSTAL-CODE]

aws s3 mb s3://$labBucket
```

### Package SAM application

```bash
sam package \
--output-template-file packaged.yaml \
--s3-bucket $labBucket
```

### Deploy CloudFormation stack

```bash
sam deploy \
--template-file packaged.yaml \
--stack-name sam-app \
--capabilities CAPABILITY_IAM
```

### Verify deployment

```bash
aws cloudformation describe-stacks \
--stack-name sam-app \
--query 'Stacks[].Outputs[?OutputKey==`HelloWorldApi`]' \
--output table
```

Confirm **CREATE_COMPLETE** in AWS Console.

### Purpose

* Create baseline infrastructure
* Validate SAM template
* Create initial CloudFormation stack

---

# Step 2 – Template Ownership and Development

The SAM template becomes part of the **application repository**.

Developers may update:

* Lambda code
* Runtime (Node.js / Python)
* Function configuration (memory / timeout)

Developers **do not modify CI/CD pipeline logic**.

Important rule:

> Developers change applications, not platform infrastructure.

---

# Step 3 – Code Commit Triggers Pipeline

When developers commit code to Git:

* AWS CodeCommit or GitHub triggers **CodePipeline automatically**

---

# Step 4 – Build Stage

CodeBuild executes:

```bash
sam build
aws cloudformation package
```

Purpose:

* Package Lambda code
* Upload artifacts to S3
* Generate `outputtemplate.yaml`

---

# Step 5 – CloudFormation Change Set

The pipeline creates a **CloudFormation Change Set**.

This shows:

* Which resources will change
* Which resources will be replaced or updated

This allows **safe deployment planning**.

---

# Step 6 – Execute Change Set

Pipeline executes the Change Set.

CloudFormation updates the stack.

If deployment preferences exist:

* CodeDeploy manages **traffic shifting**

Deployment strategies include:

* Canary deployment
* Linear deployment

---

# Step 7 – DevOps Responsibilities

DevOps engineers **do not manually deploy code every time**.

Instead they ensure:

* Pipeline reliability
* IAM permissions
* Safe deployment strategy
* Monitoring and rollback
* Environment consistency

---

# AWS SAM Basics

## What is AWS SAM?

AWS Serverless Application Model (SAM) is an extension of CloudFormation that simplifies building and deploying serverless applications.

---

## Why use SAM instead of CloudFormation?

SAM provides:

* Simplified syntax
* Built-in best practices
* Local testing
* Native CI/CD integration
* Traffic shifting support

---

## What does `sam init` do?

Creates a new serverless project including:

* Sample Lambda function
* SAM template
* Project structure

---

## What does `sam build` do?

* Installs dependencies
* Builds Lambda artifacts
* Places them inside `.aws-sam/build`

---

## How to test SAM locally

Test Lambda:

```
sam local invoke
```

Run API Gateway locally:

```
sam local start-api
```

Docker is required for local testing.

---

# S3 Requirement for Deployment

Lambda artifacts must be uploaded to S3 before CloudFormation can deploy them.

---

## What does `sam package` do?

* Uploads Lambda artifacts to S3
* Generates `packaged.yaml`

---

## What does `sam deploy` do?

Creates or updates a **CloudFormation stack** which deploys:

* Lambda
* API Gateway
* IAM Roles
* Other AWS resources

---

# CI/CD Pipeline Architecture

Pipeline services used:

| Service        | Purpose                 |
| -------------- | ----------------------- |
| CodeCommit     | Source control          |
| CodeBuild      | Build and package SAM   |
| CloudFormation | Deploy infrastructure   |
| CodeDeploy     | Manage traffic shifting |

---

## Pipeline Trigger

Pipeline runs automatically when:

* A developer pushes code to the repository

---

## buildspec.yml

Defines CodeBuild steps.

Example:

```yaml
version: 0.2

phases:
  build:
    commands:
      - export BUCKET=lab4-sam-<INITIALS>-<ZIP>
      - aws cloudformation package \
        --template-file template.yaml \
        --s3-bucket $BUCKET \
        --output-template-file outputtemplate.yaml

artifacts:
  files:
    - outputtemplate.yaml
```

---

# Change Sets and Deployment

### Why use Change Sets?

They allow previewing infrastructure changes before execution.

---

### Executing Change Sets

Pipeline creates the Change Set and then executes it automatically using another CloudFormation action.

CLI alternative:

```
aws cloudformation execute-change-set
```

---

# Traffic Shifting and Blue-Green Deployment

Traffic shifting gradually moves users from the old Lambda version to the new version.

Example:

```
10% → New version
90% → Old version
```

Benefits:

* Reduces risk
* Allows quick rollback
* Minimizes customer impact

---

## Deployment Strategy

This project uses **Blue-Green deployment with traffic shifting**.

Traffic shifting is managed by **AWS CodeDeploy using Lambda aliases**.

---

## Observing Traffic Shifting

During deployment:

Refreshing the API endpoint shows alternating responses from:

* Blue environment
* Green environment

---

## Rollback Behavior

If deployment fails:

* CodeDeploy automatically rolls back
* Traffic returns to the previous Lambda version

---

# Versioning and Redeployment

Pipeline redeploys when code changes.

Example:

```
lab4a.html → lab4b.html
```

The new commit triggers another pipeline execution.

---

# Lambda + HTML Deployment Model

The HTML file is bundled inside the Lambda package.

Example structure:

```
project/
 ├ template.yaml
 └ src/
    ├ app.js
    └ index.html
```

Example Lambda code:

```javascript
const fs = require('fs');
const path = require('path');

exports.handler = async () => {

  const html = fs.readFileSync(
    path.join(__dirname, 'index.html'),
    'utf8'
  );

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'text/html' },
    body: html
  };

};
```

Lambda reads the HTML file and returns it through API Gateway.

---

# Why `appspec.yml` Is Not Needed

For Lambda deployments:

| Reason             | Explanation             |
| ------------------ | ----------------------- |
| No server          | Lambda is fully managed |
| No file copy       | AWS handles deployment  |
| No service restart | Lambda versions         |
| Traffic control    | Lambda aliases          |

Therefore:

* No lifecycle hooks
* No shell scripts
* No appspec.yml

---

# Traffic Splitting in Lambda

```
User Request
      ↓
API Gateway
      ↓
Lambda Alias
      ↓
90% Old Version
10% New Version
```

Both versions run temporarily during deployment.

---

# Cleanup of Old Versions

After successful deployment:

* Traffic reaches 100% on new version
* Old version is no longer referenced
* AWS eventually removes unused versions

No manual cleanup required.

---

# DevOps Interview Ready Summary

**Deployment process**

1. DevOps deploys initial infrastructure using SAM
2. SAM template lives with application code
3. Developers update application logic
4. Git commit triggers CI/CD pipeline
5. Pipeline builds application using `sam build`
6. CloudFormation Change Set is created
7. Stack is updated with traffic shifting via CodeDeploy

DevOps responsibility is maintaining **SAM, pipeline, IAM, and deployment safety**, not writing application code.

---

# Quick Memory Rule

Node.js + HTML on Lambda → **HTML lives inside the Lambda deployment package**
