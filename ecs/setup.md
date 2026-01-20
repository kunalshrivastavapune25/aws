Here are the **manual console steps** to recreate a similar environment in your own AWS account (not the lab one). This setup closely matches the lab's configuration for **ECS Fargate + CodeDeploy blue/green deployments**.

The lab provides a pre-built VPC, subnets, security group, ALB + two target groups, ECS cluster, ECR repo, IAM roles, CodeCommit repo, etc. In your own account, you need to create most of these yourself (except the cluster can be created during service creation, but it's better to create it first).

### 1. Choose Region (very important)
- Use the same region as the lab (most likely **ap-northeast-1** / Tokyo — check your lab resources).
- All steps below → do them in **that region**.

### 2. Create VPC + Subnets + Internet Gateway + Route Tables
ECS Fargate with `assignPublicIp: ENABLED` (like in the lab) → needs **public subnets**.

**Recommended simple pattern (like most labs):**
- 2 public subnets (different AZs)
- No private subnets + no NAT (cheaper, internet-facing tasks)

**Console steps:**

1. Go to **VPC** → **Your VPCs** → **Create VPC**
   - Name: `my-webapp-vpc`
   - IPv4 CIDR: `10.0.0.0/16` (or `172.31.0.0/16`)
   - Tenancy: Default
   - Create → VPC only (do **not** use "VPC and more" wizard yet)

2. **Create Internet Gateway**
   - VPC → Internet Gateways → Create internet gateway
   - Name: `my-igw`
   - Create → Actions → Attach to VPC → select `my-webapp-vpc`

3. **Create 2 public subnets** (different AZs)
   - VPC → Subnets → Create subnet
   - VPC: your new vpc
   - Subnet 1:
     - Name: `public-a`
     - Availability Zone: choose first AZ (e.g. ap-northeast-1a)
     - IPv4 CIDR: `10.0.1.0/24`
   - Subnet 2:
     - Name: `public-b`
     - AZ: second AZ (e.g. ap-northeast-1c)
     - CIDR: `10.0.2.0/24`

4. **Create Route Table for public subnets**
   - VPC → Route Tables → Create route table
   - Name: `public-rt`
   - VPC: your vpc
   - Create

5. Add route to Internet Gateway
   - Select the new route table → Routes tab → Edit routes
   - Add route: Destination `0.0.0.0/0` → Target = Internet Gateway → your-igw
   - Save

6. Associate both public subnets to this route table
   - Subnet associations tab → Edit subnet associations → select both public subnets

7. **Enable Auto-assign public IPv4** on both subnets (very important for Fargate + assignPublicIp=ENABLED)
   - Subnets → select each subnet → Actions → Edit subnet settings
   - Enable auto-assign public IPv4 address → yes

Now you have:
- VPC
- 2 public subnets
- Internet Gateway
- Public route table

### 3. Create Security Group for ECS tasks
Tasks need outbound internet (pull from ECR, etc.) + inbound HTTP from ALB.

1. VPC → Security Groups → Create security group
   - Name: `ecs-tasks-sg`
   - Description: for my-webapp fargate tasks
   - VPC: your vpc

2. Inbound rules:
   - Type: HTTP, Protocol TCP, Port 80
   - Source: Anywhere-IPv4 (0.0.0.0/0)  ← temporary (later restrict to ALB SG)

3. Outbound rules: allow all (default)

### 4. Create Application Load Balancer + **two** Target Groups (mandatory for CodeDeploy blue/green)

**Target groups first:**

1. EC2 → Target Groups → Create target group (×2)

   **Target Group 1 – Blue / Production**
   - Target type: **ip** (Fargate uses awsvpc → ip targets)
   - Name: `myapp-tg-blue`
   - Protocol: HTTP, Port: 80
   - VPC: your vpc
   - Health checks: HTTP, path `/` (or `/index.html`)
   - Create

   **Target Group 2 – Green / Replacement**
   - Same settings
   - Name: `myapp-tg-green`
   - Create

**Now create ALB:**

1. EC2 → Load Balancers → Create Load Balancer → Application Load Balancer
   - Name: `my-webapp-alb`
   - Scheme: **Internet-facing**
   - IP address type: IPv4
   - VPC: your vpc
   - Mappings → select both public subnets (Availability Zones)
   - Security groups → select a new SG for ALB (or create one allowing HTTP 80 from anywhere)

2. **Listeners**
   - HTTP : 80 → forward to **myapp-tg-blue** (initial production target group)

   → Do **NOT** add second listener yet (CodeDeploy will use test listener in some configs, but basic setup only needs production listener + two TGs)

3. Create

→ Note down:
   - ALB DNS name (you'll use it later to test)
   - **Target Group ARNs** (you'll need them for `create-service.json`)

### 5. Create ECS Cluster
1. ECS → Clusters → Create Cluster
   - Cluster name: `my-webapp-cluster`
   - Infrastructure: **AWS Fargate (serverless)**
   - Create

(No need to create capacity providers manually — Fargate is default)

### 6. Create ECR Repository
1. ECR → Repositories → Create repository
   - Visibility: Private
   - Repository name: `my-webapp-repo`
   - Create

→ Note down repository URI:  
`123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/my-webapp-repo`

### 7. Create IAM Roles (important!)

**A. ECS Task Execution Role** (pulls images from ECR, CloudWatch Logs, etc.)

1. IAM → Roles → Create role
   - Trusted entity: AWS service → Elastic Container Service → Elastic Container Service Task
   - Permissions: attach **AmazonECSTaskExecutionRolePolicy** (managed)
   - Name: `ecsTaskExecutionRole`
   - Create

**B. CodeDeploy Service Role** (for blue/green)

1. IAM → Roles → Create role
   - Trusted entity: AWS service → CodeDeploy
   - Permissions: attach **AWSCodeDeployRoleForECS** (managed policy)
   - Name: `CodeDeployForECS`
   - Create

**C. CodeBuild Service Role** (needs ECR push + logs + S3)

Use managed policy: **AWSCodeBuildAdminAccess** or more minimal:
- `AmazonEC2ContainerRegistryPowerUser`
- `CloudWatchLogsFullAccess`
- `AWSCodePipelineApprover` (if needed)

**D. CodePipeline Service Role**

Usually auto-created when you create pipeline, or use existing:  
`AWSCodePipelineServiceRole-<region>-my-webapp-pipeline`

### 8. Next steps after infrastructure is ready

You can now follow the lab guide, but using **your own** values instead of lab-provided ones:

- Put your **ECR URI** into `buildspec.yaml`
- Put your **executionRoleArn** into `taskdef.json`
- Put your **two target group ARNs**, **your subnets**, **your security group** into `create-service.json`
- When creating CodeDeploy application & deployment group → select **your** cluster, service, ALB, and the **two target groups**
- In CodePipeline deploy action → select your taskdef.json placeholder `<IMAGE1_NAME>`

### Quick Summary Table – What you created vs lab-provided

| Resource                        | Lab (pre-created) | You need to create? | Console location          |
|-------------------------------|-------------------|----------------------|----------------------------|
| VPC                             | yes               | yes                  | VPC                        |
| Subnets (2 public)              | yes               | yes                  | VPC → Subnets              |
| Internet Gateway                | yes               | yes                  | VPC                        |
| Route table + 0.0.0.0 → IGW     | yes               | yes                  | VPC                        |
| Security Group for tasks        | yes               | yes                  | VPC → Security Groups      |
| ALB                             | yes               | yes                  | EC2 → Load Balancers       |
| Target Groups (2×)              | yes               | yes                  | EC2 → Target Groups        |
| ECS Cluster                     | yes               | yes (or auto)        | ECS                        |
| ECR repo                        | yes               | yes                  | ECR                        |
| IAM roles (task exec, codedeploy)| yes              | yes                  | IAM                        |

Good luck with your own deployment!

If something fails (especially permission or networking), tell me the exact error — we can debug step-by-step.
