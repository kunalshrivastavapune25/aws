Here is the **complete end-to-end setup** for AWS CodeDeploy (EC2/On-Premises) using only the **AWS Management Console UI** (no CLI commands).  

Follow these steps in order. All actions are done in the browser at https://console.aws.amazon.com/.

### Step 1: Create IAM Roles (Required for CodeDeploy)

#### 1A. Create CodeDeploy **Service Role** (allows CodeDeploy to manage deployments)
1. Go to **IAM** service → In the left menu, click **Roles** → Click **Create role**.
2. **Trusted entity type** → Select **AWS service**.
3. **Use case** → From the dropdown/search, choose **CodeDeploy** → Select the **CodeDeploy** use case (for EC2/On-Premises).
4. Click **Next**.
5. **Permissions policies** → Search for and check **AWSCodeDeployRole** (AWS managed policy) → Click **Next**.
6. **Role name** → Enter e.g. `CodeDeployServiceRole` (or any clear name).
7. (Optional) Add description and tags.
8. Review → Click **Create role**.
9. After creation, note/copy the **Role ARN** (you'll need it later).

#### 1B. Create EC2 **Instance Role** + Instance Profile (allows EC2 to be managed by CodeDeploy and access S3)
1. Still in **IAM** → **Roles** → Click **Create role**.
2. **Trusted entity type** → **AWS service**.
3. **Use case** → Choose **EC2**.
4. Click **Next**.
5. **Permissions policies** → Attach these two (search and check):
   - `AmazonS3ReadOnlyAccess` (to download your app ZIP from S3)
   - `service-role/AmazonEC2RoleforAWSCodeDeploy` (AWS managed, for CodeDeploy agent)
6. Click **Next**.
7. **Role name** → Enter e.g. `EC2CodeDeployRole`.
8. Create the role.
9. Now go to left menu → **Instance Profiles** → Click **Create instance profile**.
10. **Instance profile name** → Enter e.g. `EC2CodeDeployInstanceProfile`.
11. Click **Create**.
12. Select your new instance profile → Click **Add role** → Choose the role you just created (`EC2CodeDeployRole`) → **Add role**.

### Step 2: Launch Linux EC2 Instance (Amazon Linux 2 or 2023 recommended)
1. Go to **EC2** service → Click **Launch instance**.
2. **Name and tags** → Name: e.g. `WebServer`  
   Add tag: Key = `Name`, Value = `WebServer` (important for later targeting).
3. **Application and OS Images (Amazon Machine Image)** → Choose **Amazon Linux** → Select **Amazon Linux 2023** or **Amazon Linux 2** (free tier eligible).
4. **Instance type** → `t2.micro` or `t3.micro` (free tier).
5. **Key pair** → Choose existing or create new (you'll need this for SSH).
6. **Network settings** → 
   - Use default VPC/subnet or your own.
   - Security group → Allow SSH (port 22) from your IP.
   - (Optional) Allow HTTP (80) if your app needs it.
7. **Advanced details** → Scroll down → **IAM instance profile** → Select the one you created (`EC2CodeDeployInstanceProfile`).
8. Review → Click **Launch instance**.
9. Wait 2–5 minutes → Go to **Instances** → Select your instance → Note the **Public IPv4 address** (you'll SSH later).

### Step 3: Install CodeDeploy Agent on the EC2 Instance
(You must do this manually via SSH — console doesn't install it automatically)

1. SSH into your instance:
   - From your local machine: `ssh -i your-key.pem ec2-user@your-instance-public-ip`
2. Run these commands one by one (for **Amazon Linux 2023/2** — region `ap-northeast-1`):
   ```bash
   sudo yum update -y
   sudo yum install ruby wget -y
   cd /home/ec2-user
   wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   sudo systemctl start codedeploy-agent     # or: sudo service codedeploy-agent start
   sudo systemctl status codedeploy-agent     # or: sudo service codedeploy-agent status
   ```
3. Make sure it says **active (running)**.  
   If error → check logs: `sudo cat /var/log/aws/codedeploy-agent/codedeploy-agent.log`

### Step 4: Create CodeDeploy Application
1. Go to **CodeDeploy** service.
2. Left menu → **Applications** → Click **Create application**.
3. **Application name** → Enter `web` (or `weapp` — match what you want to use).
4. **Compute platform** → Select **EC2/On-premises**.
5. Click **Create application**.

### Step 5: Create Deployment Group
1. On your new application page → Click **Create deployment group**.
2. **Deployment group name** → Enter `webdg`.
3. **Service role** → Select the CodeDeploy service role you created earlier (`CodeDeployServiceRole`).
4. **Environment configuration** → Choose **Amazon EC2 instances**.
5. **EC2 tag** → 
   - Key: `Name`
   - Value: `WebServer`  
   (this targets your instance via the tag you added)
6. **Deployment type** → Choose **In-place** (simpler for beginners).
7. **Deployment configuration** → Choose **CodeDeployDefault.OneAtATime** (safest for single instance).
8. (Optional) Leave other settings default.
9. Click **Create deployment group**.

### Step 6: Create First Deployment (Using Your S3 ZIP)
1. On your application page → Click **Create deployment**.
2. **Deployment group** → Select `webdg`.
3. **Revision type** → Choose **My application is stored in Amazon S3**.
4. **S3 location** → Paste:
   ```
   s3://web-codedeploy-artifacts-ks-411045/web.zip
   ```
5. **Bundle type** → Select **.zip**.
6. (Important) **ETag** → Enter exactly: `b79bf8ce410b89f74cb104f4c8e52557`  
   (get fresh ETag from S3 console if needed: S3 → bucket → web.zip → Metadata tab)
7. **Deployment description** → Optional.
8. Click **Create deployment**.

### Step 7: Monitor & Verify
1. On the deployment page → Watch status (should go → In Progress → Succeeded).
2. If failed → Click the deployment ID → View **Events** / **Logs** tab.
3. SSH to instance → Check files in `/opt/myapp` and logs in `/opt/codedeploy-agent/deployment-root/...`.

### Quick Troubleshooting Tips (Console)
- Deployment fails with "No instances"? → Double-check EC2 tag matches exactly + agent is running.
- Permission errors? → Verify both roles have correct policies attached.
- Agent issues? → SSH and restart: `sudo systemctl restart codedeploy-agent`

This should get your `web` application deployed to the EC2 instance via CodeDeploy — all through the beautiful AWS Console UI!  

If any step shows an error, tell me the exact message/screen, and I'll help debug. Good luck, Kunal! 🚀








