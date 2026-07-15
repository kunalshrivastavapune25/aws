Absolutely. Given your environment (AWS Organization with a dedicated **Security Infra** account as the **Firewall Manager delegated administrator**), I would recommend treating this as an **enterprise migration project**, not just a CloudFormation development task.

Below is an end-to-end runbook that can be used by your team from discovery through production deployment.

---

# Runbook

# Migration of Existing AWS Firewall Manager & AWS WAF to CloudFormation IaC

## Objective

Migrate the existing manually created AWS Firewall Manager and AWS WAF configuration into CloudFormation without impacting production workloads.

---

# Phase 1 – Discovery

## Goal

Understand everything that currently exists.

### Deliverables

* Inventory of FMS Policies
* Inventory of WAF Rule Groups
* Inventory of Web ACLs
* Logging configuration
* Resource Associations
* AWS Organizations scope
* Existing Tags

---

## Step 1

Identify

* Delegated Admin Account
* AWS Region(s)
* Organization ID
* Firewall Manager Administrator

Document them.

---

## Step 2

Export Firewall Manager Policies

Collect

* Policy Name
* Policy ID
* Policy Type
* Resource Type
* Include Map
* Exclude Map
* OU IDs
* Account IDs
* Remediation Enabled
* Delete Unused Resources
* Resource Tags
* SecurityServicePolicyData

---

## Step 3

Export WAF Resources

Collect

* Web ACLs
* Rule Groups
* IP Sets
* Regex Pattern Sets
* Managed Rule Groups
* Custom Rules
* Rate Based Rules
* Labels
* Logging

---

## Step 4

Export Resource Associations

Document

ALB

CloudFront

API Gateway

AppSync

Verified Access

Any protected resources

---

## Step 5

Export Logging

Identify

Firehose

S3

CloudWatch

Log destinations

---

## Step 6

Take Backup

Store all exported JSON

Example

```
Discovery/

    FirewallManager/

    RuleGroups/

    WebACL/

    IPSets/

    Logging/

    Policies/

```

Commit to Git.

---

# Phase 2 – Documentation

Create an LLD.

Document

Architecture

Regions

Delegated Admin

Security Policies

Rule Groups

Priority

Actions

Default Action

Scope

Protected Resources

Logging

Exceptions

Tags

Dependencies

---

# Phase 3 – CloudFormation Design

Create separate templates.

Example

```
CloudFormation/

    RuleGroups/

    WebACL/

    IPSets/

    Logging/

    FirewallManager/

    StackSets/

```

---

Recommended stacks

Stack 1

IP Sets

↓

Stack 2

Regex Pattern Sets

↓

Stack 3

Rule Groups

↓

Stack 4

Web ACL

↓

Stack 5

Logging

↓

Stack 6

Firewall Manager Policy

---

# Phase 4 – Convert Existing Resources

Recreate every resource in YAML.

Verify

Priority

VisibilityConfig

Metric Names

Managed Rules

Custom Rules

Override Actions

Labels

Scope

Capacity

---

# Phase 5 – Validation

Deploy into

Dev OU

or

Sandbox Account

Validate

Template

Deployment

No drift

Correct rule order

Expected blocking

Expected allowing

CloudWatch metrics

Logging

---

# Phase 6 – Production Readiness

Checklist

✓ Templates validated

✓ Security review completed

✓ LLD approved

✓ Rollback plan approved

✓ Maintenance window scheduled

✓ AWS Support informed (optional)

---

# Phase 7 – Production Deployment

Deploy in order

IP Sets

↓

Regex

↓

Rule Groups

↓

Web ACL

↓

Logging

↓

Firewall Manager Policy

Do not delete the existing FMS policy until the new one has been fully validated.

---

# Phase 8 – Post Deployment Validation

Verify

FMS Policy Status

Compliance

Protected Resources

Rule Matches

Blocked Requests

Allowed Requests

Sample traffic

CloudWatch Metrics

Logging

AWS Config Compliance

---

# Phase 9 – Drift Detection

Enable

CloudFormation Drift Detection

Run regularly.

Document any manual changes.

---

# Phase 10 – Operational Handover

Update

Runbooks

LLD

HLD

Repository

Deployment Guide

Support Guide

Rollback Guide

---

# Rollback Plan

If deployment fails

1. Stop deployment.
2. Delete newly created CloudFormation stacks (if they created new resources).
3. Leave the existing Firewall Manager policies active.
4. Restore any modified settings from the JSON backups taken during discovery.
5. Validate application traffic and Firewall Manager compliance before closing the change.

---

# Repository Structure

```
firewallmanager-iac/

├── templates/
│   ├── ipsets/
│   ├── regexpatternsets/
│   ├── rulegroups/
│   ├── webacls/
│   ├── logging/
│   └── fms-policy/
│
├── parameters/
│
├── scripts/
│   ├── export-fms.py
│   ├── export-waf.py
│   ├── validate.py
│   └── compare.py
│
├── discovery/
│
├── docs/
│   ├── HLD.md
│   ├── LLD.md
│   ├── DeploymentGuide.md
│   └── Rollback.md
│
└── pipeline/
```

# Important Consideration: Existing Resources vs. CloudFormation

The biggest challenge is that **CloudFormation generally cannot adopt (import) all existing AWS Firewall Manager and AWS WAF resources**. For some resource types, you may need to recreate them through CloudFormation rather than import them into an existing stack. Before making production changes, verify the import support for each resource type you use and test the approach in a non-production environment. This decision—**import where supported vs. recreate**—should be finalized early in the project because it affects the migration strategy, downtime risk, and rollback plan.

This runbook follows a low-risk approach: discover and document first, build and validate the IaC in a non-production environment, then perform a controlled production migration with clear rollback steps.


Certainly. Since your environment is already in production, the **Discovery Phase** is the most important. Below are the **AWS Console (UI) steps** and **AWS CLI commands** for each activity. The idea is to extract everything without making any changes.

> **Prerequisites**
>
> * Log in to the **Security Infra** account (the Firewall Manager delegated administrator).
> * Use the correct region where Firewall Manager and WAF are configured (for example, `us-east-1` for CloudFront or your regional deployment).
> * Configure AWS CLI:
>
>   ```bash
>   aws configure
>   aws sts get-caller-identity
>   ```

---

# Phase 1 – Discovery

## Step 1: Verify Firewall Manager Delegated Administrator

### UI

1. Sign in to the **Management Account**.
2. Open **AWS Organizations**.
3. Go to **Services** → **Firewall Manager**.
4. Verify the delegated administrator account.

Or:

1. Sign in to the **Security Infra** account.
2. Open **AWS Firewall Manager**.
3. Ensure the dashboard loads successfully (confirming delegated admin access).

### CLI

```bash
aws organizations list-delegated-administrators \
    --service-principal fms.amazonaws.com
```

---

# Step 2: List Firewall Manager Policies

### UI

1. Open **AWS Firewall Manager**.
2. Select **Security policies**.
3. Note:

   * Policy Name
   * Policy Type
   * Resource Type
   * Remediation status
   * Included/Excluded OUs
   * Resource Tags

### CLI

```bash
aws fms list-policies
```

Save output:

```bash
aws fms list-policies > fms-policies.json
```

---

# Step 3: Export Each Firewall Manager Policy

First list policy IDs:

```bash
aws fms list-policies
```

Then:

```bash
aws fms get-policy \
    --policy-id <PolicyId>
```

Save:

```bash
aws fms get-policy \
    --policy-id <PolicyId> \
    > policy-<PolicyId>.json
```

Repeat for every policy.

---

# Step 4: List Web ACLs

### UI

AWS Console

→ WAF & Shield

→ Web ACLs

Record

* Name
* ARN
* Scope
* Default Action

### CLI

Regional:

```bash
aws wafv2 list-web-acls \
    --scope REGIONAL
```

CloudFront:

```bash
aws wafv2 list-web-acls \
    --scope CLOUDFRONT \
    --region us-east-1
```

---

# Step 5: Export Web ACL

```bash
aws wafv2 get-web-acl \
    --scope REGIONAL \
    --id <ID> \
    --name <Name>
```

---

# Step 6: List Rule Groups

### UI

WAF

↓

Rule Groups

↓

Custom Rule Groups

### CLI

Regional

```bash
aws wafv2 list-rule-groups \
    --scope REGIONAL
```

CloudFront

```bash
aws wafv2 list-rule-groups \
    --scope CLOUDFRONT \
    --region us-east-1
```

---

# Step 7: Export Rule Group

```bash
aws wafv2 get-rule-group \
    --scope REGIONAL \
    --id <ID> \
    --name <Name>
```

Save

```bash
aws wafv2 get-rule-group \
    --scope REGIONAL \
    --id <ID> \
    --name <Name> \
    > rulegroup.json
```

---

# Step 8: List IP Sets

### UI

WAF

↓

IP Sets

### CLI

```bash
aws wafv2 list-ip-sets \
    --scope REGIONAL
```

---

# Step 9: Export IP Set

```bash
aws wafv2 get-ip-set \
    --scope REGIONAL \
    --id <ID> \
    --name <Name>
```

---

# Step 10: List Regex Pattern Sets

### UI

WAF

↓

Regex Pattern Sets

### CLI

```bash
aws wafv2 list-regex-pattern-sets \
    --scope REGIONAL
```

---

# Step 11: Export Regex Pattern Set

```bash
aws wafv2 get-regex-pattern-set \
    --scope REGIONAL \
    --id <ID> \
    --name <Name>
```

---

# Step 12: Logging Configuration

### UI

WAF

↓

Web ACL

↓

Logging and Metrics

Record

* Firehose
* S3
* CloudWatch

### CLI

```bash
aws wafv2 get-logging-configuration \
    --resource-arn <WebACLArn>
```

---

# Step 13: Resource Associations

### UI

Inside each Web ACL

↓

Associated AWS Resources

Document

* ALBs
* API Gateway
* AppSync
* CloudFront
* Cognito
* Verified Access

### CLI

```bash
aws wafv2 list-resources-for-web-acl \
    --web-acl-arn <ARN> \
    --resource-type APPLICATION_LOAD_BALANCER
```

Repeat for other supported resource types.

---

# Step 14: Managed Rule Groups

Inside Web ACL

Record

* Vendor
* Name
* Version
* Override Actions

These details are included in the `get-web-acl` output.

---

# Step 15: AWS Organizations Information

```bash
aws organizations list-roots
```

```bash
aws organizations list-organizational-units-for-parent \
    --parent-id r-xxxx
```

```bash
aws organizations list-accounts
```

---

# Step 16: Export Tags

For Web ACL

```bash
aws wafv2 list-tags-for-resource \
    --resource-arn <ARN>
```

For Firewall Manager Policy

```bash
aws fms list-tags-for-resource \
    --resource-arn <PolicyArn>
```

---

# Step 17: Verify Firewall Manager Compliance

### UI

Firewall Manager

↓

Compliance

↓

Check all resources

### CLI

```bash
aws fms get-compliance-detail \
    --policy-id <PolicyId> \
    --member-account <AccountId>
```

---

# Suggested Discovery Folder Structure

Store all outputs in a structured folder:

```text
Discovery/
├── FirewallManager/
│   ├── list-policies.json
│   ├── policy-1.json
│   └── policy-2.json
├── WAF/
│   ├── web-acls/
│   ├── rule-groups/
│   ├── ip-sets/
│   ├── regex-pattern-sets/
│   └── logging/
├── Organizations/
│   ├── accounts.json
│   ├── ous.json
│   └── delegated-admin.json
└── Tags/
```

### Next Step

Once the discovery data is collected, the next phase is to **map each exported JSON object to its equivalent CloudFormation resource** (`AWS::WAFv2::RuleGroup`, `AWS::WAFv2::WebACL`, `AWS::FMS::Policy`, etc.), identify any properties that CloudFormation cannot manage, and then build modular CloudFormation templates in the correct dependency order. This is the phase where the IaC templates are authored and validated before any production deployment.




You're right to question that. There are actually **two different AWS services** that people often mix up:

1. **AWS WAF + AWS Firewall Manager**
2. **AWS Network Firewall + AWS Firewall Manager**

From what you've just described ("go inside the member account → VPC → selected VPC → firewall-related stuff"), it sounds like you're actually working with **AWS Network Firewall**, not just AWS WAF.

## If you're using AWS Network Firewall

The flow is different:

```text
Firewall Manager (Security Infra Account)
        ↓
Security Policy
        ↓
Distributed to Member Accounts
        ↓
VPC
        ↓
AWS Network Firewall
        ↓
Firewall Policy
        ↓
Stateless Rule Groups
Stateful Rule Groups
```

### In the Security Infra (Delegated Admin) account

1. Go to **AWS Firewall Manager**.
2. Open **Security Policies**.
3. Select your policy.
4. Check:

   * Policy type = **AWS Network Firewall** (instead of WAF)
   * Included OUs/Accounts
   * Resource type = VPC
   * Auto-remediation
   * Firewall policy being applied

### In a Member Account

Go to:

**VPC Console → Network Firewall**

You'll typically see:

* Firewalls
* Firewall policies
* Rule groups
* TLS inspection configurations (if used)

Select a **Firewall**.

From there you can inspect:

* Firewall name
* Associated VPC
* Subnets where firewall endpoints are deployed
* Firewall policy attached

Then open the **Firewall Policy** to see:

* Stateless rule groups
* Stateful rule groups
* Default actions
* Engine options

---

## If you're actually using AWS WAF

Then the VPC console is **not** where you'd look. WAF protects Layer 7 resources such as ALBs, API Gateway, and CloudFront, and is managed through the WAF console.

---

## I have one question to confirm your environment

In your Firewall Manager **Security Policies**, what is the **Policy Type**?

Is it:

* **AWS WAF**
* **AWS Network Firewall**
* Or do you have **both**?

This distinction is important because the UI navigation, resources, and CloudFormation templates are completely different for WAF and Network Firewall.



Yes. Below is a complete end-to-end flowchart that covers the concepts we've discussed. I've separated the **management plane** (where you configure policies) from the **runtime plane** (where traffic is inspected).

# End-to-End AWS Firewall Manager Architecture

```text
                           AWS ORGANIZATION
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
            Management Account             Security Infra Account
             (Organization)             (Delegated Administrator)
                                               │
                                               │
                                      AWS Firewall Manager
                                               │
                           ┌───────────────────┴───────────────────┐
                           │                                       │
                    Security Policy 1                      Security Policy 2
                 (Production ALBs)                    (CloudFront / APIs)
                           │
                           │
          Defines everything Firewall Manager should enforce
                           │
          ┌────────────────┼─────────────────────────────────────┐
          │                │                                     │
     Target OUs      Target Accounts                  Resource Type
      (Prod OU)      (Specific Accounts)         ALB / API / CF / VPC
          │                │                                     │
          └────────────────┴─────────────────────────────────────┘
                           │
                    References
                           │
                    AWS WAF Web ACL
                           │
        ┌──────────────────┼────────────────────┐
        │                  │                    │
 AWS Managed Rules   Custom Rule Group A   Custom Rule Group B
        │                  │                    │
        │          ┌───────┼────────┐           │
        │          │       │        │           │
        │      SQLi Rule  XSS Rule  Rate Limit  Geo Block Rule
        │
        └─────────────────────────────────────────────────────────

                    Firewall Manager distributes
                           │
         ┌─────────────────┼──────────────────┐
         │                 │                  │
      Member Account 1  Member Account 2  Member Account 3
         │                 │                  │
         │                 │                  │
      Application      Application       Application
         │                 │                  │
      ALB/API GW       ALB/API GW        CloudFront
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │
                     Web ACL Attached
                           │
                    Incoming HTTP Request
                           │
                  Rule Evaluation Begins
                           │
         Rule 1 → Rule 2 → Rule 3 → Managed Rules
                           │
               Match? ─────────────── No Match?
                  │                        │
              BLOCK                     ALLOW
                  │                        │
                  └────────────┬───────────┘
                               │
                     Send Logs & Metrics
                               │
                     CloudWatch / Firehose / S3
```

---

# CloudFormation Dependency Flow

```text
IP Sets
    │
    ▼
Regex Pattern Sets
    │
    ▼
Custom Rule Groups
    │
    ▼
Web ACL
    │
    ▼
Logging Configuration
    │
    ▼
Firewall Manager Policy
    │
    ▼
StackSet (Optional)
```

---

# Discovery Flow (What You'll Do)

```text
Login to Security Infra Account
          │
          ▼
Firewall Manager
          │
          ▼
Security Policies
          │
          ▼
Select Policy
          │
          ├── Note Target Accounts
          ├── Note OUs
          ├── Note Resource Type
          ├── Note Auto Remediation
          └── Identify Web ACL
                    │
                    ▼
             AWS WAF Console
                    │
                    ▼
                Web ACL
                    │
          ┌─────────┼───────────┐
          │         │           │
     Rule Groups  Logging  Associated Resources
          │
          ▼
   Individual Rules
```

---

# Verification Flow in a Member Account

```text
Member Account
      │
      ▼
Protected Resource
(ALB / API Gateway / CloudFront)
      │
      ▼
Check Web ACL Association
      │
      ▼
Verify Rules Applied
      │
      ▼
Generate Test Traffic
      │
      ▼
Check Logs
      │
      ▼
Firewall Manager Compliance
```

---

# Migration Flow

```text
Existing Manual Environment
           │
           ▼
Inventory & Discovery
           │
           ▼
Export Configuration (CLI)
           │
           ▼
Prepare LLD
           │
           ▼
Create CloudFormation Templates
           │
           ▼
Deploy to Test Account / OU
           │
           ▼
Validate Functionality
           │
           ▼
Create Change Set
           │
           ▼
Production Deployment
           │
           ▼
Verify Compliance
           │
           ▼
Enable Drift Detection
           │
           ▼
CloudFormation becomes the Source of Truth
```

## One clarification

The flowchart above is for **AWS Firewall Manager managing AWS WAF**. Earlier you mentioned navigating through **VPC** in the member account, which suggests your environment may also include **AWS Network Firewall**.

If your organization uses **both AWS WAF and AWS Network Firewall** (which is common in enterprise environments), I can create a single architecture diagram that shows how Firewall Manager centrally manages **both services** and clearly distinguishes which resources belong to WAF versus Network Firewall. That would be the most complete reference for your project.
