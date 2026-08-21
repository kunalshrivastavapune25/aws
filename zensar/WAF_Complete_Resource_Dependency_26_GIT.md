# WAF Logs - Complete Resource Dependency (26 Resources)

**CRITICAL: CWLtoFirehoseCrossAccountRole (#8) was MISSING - Now Identified!**

---

## Quick Summary

| Metric | Count |
|--------|-------|
| Total Resources | 26 |
| S3 Buckets | 1 |
| KMS Keys | 1 |
| IAM Roles | 6 |
| IAM Policies | 9 |
| Firehose Streams | 1 |
| Lambda Functions | 1 |
| CWL Destinations | 1 |
| Subscription Filters | 6 |

---

## Resource Table - Log Archive Account (816069122659)

| # | Name | Type | ARN | Prerequisites | Dependents |
|---|------|------|-----|---|---|
| 1 | arch-prd-waflogs | S3 | `arn:aws:s3:::arch-prd-waflogs` | NONE | #3,#4,#5,#9 |
| 2 | c80617f9-c896-... | KMS | `arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56` | NONE | #3,#4,#5,#7,#11 |
| 3 | KMS Key Policy | Policy | Attached to #2 | #2+All Roles | - |
| 4 | S3 Bucket Policy | Policy | Attached to #1 | #1+All Roles+#2 | - |

---

## Resource Table - Security Infra Account (119810927880)

| # | Name | Type | ARN | Prerequisites | Dependents |
|---|------|------|-----|---|---|
| 5 | CloudWatchLogsToS3 | Firehose | `arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3` | #1,#2,#6 | #7,#12,#9,#14-25 |
| 6 | KinesisFirehose Role | IAM Role | `arn:aws:iam::119810927880:role/KinesisFirehoseServiceRole-...` | NONE | #5,#7,#12 |
| 7 | FirehoseS3Policy | Policy | Attached to #6 | #6,#1,#2 | - |
| ⭐8 | CWLtoFirehoseCrossAccountRole | IAM Role | `arn:aws:iam::119810927880:role/CWLtoFirehoseCrossAccountRole` | NONE | #12,#5,#14-25 |
| 9 | FirehoseWAFLogTransformer | Lambda | `arn:aws:lambda:us-east-1:119810927880:function:FirehoseWAFLogTransformer` | #10 | #5 |
| 10 | FirehoseWAFLogTransformerRole | IAM Role | `arn:aws:iam::119810927880:role/FirehoseWAFLogTransformerRole` | NONE | #9,#11 |
| 11 | FirehoseWAFLogTransformerPolicy | Policy | Attached to #10 | #10,#5,#2 | - |
| 12 | CentralFirehoseDestination | CWL Dest | `arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination` | #5,#6,⭐#8 | #13,#14-25 |
| 13 | AllowOrgAccountsSubscription | Policy | Attached to #12 | #12 | - |

**⭐ #8 IS CRITICAL** - Without this role, logs cannot flow from Destination (#12) to Firehose (#5)

---

## Resource Table - Production Account (584477831165)

| # | Name | Type | Prerequisites | Dependents |
|---|------|------|---|---|
| 14 | aws-waf-logs-cf-waf-prod-api | Filter | Log Group, #12, #16 | Routes to #12 |
| 15 | aws-waf-logs-cf-waf-prod-webapp | Filter | Log Group, #12, #16 | Routes to #12 |
| 16 | CWLSubscriptionFilterRole | IAM Role | NONE | #14,#15,#17 |
| 17 | CWLSubscriptionFilterPolicy | Policy | #16, #5 (X-account) | - |

---

## Resource Table - Test Account (762649987860)

| # | Name | Type | Prerequisites | Dependents |
|---|------|------|---|---|
| 18 | aws-waf-logs-cf-waf-tst-api | Filter | Log Group, #12, #20 | Routes to #12 |
| 19 | aws-waf-logs-cf-waf-tst-webapp | Filter | Log Group, #12, #20 | Routes to #12 |
| 20 | CWLSubscriptionFilterRole | IAM Role | NONE | #18,#19,#21 |
| 21 | CWLSubscriptionFilterPolicy | Policy | #20, #5 (X-account) | - |

---

## Resource Table - PPE Account (097357356161)

| # | Name | Type | Prerequisites | Dependents |
|---|------|------|---|---|
| 22 | aws-waf-logs-cf-waf-ppe-api | Filter | Log Group, #12, #24 | Routes to #12 |
| 23 | aws-waf-logs-cf-waf-ppe-webapp | Filter | Log Group, #12, #24 | Routes to #12 |
| 24 | CWLSubscriptionFilterRole | IAM Role | NONE | #22,#23,#25 |
| 25 | CWLSubscriptionFilterPolicy | Policy | #24, #5 (X-account) | - |

---

## Resource Table - Implicit Permission

| # | Name | Type | Prerequisites | Dependents |
|---|------|------|---|---|
| 26 | CWLtoFirehoseCrossAccountPolicy | Policy | ⭐#8, #5 | - |

---

## Creation Order (4 Layers)

### Layer 1: Foundation (Create First)
```
Create in this order:
- #1: S3 Bucket (Log Archive)
- #2: KMS Key (Log Archive)
- #6: Firehose Role (Security Infra)
- ⭐#8: CWLtoFirehoseCrossAccountRole (Security Infra) ← NEW & CRITICAL
- #10: Lambda Role (Security Infra)
- #16: Sub Filter Role (Prod)
- #20: Sub Filter Role (Test)
- #24: Sub Filter Role (PPE)
```

### Layer 2: Policies (Attach to Roles)
```
Create in this order:
- #3: KMS Key Policy (needs all Role ARNs from Layer 1)
- #4: S3 Bucket Policy (needs all Role ARNs from Layer 1)
- #7: Firehose Role Policy
- #11: Lambda Role Policy
- #26: CWLtoFirehoseCrossAccountPolicy ← NEW & CRITICAL
- #17: Prod Sub Filter Policy
- #21: Test Sub Filter Policy
- #25: PPE Sub Filter Policy
```

### Layer 3: Core Services (Deploy)
```
Create in this order:
- #5: Firehose Stream (needs #1 S3 ARN, #2 KMS ARN, #6 Role)
- #9: Lambda Function (needs #10 Role)
- #12: CWL Destination (needs #5 Firehose ARN, #6 Role ARN, ⭐#8 Role ARN)
```

### Layer 4: Routing (Setup Filters)
```
Create in this order:
- #13: Destination Policy
- #14-15: Prod Filters (need #12 Destination ARN, #16 Role)
- #18-19: Test Filters (need #12 Destination ARN, #20 Role)
- #22-23: PPE Filters (need #12 Destination ARN, #24 Role)
```

---

## Critical Dependencies Map

### Firehose Stream (#5) requires:
- S3 Bucket ARN (#1)
- KMS Key ARN (#2)
- Firehose Role (#6)

### CWL Destination (#12) requires:
- Firehose ARN (#5)
- Firehose Role ARN (#6)
- **⭐ CWLtoFirehoseCrossAccountRole ARN (#8)** ← MOST CRITICAL

### KMS Key (#2) is used by:
- S3 Policy (#4)
- Firehose Policy (#7)
- Lambda Policy (#11)
- All encryption operations

### All Subscription Filters (#14-25) require:
- Destination ARN (#12)
- Their respective Role (#16, #20, #24)

---

## Data Flow

```
WAF Log Group (Prod/Test/PPE)
        ↓
Subscription Filter (#14-25)
        ↓
CWL Destination (#12)
    [Uses Role #8 ← CRITICAL]
        ↓
Firehose Stream (#5)
    [Transform via Lambda #9]
        ↓
S3 Bucket (#1)
    [Encrypted with KMS #2]
        ↓
Microsoft Sentinel
```

---

## Missing Resource Identified: ⭐ #8

### CWLtoFirehoseCrossAccountRole

**What:** IAM role enabling CloudWatch Logs to put records into Firehose

**Account:** Security Infra (119810927880)

**ARN:** `arn:aws:iam::119810927880:role/CWLtoFirehoseCrossAccountRole`

**Trust Principal:** `logs.us-east-1.amazonaws.com`

**Permissions:**
- `firehose:PutRecord`
- `firehose:PutRecordBatch`

**Resource:** `arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3`

**Attached To:** CWL Destination (#12)

**Critical For:** Cross-account subscription filters to work

**Without It:** 
```
ERROR: AccessDenied - User is not authorized to perform: firehose:PutRecord
RESULT: Logs do NOT flow to Firehose or S3
```

---

## Cross-Account Role Permissions

### From Prod (584477831165)
- Role #16 needs Firehose ARN #5
- Filters #14-15 need Destination ARN #12

### From Test (762649987860)
- Role #20 needs Firehose ARN #5
- Filters #18-19 need Destination ARN #12

### From PPE (097357356161)
- Role #24 needs Firehose ARN #5
- Filters #22-23 need Destination ARN #12

---

## Account Summary

| Account | # Resources | Types |
|---------|-------------|-------|
| Log Archive (816069122659) | 4 | S3, KMS, 2 Policies |
| Security Infra (119810927880) | 9 | Firehose, 3 Roles, 4 Policies, Destination, Lambda |
| Production (584477831165) | 4 | 2 Filters, Role, Policy |
| Test (762649987860) | 4 | 2 Filters, Role, Policy |
| PPE (097357356161) | 4 | 2 Filters, Role, Policy |
| **TOTAL** | **26** | |

---

## Implementation Checklist

- [ ] **Layer 1:** Create all Roles (#6, ⭐#8, #10, #16, #20, #24) + Base resources (#1, #2)
- [ ] **Layer 2:** Attach all Policies (#3, #4, #7, #11, #26, #17, #21, #25)
- [ ] **Layer 3:** Deploy Services (#5, #9, #12)
- [ ] **Layer 4:** Setup Routing (#13, #14-25)
- [ ] Verify all cross-account permissions
- [ ] Test end-to-end log flow (5 min max latency)
- [ ] Validate S3 files are encrypted with KMS #2
- [ ] Enable Security Hub monitoring

---

## Key Takeaways

✅ **26 Total Resources** (was incorrectly listed as 22)

✅ **CWLtoFirehoseCrossAccountRole (#8)** is the critical missing piece

✅ **MUST create #8 in Layer 1** - Before creating #12 Destination

✅ **Without #8** - Logs will NOT flow from Destination to Firehose

✅ **Complete dependency chain** - All prerequisites documented

✅ **Ready for implementation** - All 4 layers clearly defined

---

**Document Version:** 2.0 (Git-optimized)  
**Last Updated:** [Date]  
**Status:** Ready for Stakeholder Review
