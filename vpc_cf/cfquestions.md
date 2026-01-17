### Interview Preparation for AWS CloudFormation Topics

Hi Kunal! Below, I've structured interview prep materials based on the topics you listed. For each topic, I've included:

- **5-10 common interview questions** (mix of basic, intermediate, and advanced to cover scenarios).
- **Answers** with clear explanations.
- **Sample code snippets** (YAML from CloudFormation templates) where relevant for better understanding.

These are designed to help you prepare thoroughly—focus on explaining concepts verbally, as interviewers often ask follow-ups like "Why?" or "Give an example." Practice with real templates in your AWS account. Let's dive in!

#### 1. CloudFormation Basics
**Key Notes Recap**: IaC for provisioning; stacks as units; template sections (Resources mandatory); lifecycle with rollback; benefits like repeatability.

**Questions and Answers:**

1. **What is AWS CloudFormation, and how does it relate to Infrastructure as Code (IaC)?**  
   **Answer**: AWS CloudFormation is an IaC service that allows you to model, provision, and manage AWS resources using declarative templates in YAML or JSON. It automates resource creation in a repeatable way, reducing manual errors. For example, you define a VPC and EC2 in a template, and CloudFormation handles orchestration.

2. **What is a CloudFormation stack, and why is it useful?**  
   **Answer**: A stack is a collection of AWS resources created and managed as a single unit from a template. It's useful for grouping related resources (e.g., VPC + EC2 + SG), enabling easy updates/deletions, and ensuring consistency across environments like dev/prod.

3. **What are the mandatory and optional sections in a CloudFormation template?**  
   **Answer**: Mandatory: `Resources` (defines AWS resources). Optional: `Parameters` (inputs), `Outputs` (exposed values), `Mappings` (lookups), `Conditions` (logic), `Metadata` (e.g., for Init), `Transform` (macros), `Description` (human-readable info).

4. **Explain the lifecycle of a CloudFormation stack.**  
   **Answer**: Create (via CLI/API/Console), Update (modify template/parameters), Delete (removes all resources). On failure during create/update, it rolls back by default to prevent partial states—use `--disable-rollback` for debugging.

5. **Why use CloudFormation over manual AWS Console configuration?**  
   **Answer**: Provides automation, version control (templates in Git), repeatability (same template for multiple envs), auditability, and integration with CI/CD. It ensures consistent, idempotent deployments without human error.

6. **What happens if a stack creation fails midway?**  
   **Answer**: CloudFormation rolls back changes automatically, deleting partially created resources. You can disable this with `--disable-rollback` to inspect issues.

7. **How does CloudFormation handle dependencies between resources?**  
   **Answer**: Implicitly via references (e.g., !Ref VPC in Subnet), or explicitly with `DependsOn` attribute. It creates resources in parallel where possible.

**Sample Code Snippet** (Basic Template Structure):
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Basic VPC Stack
Parameters:
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
Outputs:
  VpcId:
    Value: !Ref MyVPC
```

#### 2. CLI Commands (Key Ones)
**Key Notes Recap**: Create/update/delete stacks; status checks; drift detection.

**Questions and Answers:**

1. **How do you create a CloudFormation stack using the AWS CLI?**  
   **Answer**: Use `aws cloudformation create-stack --stack-name <name> --template-body file://template.yaml --parameters ParameterKey=Key,ParameterValue=Value --disable-rollback` (optional for no rollback on failure).

2. **What command checks the status of a stack?**  
   **Answer**: `aws cloudformation describe-stacks --stack-name <name>` for full details, or `--query "Stacks[0].StackStatus"` for just status (e.g., CREATE_COMPLETE).

3. **How do you delete a stack?**  
   **Answer**: `aws cloudformation delete-stack --stack-name <name>`. It deletes all resources unless protected by DeletionPolicy.

4. **What is drift detection, and how do you perform it via CLI?**  
   **Answer**: Drift detects manual changes outside CloudFormation. Run `aws cloudformation detect-stack-drift --stack-name <name>`, then `describe-stack-resource-drifts --stack-name <name> --stack-resource-drift-status-filters MODIFIED DELETED` to view details.

5. **Why might you use `--disable-rollback` when creating a stack?**  
   **Answer**: For debugging failures—prevents automatic cleanup, allowing inspection of partial resources.

6. **How do you list all stacks in a region?**  
   **Answer**: `aws cloudformation list-stacks --query "StackSummaries[?StackStatus!='DELETE_COMPLETE']"`.

**Sample Code Snippet** (N/A - CLI commands are not YAML, but here's a script example for automation):
```bash
#!/bin/bash
aws cloudformation create-stack --stack-name MyStack --template-body file://template.yaml --disable-rollback
aws cloudformation wait stack-create-complete --stack-name MyStack
aws cloudformation describe-stacks --stack-name MyStack --query "Stacks[0].StackStatus"
```

#### 3. Change Sets (Production Must)
**Key Notes Recap**: Preview updates; create/review/execute; avoids outages.

**Questions and Answers:**

1. **What is a Change Set in CloudFormation?**  
   **Answer**: A preview of proposed stack changes (add/modify/replace/delete resources) before applying them, ensuring safe updates without immediate impact.

2. **How do you create and execute a Change Set?**  
   **Answer**: Create: `aws cloudformation create-change-set --stack-name <name> --change-set-name <cs-name> --template-body file://updated.yaml --parameters ...`. Review: `describe-change-set`. Execute: `execute-change-set --stack-name <name> --change-set-name <cs-name>`.

3. **Why are Change Sets important in production?**  
   **Answer**: They prevent outages by allowing review of impacts (e.g., resource replacement downtime). Mandatory for compliance and risk management.

4. **What happens after executing a Change Set?**  
   **Answer**: The stack updates; Change Set is deleted; status becomes UPDATE_COMPLETE if successful.

5. **Can you delete a Change Set?**  
   **Answer**: Yes, `aws cloudformation delete-change-set --stack-name <name> --change-set-name <cs-name>` if not executed.

6. **What if a Change Set shows a resource will be replaced?**  
   **Answer**: It means the resource can't be updated in-place (e.g., EC2 type change)—prepare for potential downtime or data loss.

**Sample Code Snippet** (N/A - CLI-focused, but integrate in template updates).

#### 4. Intrinsic Functions (Most Used)
**Key Notes Recap**: !Ref, !GetAtt, !Sub, etc.; pseudo params.

**Questions and Answers:**

1. **What is the difference between !Ref and !GetAtt?**  
   **Answer**: !Ref returns the resource's physical ID or parameter value; !GetAtt returns a specific attribute (e.g., !GetAtt Instance.PublicIp).

2. **How does !Sub work, and why prefer it over Fn::Sub?**  
   **Answer**: !Sub substitutes variables in strings (e.g., !Sub "http://${Instance.PublicDnsName}"). Short form for readability in YAML.

3. **Explain !ImportValue and its use.**  
   **Answer**: Imports an exported output from another stack (cross-stack ref), e.g., !ImportValue ExportedVpcId. Can't delete exporting stack if imported.

4. **What are pseudo parameters? Give examples.**  
   **Answer**: Built-in refs like AWS::Region, AWS::AccountId, AWS::StackName—available without declaration.

5. **How do you use !Base64 for UserData?**  
   **Answer**: Encodes scripts for EC2 bootstrap, e.g., !Base64 "#!/bin/bash\nyum install httpd".

6. **What is AWS::NoValue used for?**  
   **Answer**: Conditionally removes a property, e.g., in !If.

7. **Give an example of !GetAZs.**  
   **Answer**: !GetAZs "" returns AZ list for the region.

**Sample Code Snippet**:
```yaml
Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref AWS::Region, AMI]
      UserData: !Base64 !Sub |
        #!/bin/bash
        echo "Region: ${AWS::Region}"
Outputs:
  Url: !Sub "http://${Instance.PublicDnsName}"
```

#### 5. Parameters vs Mappings vs Conditions
**Key Notes Recap**: Inputs vs lookups vs logic.

**Questions and Answers:**

1. **What are Parameters, and how are they used?**  
   **Answer**: Dynamic inputs passed at runtime (e.g., InstanceType). Defined with Type, Default, AllowedValues.

2. **How do Mappings differ from Parameters?**  
   **Answer**: Mappings are static key-value lookups (e.g., Region → AMI ID); no runtime input needed.

3. **Explain Conditions and their syntax.**  
   **Answer**: Boolean logic (e.g., !Equals [!Ref Env, prod]) to control resource creation or properties via !If.

4. **When would you use Parameters over Mappings?**  
   **Answer**: For user-configurable values (e.g., env-specific); Mappings for fixed data.

5. **Can Conditions reference Parameters?**  
   **Answer**: Yes, e.g., create NAT only if Param Env=prod.

**Sample Code Snippet**:
```yaml
Parameters:
  Env: {Type: String, AllowedValues: [dev, prod]}
Mappings:
  RegionMap: {us-east-1: {AMI: ami-123}}
Conditions:
  IsProd: !Equals [!Ref Env, prod]
Resources:
  NAT:
    Condition: IsProd
    Type: AWS::EC2::NatGateway
```

#### 6. Outputs & Cross-Stack
**Key Notes Recap**: Expose values; export/import.

**Questions and Answers:**

1. **What are Outputs in a template?**  
   **Answer**: Expose stack values (e.g., EC2 IP) for querying or cross-stack use.

2. **How do you enable cross-stack references?**  
   **Answer**: Add `Export: Name` to Output; import with !ImportValue in another stack.

3. **What limitation exists with exported Outputs?**  
   **Answer**: Can't delete the exporting stack if value is imported elsewhere.

4. **How to query Outputs via CLI?**  
   **Answer**: `aws cloudformation describe-stacks --stack-name <name> --query "Stacks[0].Outputs"`.

**Sample Code Snippet**:
```yaml
Outputs:
  VpcId:
    Value: !Ref MyVPC
    Export: {Name: SharedVpcId}
# In another stack:
Resources:
  Subnet:
    Properties:
      VpcId: !ImportValue SharedVpcId
```

#### 7. Advanced Features
**Key Notes Recap**: Nested stacks, StackSets, policies, DeletionPolicy, custom resources, UserData.

**Questions and Answers:**

1. **What are Nested Stacks?**  
   **Answer**: Stacks within stacks for modularity (e.g., VPC template reused). Use AWS::CloudFormation::Stack with TemplateURL.

2. **Explain StackSets.**  
   **Answer**: Deploys stacks across multiple accounts/regions via AWS Organizations. Service-managed for delegated admin.

3. **What is a Stack Policy?**  
   **Answer**: JSON policy to protect resources from updates (e.g., deny replace on DB).

4. **How does DeletionPolicy work?**  
   **Answer**: On stack delete: Retain (keep resource), Snapshot (backup then delete), Delete (default).

5. **What are Custom Resources?**  
   **Answer**: Lambda/SNS-backed for non-AWS actions (e.g., custom provisioning).

6. **Explain UserData in EC2.**  
   **Answer**: Script run on first boot for bootstrap (e.g., install software). Use with cfn-init for config.

7. **When to use StackSets over single stacks?**  
   **Answer**: For multi-account governance (e.g., security baselines across org).

**Sample Code Snippet** (UserData + DeletionPolicy):
```yaml
Resources:
  Instance:
    Type: AWS::EC2::Instance
    DeletionPolicy: Retain  # Keep on stack delete
    Properties:
      UserData: !Base64 |
        #!/bin/bash
        yum install -y httpd
        systemctl start httpd
```

Practice these with mock interviews—focus on real-world scenarios like "How would you handle a failing update?" Good luck, Kunal! If you need more questions or mocks, let me know.
