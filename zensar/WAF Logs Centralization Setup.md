# WAF Logs Centralization Setup - Correct Operational Sequence

## Architecture Overview
```
Source Accounts (WAF logs)
    ↓
CloudWatch Logs Subscription Filter
    ↓
CWL Destination (Security Infra Account)
    ↓
Kinesis Firehose (Security Infra Account)
    ↓
Lambda Transformer (optional, for Sentinel format)
    ↓
S3 Bucket (Log Archive Account 816069122659)
    ↓
Microsoft Sentinel
```

---

## PHASE 1: Security Infra Account Setup (119810927880)

### Step 1.1: Create Firehose IAM Role
**Why first?** The role must exist before creating the Firehose stream.

**Run in Security Infra CloudShell:**

```bash
# Create trust policy file
cat > firehose-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "firehose.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1 \
  --assume-role-policy-document file://firehose-trust-policy.json
```

### Step 1.2: Attach Firehose IAM Permissions
**Run in Security Infra CloudShell:**

```bash
# Create permissions policy file
cat > firehose-permissions-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::arch-prd-waflogs",
        "arn:aws:s3:::arch-prd-waflogs/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56"
    }
  ]
}
EOF

# Attach policy to role
aws iam put-role-policy \
  --role-name KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1 \
  --policy-name FirehoseS3Policy \
  --policy-document file://firehose-permissions-policy.json
```

### Step 1.3: Create Firehose Delivery Stream
**Now create the stream with the role:**

**Run in Security Infra CloudShell:**

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name CloudWatchLogsToS3 \
  --delivery-stream-type DirectPut \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::arch-prd-waflogs",
    "RoleARN": "arn:aws:iam::119810927880:role/KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1",
    "Prefix": "waf-logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "waf-logs-errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    "BufferingHints": {
      "SizeInMBs": 64,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "UNCOMPRESSED",
    "EncryptionConfiguration": {
      "KMSEncryptionConfig": {
        "AWSKMSKeyARN": "arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56"
      }
    }
  }' \
  --region us-east-1
```

### Step 1.4: Verify Firehose is ACTIVE
**Run in Security Infra CloudShell:**

```bash
aws firehose describe-delivery-stream \
  --delivery-stream-name CloudWatchLogsToS3 \
  --region us-east-1 \
  --query 'DeliveryStreamDescription.DeliveryStreamStatus'
```

**Expected output:** `"ACTIVE"`

### Step 1.5: Create CWL Destination
**This is the central hub that all source accounts will send logs to:**

**Run in Security Infra CloudShell:**

```bash
aws logs put-destination \
  --destination-name "CentralFirehoseDestination" \
  --target-arn "arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3" \
  --role-arn "arn:aws:iam::119810927880:role/KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1" \
  --region us-east-1
```

### Step 1.6: Apply Destination Policy
**Allows source accounts to send logs to this destination:**

**Run in Security Infra CloudShell:**

```bash
# First, get your Organization ID
aws organizations describe-organization \
  --query 'Organization.Id' \
  --output text

# Apply destination policy (update o-j821cgdqkd with your Org ID if different)
aws logs put-destination-policy \
  --destination-name "CentralFirehoseDestination" \
  --access-policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowOrgAccountsSubscription",
        "Effect": "Allow",
        "Principal": {
          "AWS": "*"
        },
        "Action": "logs:PutSubscriptionFilter",
        "Resource": "arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination",
        "Condition": {
          "StringEquals": {
            "aws:PrincipalOrgID": "o-j821cgdqkd"
          }
        }
      }
    ]
  }' \
  --region us-east-1
```

---

## PHASE 2: Source Account Setup (Per Account)
Repeat this phase for each source account: **584477831165**, **762649987860**, **097357356161**

### Step 2.1: Create CWL Subscription Filter Role
**Run in Source Account CloudShell (replace 584477831165 with actual account):**

```bash
# Create trust policy
cat > cwl-subscription-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "logs.us-east-1.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name CWLSubscriptionFilterRole \
  --assume-role-policy-document file://cwl-subscription-trust.json
```

### Step 2.2: Attach Subscription Filter Permissions
**Run in Source Account CloudShell:**

```bash
# Create permissions policy
cat > cwl-subscription-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ],
      "Resource": "arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3"
    }
  ]
}
EOF

# Attach policy to role
aws iam put-role-policy \
  --role-name CWLSubscriptionFilterRole \
  --policy-name CWLSubscriptionFilterPolicy \
  --policy-document file://cwl-subscription-policy.json
```

### Step 2.3: Create Subscription Filters for Each Log Group
**Run in Source Account CloudShell (update ACCOUNT_ID):**

```bash
ACCOUNT_ID="584477831165"  # Change for each source account
DESTINATION_ARN="arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CWLSubscriptionFilterRole"

# Production Account WAF Logs
aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-prod-api" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1

aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-prod-webapp" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1
```

### Step 2.4: Repeat for Test Account (762649987860)
**Run in Test Account CloudShell:**

```bash
ACCOUNT_ID="762649987860"
DESTINATION_ARN="arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CWLSubscriptionFilterRole"

aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-tst-api" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1

aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-tst-webapp" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1
```

### Step 2.5: Repeat for PPE Account (097357356161)
**Run in PPE Account CloudShell:**

```bash
ACCOUNT_ID="097357356161"
DESTINATION_ARN="arn:aws:logs:us-east-1:119810927880:destination:CentralFirehoseDestination"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CWLSubscriptionFilterRole"

aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-ppe-api" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1

aws logs put-subscription-filter \
  --log-group-name "aws-waf-logs-cf-waf-ppe-webapp" \
  --filter-name "ToSecurityInfraFirehose" \
  --filter-pattern "" \
  --destination-arn "$DESTINATION_ARN" \
  --role-arn "$ROLE_ARN" \
  --region us-east-1
```

---

## PHASE 3: Lambda Transformer Setup (Optional but Recommended for Sentinel)

### Step 3.1: Create Lambda Execution Role
**Run in Security Infra CloudShell:**

```bash
# Create trust policy
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name FirehoseWAFLogTransformerRole \
  --assume-role-policy-document file://lambda-trust-policy.json
```

### Step 3.2: Attach Lambda Permissions
**Run in Security Infra CloudShell:**

```bash
# Create permissions policy
cat > lambda-permissions-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:119810927880:*"
    },
    {
      "Sid": "FirehoseAccess",
      "Effect": "Allow",
      "Action": [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ],
      "Resource": "arn:aws:firehose:us-east-1:119810927880:deliverystream/CloudWatchLogsToS3"
    },
    {
      "Sid": "KMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56"
    }
  ]
}
EOF

# Attach policy
aws iam put-role-policy \
  --role-name FirehoseWAFLogTransformerRole \
  --policy-name FirehoseWAFLogTransformerPolicy \
  --policy-document file://lambda-permissions-policy.json

# Attach basic Lambda execution policy
aws iam attach-role-policy \
  --role-name FirehoseWAFLogTransformerRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

### Step 3.3: Create Lambda Function (Python)
**Create a Lambda function via AWS Console or CLI:**
- **Function name:** `FirehoseWAFLogTransformer`
- **Runtime:** Python 3.12
- **Architecture:** x86_64
- **Timeout:** 3 minutes
- **Memory:** 256 MB

**Use this code:**

```python
import base64
import gzip
import json
import io

def lambda_handler(event, context):
    output_records = []
    
    for record in event['records']:
        # Step 1 - Decode base64
        compressed_data = base64.b64decode(record['data'])
        
        # Step 2 - Decompress gzip
        with gzip.GzipFile(fileobj=io.BytesIO(compressed_data)) as f:
            raw_data = f.read().decode('utf-8')
        
        # Step 3 - Parse CloudWatch Logs wrapper
        cw_logs = json.loads(raw_data)
        
        # Step 4 - Handle CONTROL_MESSAGE (heartbeat - skip)
        if cw_logs.get('messageType') == 'CONTROL_MESSAGE':
            output_records.append({
                'recordId': record['recordId'],
                'result': 'Dropped',
                'data': record['data']
            })
            continue
        
        # Step 5 - Extract and transform each log event
        sentinel_events = []
        
        for log_event in cw_logs.get('logEvents', []):
            try:
                # Parse the stringified WAF JSON message
                waf_log = json.loads(log_event['message'])
                
                # Step 6 - Build Sentinel acceptable flat format
                sentinel_record = {
                    'timestamp': waf_log.get('timestamp'),
                    'formatVersion': waf_log.get('formatVersion'),
                    'webaclId': waf_log.get('webaclId'),
                    'terminatingRuleId': waf_log.get('terminatingRuleId'),
                    'terminatingRuleType': waf_log.get('terminatingRuleType'),
                    'action': waf_log.get('action'),
                    'httpSourceName': waf_log.get('httpSourceName'),
                    'httpSourceId': waf_log.get('httpSourceId'),
                    'owner': cw_logs.get('owner'),
                    'logGroup': cw_logs.get('logGroup'),
                    'httpRequest': {
                        'clientIp': waf_log.get('httpRequest', {}).get('clientIp'),
                        'country': waf_log.get('httpRequest', {}).get('country'),
                        'uri': waf_log.get('httpRequest', {}).get('uri'),
                        'httpMethod': waf_log.get('httpRequest', {}).get('httpMethod'),
                        'host': waf_log.get('httpRequest', {}).get('host'),
                        'args': waf_log.get('httpRequest', {}).get('args'),
                        'httpVersion': waf_log.get('httpRequest', {}).get('httpVersion'),
                        'requestId': waf_log.get('httpRequest', {}).get('requestId')
                    },
                    'ruleGroupList': waf_log.get('ruleGroupList', []),
                    'rateBasedRuleList': waf_log.get('rateBasedRuleList', []),
                    'nonTerminatingMatchingRules': waf_log.get('nonTerminatingMatchingRules', []),
                    'labels': waf_log.get('labels', []),
                    'ja3Fingerprint': waf_log.get('ja3Fingerprint'),
                    'ja4Fingerprint': waf_log.get('ja4Fingerprint'),
                    'requestBodySize': waf_log.get('requestBodySize'),
                    'terminatingRuleMatchDetails': waf_log.get('terminatingRuleMatchDetails', [])
                }
                
                sentinel_events.append(sentinel_record)
                
            except (json.JSONDecodeError, KeyError) as e:
                print(f"Error processing log event: {e}")
                continue
        
        # Step 7 - Convert to JSON lines format
        output_data = '\n'.join([json.dumps(event) for event in sentinel_events])
        if output_data:
            output_data += '\n'
        
        # Step 8 - Encode back to base64 for Firehose
        encoded_data = base64.b64encode(output_data.encode('utf-8')).decode('utf-8')
        
        output_records.append({
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': encoded_data
        })
    
    return {'records': output_records}
```

### Step 3.4: Attach Lambda to Firehose
**Run in Security Infra CloudShell:**

```bash
# Get Lambda function ARN
LAMBDA_ARN=$(aws lambda get-function \
  --function-name FirehoseWAFLogTransformer \
  --region us-east-1 \
  --query 'Configuration.FunctionArn' \
  --output text)

# Update Firehose to use Lambda transformer
aws firehose update-delivery-stream \
  --delivery-stream-name CloudWatchLogsToS3 \
  --delivery-stream-type DirectPut \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::arch-prd-waflogs",
    "RoleARN": "arn:aws:iam::119810927880:role/KinesisFirehoseServiceRole-CloudWatchLog-eu-west-1",
    "ProcessingConfiguration": {
      "Enabled": true,
      "Processors": [
        {
          "Type": "Lambda",
          "Parameters": [
            {
              "ParameterName": "LambdaArn",
              "ParameterValue": "'$LAMBDA_ARN'"
            }
          ]
        }
      ]
    },
    "Prefix": "waf-logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "waf-logs-errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    "BufferingHints": {
      "SizeInMBs": 64,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "UNCOMPRESSED",
    "EncryptionConfiguration": {
      "KMSEncryptionConfig": {
        "AWSKMSKeyARN": "arn:aws:kms:eu-west-1:816069122659:key/c80617f9-c896-4cfc-a050-5bb37b97be56"
      }
    }
  }' \
  --region us-east-1
```

---

## VERIFICATION & TROUBLESHOOTING

### Verify Everything is Connected
**Run in Security Infra CloudShell:**

```bash
# 1. Check Firehose status
echo "=== Firehose Status ==="
aws firehose describe-delivery-stream \
  --delivery-stream-name CloudWatchLogsToS3 \
  --region us-east-1 \
  --query 'DeliveryStreamDescription.DeliveryStreamStatus'

# 2. Check Destination exists
echo "=== CWL Destination ==="
aws logs describe-destinations \
  --region us-east-1 \
  --query 'destinations[?destinationName==`CentralFirehoseDestination`]'

# 3. Check Lambda logs for errors
echo "=== Lambda Transformer Logs ==="
aws logs tail /aws/lambda/FirehoseWAFLogTransformer \
  --follow \
  --region us-east-1

# 4. Verify files in S3
echo "=== S3 Files ==="
aws s3 ls s3://arch-prd-waflogs/waf-logs/ \
  --recursive \
  --region eu-west-1 \
  | tail -20
```

### Common Issues

| Issue | Solution |
|-------|----------|
| **Firehose in CREATING state** | Wait 2-3 minutes for full activation |
| **Logs not flowing** | Verify subscription filter role has correct permissions |
| **Lambda errors** | Check `/aws/lambda/FirehoseWAFLogTransformer` logs |
| **S3 permission denied** | Verify Firehose role has S3 and KMS permissions |
| **Missing files in S3** | Check if WAF is actually logging events |

---

## Summary Checklist

- [ ] Phase 1: Security Infra Account
  - [ ] Create Firehose role (Step 1.1)
  - [ ] Attach Firehose permissions (Step 1.2)
  - [ ] Create Firehose stream (Step 1.3)
  - [ ] Verify ACTIVE status (Step 1.4)
  - [ ] Create CWL Destination (Step 1.5)
  - [ ] Apply Destination policy (Step 1.6)

- [ ] Phase 2: Source Accounts (repeat for each)
  - [ ] Production (584477831165)
    - [ ] Create role (Step 2.1)
    - [ ] Attach permissions (Step 2.2)
    - [ ] Create subscription filters (Step 2.3)
  - [ ] Test (762649987860)
    - [ ] Create role (Step 2.1)
    - [ ] Attach permissions (Step 2.2)
    - [ ] Create subscription filters (Step 2.4)
  - [ ] PPE (097357356161)
    - [ ] Create role (Step 2.1)
    - [ ] Attach permissions (Step 2.2)
    - [ ] Create subscription filters (Step 2.5)

- [ ] Phase 3: Lambda Transformer (Optional)
  - [ ] Create Lambda role (Step 3.1)
  - [ ] Attach Lambda permissions (Step 3.2)
  - [ ] Create Lambda function (Step 3.3)
  - [ ] Attach Lambda to Firehose (Step 3.4)

- [ ] Verification
  - [ ] All statuses ACTIVE
  - [ ] Files appearing in S3
  - [ ] Lambda transformation working
