
## Cloud DevOps - Commands & Flows

### 1. AWS Authentication 
```text
   → Use AWS SSO or aws-vault for secure access
   aws sso login --profile kunalshrivastava1
   # OR
   aws-vault exec kunalshrivastava1 --duration=8h 
   aws sts get-caller-identity   # quick identity check

   → Always verify: aws sts get-caller-identity
```

### 2. VPC + CloudFormation (IaC Best Practices)
```text
   cd "C:\kunal\cloudthat training devops\scope\vpc"
   aws --version
   aws cloudformation validate-template --template-body file://Enterprise_VPC_Complete_working_ecs.yaml

   # Create stack (dry-run like)
   aws cloudformation create-stack --stack-name LabVPC-Demo `
     --parameters ParameterKey=InstanceType,ParameterValue=t2.micro `
     --template-body file://Enterprise_VPC_Complete_working_ecs.yaml `
     --disable-rollback --region ap-northeast-1
	 
   aws cloudformation create-stack --stack-name LabVPC-Demo --parameters ParameterKey=InstanceType,ParameterValue=t2.micro --template-body file://Enterprise_VPC_Complete_working_ecs.yaml --disable-rollback --region ap-northeast-1 --capabilities CAPABILITY_NAMED_IAM	 

   # Monitor (repeat until CREATE_COMPLETE)
   aws cloudformation describe-stacks --stack-name LabVPC-Demo --query "Stacks[0].StackStatus" --output text --region ap-northeast-1

   # Drift detection 
   aws cloudformation describe-stack-resource-drifts --stack-name LabVPC-Demo --region ap-northeast-1

   # ChangeSet for safe updates (show safety!)
   aws cloudformation create-change-set --stack-name LabVPC-Demo --change-set-name ReviewChanges --template-body file://Enterprise_VPC_Complete_working_ecs.yaml  --region ap-northeast-1

   aws cloudformation describe-change-set --stack-name LabVPC-Demo --change-set-name ReviewChanges  --region ap-northeast-1

   # Execute only if approved
   # aws cloudformation execute-change-set --stack-name LabVPC-Demo --change-set-name ReviewChanges  --region ap-northeast-1

   # Cleanup 
   # aws cloudformation delete-stack --stack-name LabVPC-Demo  --region ap-northeast-1
   https://app.diagrams.net/#G1PNVyq6ni_eR1YdgFZg4Iu7ovo6vKe22p#%7B%22pageId%22%3A%22-5Oxv9spdVYTsKXrqEpt%22%7D    
```http://54.248.101.211/index.html
```
### 3. CodeDeploy (EC2 + Agent + Blue/Green potential) 
```text
UseCase: When direct deployment needed without pipeline	
   → Tag EC2: Name = my-web-server
   → IAM role: AmazonSSMManagedInstanceCore attached (must for SSM)
   → Attach IAM role with AmazonSSMManagedInstanceCore policy (must for SSM)

	Verify SSM Agent is working 
	aws ssm describe-instance-information --filters "Key=tag:Name,Values=my-web-server" --region ap-northeast-1
	# If not online → check console, reboot instance, or add user data to start/enable amazon-ssm-agent
	#!/bin/bash
	# Install / ensure SSM Agent (usually already there)
	sudo systemctl enable amazon-ssm-agent
	sudo systemctl start amazon-ssm-agent
	sudo systemctl status amazon-ssm-agent

	mkdir /tmp/ssm
	cd /tmp/ssm
	wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
	sudo rpm --install amazon-ssm-agent.rpm
	sudo systemctl enable amazon-ssm-agent
	sudo systemctl start amazon-ssm-agent

   cd "C:\kunal\cloudthat training devops\scope\codedeploy"

   # Install CodeDeploy agent via SSM (tag-based – safe)
   aws ssm send-command --region ap-northeast-1 --document-name "AWS-RunShellScript" --targets "Key=tag:Name,Values=my-web-server" --parameters "commands=yum install -y ruby wget,cd /tmp,wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install,chmod +x install,./install auto,systemctl enable codedeploy-agent || true,systemctl start codedeploy-agent || true" --comment "Install CodeDeploy Agent" --output text

   # Check command status (get CommandId from above output)
   aws ssm list-command-invocations --command-id  d14e72a8-a155-458c-abd3-a9fa73bd3e29 --details --region ap-northeast-1

   # Prepare artifact
   aws s3 mb s3://web-cd-artifacts-ks-demo-tokyo --region ap-northeast-1   # unique name
   
   # Make application web from console
   # make deployment grp - webdg
   aws deploy push --application-name web --source web --s3-location s3://web-cd-artifacts-ks-demo-tokyo/web.zip --region ap-northeast-1

   # Deploy
	C:\kunal\cloudthat training devops\scope\codedeploy>aws deploy push --application-name web --source web --s3-location s3://web-cd-artifacts-ks-demo-tokyo/web.zip --region ap-northeast-1
		To deploy with this revision, run:
		aws deploy create-deployment --application-name web --s3-location bucket=web-cd-artifacts-ks-demo-tokyo,key=web.zip,bundleType=zip,eTag=dfa6d5eafe96f625fea26e3dc1911f32 --deployment-group-name <deployment-group-name> --deployment-config-name <deployment-config-name> --description <description>

		C:\kunal\cloudthat training devops\scope\codedeploy>aws deploy create-deployment --application-name web --s3-location bucket=web-cd-artifacts-ks-demo-tokyo,key=web.zip,bundleType=zip,eTag=dfa6d5eafe96f625fea26e3dc1911f32 --deployment-group-name webdg
		{
			"deploymentId": "d-VR9M0X3V4"
		}

   # Monitor in console or CLI
   aws deploy get-deployment --deployment-id d-VR9M0X3V4 --region ap-northeast-1
   aws deploy get-deployment --deployment-id d-VR9M0X3V4 --region ap-northeast-1 --query "deploymentInfo.{status:status,deploymentConfigName:deploymentConfigName,deploymentGroupName:deploymentGroupName,createTime:createTime}"
   
   http://54.248.101.211/index.html
   → "If agent fails → check SSM role, VPC endpoints, yum repos, iam related roles in deloyment events. For prod use blue/green."
```

### 4. CodePipeline (Source → Build → Deploy)
```text
UseCase: When deployment needed pipeline	
   # Make a code pipeline 
		Pipeline Name : - webpipeline
		Source Stage: S3 (web-codepipeline-artifacts-ks-411045)
		Deploy Stage: application-name my-web-server-app , deployment-group-name my-web-server-appdg
			
   cd "C:\kunal\cloudthat training devops\scope\codepipeline"
			

   aws s3 mb s3://web-codepipeline-artifacts-ks-411045 --region ap-northeast-1
   aws s3 cp web.zip s3://web-codepipeline-artifacts-ks-411045/web.zip --region ap-northeast-1


   → Open AWS Console → CodePipeline → show pipeline stages (Source: S3/CodeCommit, Build: CodeBuild, Deploy: CodeDeploy/ECS)

   → "This automates from commit → production. Can add approval stage, tests, notifications."
	http://54.248.101.211/index.html
```
### 5. SAM (Serverless)
```text
UseCase: When deployment has all serverless services
   cd "C:\kunal\cloudthat training devops\scope\aws sam\sam-app"

   # Edit template.yaml or code → git add/commit/push
   git status
   git commit -m "Update Lambda function - added feature X"
   git push

   → Show pipeline trigger in CodePipeline or SAM pipeline
   → Check CloudFormation / Lambda output for endpoint URL
	https://0qlbdmiz76.execute-api.ap-northeast-1.amazonaws.com/Prod/hello/
   → "SAM extends CloudFormation for serverless – easy local testing with sam local invoke."
```

### 6. ECS / Fargate Pipeline
```text
UseCase: When Fargate Containers are needed
   For Set up - https://github.com/kunalshrivastavapune25/aws/blob/main/ecs/setup.md
   
   
   
   cd "C:\kunal\cloudthat training devops\scope\ecs ecr fargate\MY_WEB_APP_DEMO"
	aws ecs create-service --service-name MyApp-Web-service --cli-input-json file://create-service.json
	also in alb the weight of green should be 0
   # Change Dockerfile or app code → commit & push
   git add .
   git commit -m "Update container image version"
   git push

   → Show ECR repo (image pushed), CodePipeline stages, ECS service update
   → ALB DNS: http://my-webapp-alb-2030003135.ap-northeast-1.elb.amazonaws.com/ (or update to ap-northeast-1)

   → "Blue/green on ECS via CodeDeploy or rolling updates. Monitoring via CloudWatch."

```

### 7. eks
```text

1- install custer and nodes in the vpc we made
# Create Cluster (Section-01-02)
eksctl create cluster --name=eksdemo1 --region=ap-northeast-1 --vpc-public-subnets="subnet-0d52941dc72a58756,subnet-02ffca61faa248587" --vpc-private-subnets="subnet-0cf5e41795306d7d9,subnet-0b2a07b365cbb8ccd" --version="1.29" --without-nodegroup

# Get List of clusters (Section-01-02)
eksctl get cluster   

# Template (Section-01-02)
eksctl utils associate-iam-oidc-provider \
    --region region-code \
    --cluster <cluter-name> \
    --approve

# Replace with region & cluster name (Section-01-02)
eksctl utils associate-iam-oidc-provider --region ap-northeast-1 --cluster eksdemo1 --approve

# Create EKS NodeGroup in VPC Private Subnets (Section-07-01)
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=ap-northeast-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking       



eksctl create nodegroup --cluster=eksdemo1 --region=ap-northeast-1 --name=eksdemo1-ng-private1 --node-type=t3.medium --nodes-min=2 --nodes-max=4 --node-volume-size=20 --ssh-access --ssh-public-key=eks_demo --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access --node-private-networking

eksctl delete nodegroup --cluster=eksdemo1 --region=ap-northeast-1 --name=eksdemo1-ng-private1

# Verfy EKS Cluster
eksctl get cluster

# Verify EKS Node Groups
eksctl get nodegroup --cluster=eksdemo1

# Verify if any IAM Service Accounts present in EKS Cluster
eksctl get iamserviceaccount --cluster=eksdemo1
Observation:
1. No k8s Service accounts as of now. 

# Configure kubeconfig for kubectl
eksctl get cluster # TO GET CLUSTER NAME
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
aws eks --region ap-northeast-1 update-kubeconfig --name eksdemo1

# Verify EKS Nodes in EKS Cluster using kubectl
kubectl get nodes

# Verify using AWS Management Console
1. EKS EC2 Nodes (Verify Subnet in Networking Tab)
2. EKS Cluster

create policy
C:\kunal\udemy_eks\git\aws-eks-kubernetes-masterclass\08-NEW-ELB-Application-LoadBalancers\08-01-Load-Balancer-Controller-Install>aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy2 --policy-document file://iam_policy_latest.json
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy2",
        "PolicyId": "ANPAS6UTTHCBS643ESUJO",
        "Arn": "arn:aws:iam::203246745731:policy/AWSLoadBalancerControllerIAMPolicy2",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-02-02T10:43:03+00:00",
        "UpdateDate": "2026-02-02T10:43:03+00:00"
    }
}

# Replaced name, cluster and policy arn (Policy arn we took note in step-02)
eksctl create iamserviceaccount --cluster=eksdemo1 --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::203246745731:policy/AWSLoadBalancerControllerIAMPolicy2 --override-existing-serviceaccounts --approve

kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
Obseravation:
1. We should see a new Service account created. 

# Describe Service Account aws-load-balancer-controller
kubectl describe sa aws-load-balancer-controller -n kube-system


helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eksdemo1 --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=ap-northeast-1 --set vpcId=vpc-0eaad2311d96249ff --set image.repository=public.ecr.aws/eks/aws-load-balancer-controller





2- install load balancer controller


3- install the manifest




4- check the app


5- make the pipeline



6- practice vpas, hpas, nas



7- deploy a microservice 




8- see the logs in xray and cloud watch
http://externaldns-ingress-2083053814.ap-northeast-1.elb.amazonaws.com/app1/
```



