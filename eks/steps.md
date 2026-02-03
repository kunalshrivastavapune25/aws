Sure 👍 I’ll keep it **very simple, short, and step-by-step** (layman style).

Below is the **simplest AWS CodePipeline setup for EKS deployment**.

---

## Big Picture (1-line)

**GitHub → CodeBuild → EKS (kubectl apply)**

That’s it. No Jenkins, no extra tools.

---

## What you need (minimum)

1. **EKS cluster already running**
2. **Kubernetes YAML files** (deployment, service, etc.)
3. **GitHub repo**
4. **IAM role with EKS access**

---

## Simple Git Repo Structure

```
repo
 ├─ app/
 │   └─ index.html
 ├─ k8s/
 │   ├─ deployment.yaml
 │   └─ service.yaml
 └─ buildspec.yml
```

---

## Step 1: Create IAM Role for CodeBuild

**AWS Console → IAM → Roles → Create role**

* Trusted entity: **CodeBuild**
* Attach policies:

  * `AmazonEKSClusterPolicy`
  * `AmazonEKSWorkerNodePolicy`
  * `AmazonEC2ContainerRegistryReadOnly`
  * (Optional) `AdministratorAccess` (for learning only)

👉 This allows CodeBuild to talk to EKS.

---

## Step 2: Allow CodeBuild to Access EKS

On your local machine (one time):

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks
```

Edit **aws-auth configmap** in EKS and add CodeBuild role:

```bash
kubectl edit configmap aws-auth -n kube-system
```

Add:

```yaml
- rolearn: arn:aws:iam::<ACCOUNT_ID>:role/CodeBuildRole
  username: codebuild
  groups:
    - system:masters
```

✅ Very important step (many people miss this).

---

## Step 3: buildspec.yml (VERY SIMPLE)

This file runs `kubectl apply`.

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Installing kubectl
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.29.0/2023-11-14/bin/linux/amd64/kubectl
      - chmod +x kubectl
      - mv kubectl /usr/local/bin/

  pre_build:
    commands:
      - echo Updating kubeconfig
      - aws eks update-kubeconfig --region ap-south-1 --name my-eks

  build:
    commands:
      - echo Deploying to EKS
      - kubectl apply -f k8s/
```

👉 That’s all. No Docker needed if image already exists.

---

## Step 4: Create CodeBuild Project

**AWS Console → CodeBuild → Create project**

* Source: **GitHub**
* Environment:

  * Image: **Amazon Linux 2**
  * Privileged mode: ❌ No
* Service role: **CodeBuildRole**
* Buildspec: **Use buildspec.yml from repo**

Save.

---

## Step 5: Create CodePipeline

**AWS Console → CodePipeline → Create pipeline**

### Stage 1: Source

* Source provider: **GitHub**
* Repo & branch

### Stage 2: Build

* Build provider: **AWS CodeBuild**
* Select the CodeBuild project

❌ No Deploy stage needed (kubectl already deploys)

---

## Final Flow (Super Simple)

```
Git push
   ↓
CodePipeline
   ↓
CodeBuild
   ↓
kubectl apply
   ↓
EKS updated
```

---

## If You Have Docker Image

* Add Docker build & push to ECR in `buildspec.yml`
* Update image tag
* Then `kubectl apply`

(We can do this next if you want)

---

## Common Mistakes (Quick)

❌ CodeBuild role not added in aws-auth
❌ Wrong EKS region
❌ kubectl version mismatch

---

If you want, next I can:

* Add **ECR + Docker**
* Show **Blue/Green / Rolling update**
* Or give **full AWS Console click-by-click steps**

Just tell me 👍
