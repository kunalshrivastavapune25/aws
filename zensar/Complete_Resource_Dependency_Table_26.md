# Complete Resource Dependency Table - All 26 Resources

**Including the newly identified CWLtoFirehoseCrossAccountRole**

---

## Full Dependency Matrix

| # | Resource Name | Resource Type | Account | ARN | Prerequisites (Needed to Create This) | Dependents (Resources That Need This ARN) |
|---|---|---|---|---|---|---|
| 1 | arch-prd-waflogs | S3 Bucket | Log Archive (816069122659) | `arn:aws:s3:::arch-prd-waflogs` | NONE (create first) | Firehose (destination), S3 Bucket Policy, Lambda (read/write) |
| 2 | c80617f9-c896-4cfc-a050-5bb37b97be56 | KMS Key | Log Archive (816069122659) | `arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56` | NONE (create first) | KMS Key Policy, S3 Bucket Policy, Firehose (encryption), Lambda (encryption) |
| 3 | (KMS Key Policy) | IAM Policy | Log Archive (816069122659) | Attached to Key `c80617f9...` | KMS Key ARN + All Role ARNs (Firehose, Lambda, CWLtoFirehose) | TERMINAL (policy only) |
| 4 | (S3 Bucket Policy) | IAM Policy | Log Archive (816069122659) | Attached to Bucket `arch-prd-waflogs` | S3 Bucket ARN + All Role ARNs + KMS Key ARN | TERMINAL (policy only) |
| 5 | CloudWatchLogsToS3 | Kinesis Firehose | Security Infra (119810927880) | `arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3` | S3 Bucket ARN, KMS Key ARN, Firehose Role (#6) | Firehose Role Policy (#7), CWL Destination (#12), Lambda transformer (#8), All Subscription Filters (#14-25) |
| 6 | KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1 | IAM Role | Security Infra (119810927880) | `arn:aws:iam::119810927880:role/KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1` | NONE (can create anytime) | Firehose Stream (#5), Firehose Role Policy (#7), CWL Destination (#12), KMS/S3 Policies (#3, #4) |
| 7 | FirehoseS3Policy | IAM Policy | Security Infra (119810927880) | Attached to Role `KinesisFirehoseServiceRole...` | Firehose Role (#6), S3 Bucket ARN, KMS Key ARN | TERMINAL (policy only) |
| ⭐8 | CWLtoFirehoseCrossAccountRole | IAM Role | Security Infra (119810927880) | `arn:aws:iam::119810927880:role/CWLtoFirehoseCrossAccountRole` | NONE (can create anytime) | CWL Destination (#12), Firehose Stream (#5), Cross-account Subscription Filters (#14-25) |
| 9 | FirehoseWAFLogTransformer | Lambda Function | Security Infra (119810927880) | `arn:aws:lambda:us-east-1:119810927880:function:FirehoseWAFLogTransformer` | Lambda Role (#10) | Firehose Stream (#5 - as transformer), CloudWatch Logs (execution logs) |
| 10 | FirehoseWAFLogTransformerRole | IAM Role | Security Infra (119810927880) | `arn:aws:iam::119810927880:role/FirehoseWAFLogTransformerRole` | NONE (can create anytime) | Lambda Function (#9), Lambda Role Policy (#11), KMS/S3 Policies (#3, #4) |
| 11 | FirehoseWAFLogTransformerPolicy | IAM Policy | Security Infra (119810927880) | Attached to Role `FirehoseWAFLogTransformerRole` | Lambda Role (#10), Firehose Stream ARN (#5), KMS Key ARN | TERMINAL (policy only) |
| 12 | CentralFirehoseDestination | CloudWatch Logs Destination | Security Infra (119810927880) | `arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination` | Firehose Stream ARN (#5), Firehose Role ARN (#6), CWLtoFirehose Role ARN (⭐#8) | Destination Policy (#13), All Subscription Filters (#14-25) |
| 13 | AllowOrgAccountsSubscription | CWL Destination Policy | Security Infra (119810927880) | Attached to Destination `CentralFirehoseDestination` | Destination ARN (#12), Organization ID (o-j821cgdqkd) | TERMINAL (policy only) |
| 14 | aws-waf-logs-cf-waf-prod-api (Filter) | Subscription Filter | Prod (584477831165) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-prod-api), Destination ARN (#12), Prod Sub Filter Role (#16) | Routes logs to Destination automatically |
| 15 | aws-waf-logs-cf-waf-prod-webapp (Filter) | Subscription Filter | Prod (584477831165) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-prod-webapp), Destination ARN (#12), Prod Sub Filter Role (#16) | Routes logs to Destination automatically |
| 16 | CWLSubscriptionFilterRole (Prod) | IAM Role | Prod (584477831165) | `arn:aws:iam::584477831165:role/CWLSubscriptionFilterRole` | NONE (can create anytime) | Prod Subscription Filters (#14, #15), Prod Sub Filter Policy (#17) |
| 17 | CWLSubscriptionFilterPolicy (Prod) | IAM Policy | Prod (584477831165) | Attached to Role `CWLSubscriptionFilterRole` | Prod Sub Filter Role (#16), Firehose Stream ARN (#5 - cross-account) | TERMINAL (policy only) |
| 18 | aws-waf-logs-cf-waf-tst-api (Filter) | Subscription Filter | Test (762649987860) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-tst-api), Destination ARN (#12), Test Sub Filter Role (#20) | Routes logs to Destination automatically |
| 19 | aws-waf-logs-cf-waf-tst-webapp (Filter) | Subscription Filter | Test (762649987860) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-tst-webapp), Destination ARN (#12), Test Sub Filter Role (#20) | Routes logs to Destination automatically |
| 20 | CWLSubscriptionFilterRole (Test) | IAM Role | Test (762649987860) | `arn:aws:iam::762649987860:role/CWLSubscriptionFilterRole` | NONE (can create anytime) | Test Subscription Filters (#18, #19), Test Sub Filter Policy (#21) |
| 21 | CWLSubscriptionFilterPolicy (Test) | IAM Policy | Test (762649987860) | Attached to Role `CWLSubscriptionFilterRole` | Test Sub Filter Role (#20), Firehose Stream ARN (#5 - cross-account) | TERMINAL (policy only) |
| 22 | aws-waf-logs-cf-waf-ppe-api (Filter) | Subscription Filter | PPE (097357356161) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-ppe-api), Destination ARN (#12), PPE Sub Filter Role (#24) | Routes logs to Destination automatically |
| 23 | aws-waf-logs-cf-waf-ppe-webapp (Filter) | Subscription Filter | PPE (097357356161) | Filter resource (no direct ARN) | Log Group (aws-waf-logs-cf-waf-ppe-webapp), Destination ARN (#12), PPE Sub Filter Role (#24) | Routes logs to Destination automatically |
| 24 | CWLSubscriptionFilterRole (PPE) | IAM Role | PPE (097357356161) | `arn:aws:iam::097357356161:role/CWLSubscriptionFilterRole` | NONE (can create anytime) | PPE Subscription Filters (#22, #23), PPE Sub Filter Policy (#25) |
| 25 | CWLSubscriptionFilterPolicy (PPE) | IAM Policy | PPE (097357356161) | Attached to Role `CWLSubscriptionFilterRole` | PPE Sub Filter Role (#24), Firehose Stream ARN (#5 - cross-account) | TERMINAL (policy only) |
| 26 | (CWLtoFirehoseCrossAccountPolicy) | Implicit Permission Policy | Security Infra (119810927880) | Attached to Role `CWLtoFirehoseCrossAccountRole` (⭐#8) | CWLtoFirehose Role (#8), Firehose Stream ARN (#5) | TERMINAL (policy only) |

---

## Key Legend

| Symbol | Meaning |
|---|---|
| ⭐ | Newly identified missing resource |
| NONE | No prerequisites - can be created first |
| TERMINAL | Policy only - depends on other resources but nothing depends on it |
| ARN | Amazon Resource Name - the full identifier |

---

## Creation Order (Layer by Layer)

### Layer 1: Foundation (No Prerequisites)
- #1: S3 Bucket
- #2: KMS Key
- #6: Firehose Role
- ⭐#8: CWLtoFirehoseCrossAccountRole ← **NEW!**
- #10: Lambda Role
- #16: Prod Sub Filter Role
- #20: Test Sub Filter Role
- #24: PPE Sub Filter Role

### Layer 2: Policies (Depends on Layer 1)
- #3: KMS Key Policy (needs Key + all Role ARNs)
- #4: S3 Bucket Policy (needs Bucket + all Role ARNs + Key ARN)
- #7: Firehose Role Policy (needs Role + S3 ARN + KMS ARN)
- #11: Lambda Role Policy (needs Role + Firehose ARN + KMS ARN)
- #26: CWLtoFirehoseCrossAccountPolicy ← **NEW!**
- #17: Prod Sub Filter Policy (needs Role + Firehose ARN)
- #21: Test Sub Filter Policy (needs Role + Firehose ARN)
- #25: PPE Sub Filter Policy (needs Role + Firehose ARN)

### Layer 3: Services (Depends on Layers 1-2)
- #5: Firehose Stream (needs S3 ARN + KMS ARN + Role #6)
- #9: Lambda Function (needs Role #10)
- #12: CWL Destination (needs Firehose ARN #5 + Role ARN #6 + CWLtoFirehose Role ARN ⭐#8) ← **KEY: Needs #8!**

### Layer 4: Routing (Depends on Layers 1-3)
- #13: Destination Policy (needs Destination #12)
- #14-#25: Subscription Filters (need Destination ARN #12 + their respective Roles)

---

## Critical Resource Dependencies

### Firehose Stream (#5) is needed by:
- Destination (#12)
- All Subscription Filter Policies (#17, #21, #25)
- All Subscription Filters (#14-25)

### CWL Destination (#12) is needed by:
- All Subscription Filters (#14-25)
- Enables cross-account log forwarding

### CWLtoFirehoseCrossAccountRole (⭐#8) is needed by:
- CWL Destination (#12) - CRITICAL!
- Without it: Logs can't flow from Destination to Firehose

### KMS Key (#2) is needed by:
- S3 Bucket Policy (#4)
- Firehose Role Policy (#7)
- Lambda Role Policy (#11)
- All encryption operations

### S3 Bucket (#1) is needed by:
- Firehose (as destination)
- S3 Bucket Policy (#4)
- All log storage

---

## Cross-Account Dependencies

| From Account | To Account | Resource Needed | ARN Used |
|---|---|---|---|
| Prod (#584477831165) | Security Infra (#119810927880) | Firehose ARN | #5 ARN in Policy #17 |
| Prod (#584477831165) | Security Infra (#119810927880) | Destination ARN | #12 ARN in Filters #14-15 |
| Test (#762649987860) | Security Infra (#119810927880) | Firehose ARN | #5 ARN in Policy #21 |
| Test (#762649987860) | Security Infra (#119810927880) | Destination ARN | #12 ARN in Filters #18-19 |
| PPE (#097357356161) | Security Infra (#119810927880) | Firehose ARN | #5 ARN in Policy #25 |
| PPE (#097357356161) | Security Infra (#119810927880) | Destination ARN | #12 ARN in Filters #22-23 |
| Security Infra (#119810927880) | Log Archive (#816069122659) | S3 Bucket ARN | #1 ARN in Firehose #5 |
| Security Infra (#119810927880) | Log Archive (#816069122659) | KMS Key ARN | #2 ARN in Firehose #5 & Policies |

---

## Data Flow Path

```
Source Account Log Group
    ↓ (Subscription Filter #14-25)
    └─ Requires: Destination ARN (#12) + Sub Filter Role (#16/20/24)

CWL Destination (#12)
    ↓ (Routes via CWLtoFirehoseCrossAccountRole ⭐#8)
    └─ Requires: CWLtoFirehose Role (#8) + Firehose Role (#6)

Kinesis Firehose (#5)
    ├─ Requires: S3 ARN (#1) + KMS ARN (#2) + Role (#6)
    ├─ Transform: Lambda Function (#9) with Role (#10)
    └─ Encrypts: With KMS Key (#2)

S3 Bucket (#1)
    └─ Receives: Encrypted data from Firehose
    └─ Encrypts: At rest with KMS Key (#2)
```

---

## What's New: CWLtoFirehoseCrossAccountRole (⭐#8)

**This was the missing piece!**

| Aspect | Details |
|---|---|
| **Name** | CWLtoFirehoseCrossAccountRole |
| **Purpose** | Allows CloudWatch Logs service to put records into Firehose |
| **Trust Principal** | `logs.us-east-1.amazonaws.com` |
| **Permissions** | `firehose:PutRecord`, `firehose:PutRecordBatch` |
| **Resource** | `arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3` |
| **Attached To** | CWL Destination (#12) |
| **Critical For** | Cross-account subscription filters to work |
| **Without It** | Error: "AccessDenied" on firehose:PutRecord |

---

## Summary Statistics

| Metric | Count |
|---|---|
| **Total Resources** | 26 |
| **S3 Buckets** | 1 |
| **KMS Keys** | 1 |
| **IAM Roles** | 6 (Firehose, CWLtoFirehose ⭐, Lambda, 3× Sub Filter) |
| **IAM Policies** | 9 (KMS, S3, Firehose, CWLtoFirehose ⭐, Lambda, 3× Sub Filter) |
| **Firehose Streams** | 1 |
| **Lambda Functions** | 1 |
| **CWL Destinations** | 1 |
| **Subscription Filters** | 6 |
| **Total Accounts** | 5 (Log Archive, Security Infra, Prod, Test, PPE) |
| **Total Regions** | 2 (us-east-1, eu-west-1) |

---

## Creation Checklist

- [ ] Layer 1: Create Foundation (S3, KMS, all Roles including ⭐#8)
- [ ] Layer 2: Attach Policies (KMS, S3, all Policies including ⭐#26)
- [ ] Layer 3: Deploy Services (Firehose #5, Lambda #9, Destination #12)
- [ ] Layer 4: Setup Routing (Destination Policy #13, Subscription Filters #14-25)

**Total: 26 Resources across 5 accounts, 2 regions**

⭐ **NEW:** CWLtoFirehoseCrossAccountRole (Resource #8) - CRITICAL for pipeline!
