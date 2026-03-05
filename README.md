
# Hunters AI SIEM Integration with AWS - Complete Implementation Guide

## Executive Summary

This comprehensive guide provides step-by-step instructions for implementing a centralized security logging infrastructure that integrates with the Hunters AI SIEM platform. The solution automatically collects and forwards logs from CloudWatch, CloudTrail, GuardDuty, and VPC Flow Logs across multiple AWS accounts to a central S3 bucket for security monitoring and threat detection.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Solution Overview](#solution-overview)
3. [Prerequisites](#prerequisites)
4. [Architecture Design](#architecture-design)
5. [Deployment Guide](#deployment-guide)
6. [Testing & Validation](#testing-and-validation)
7. [Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)

---

## Introduction

Organizations operating in multi-account AWS environments face the challenge of centralizing security logs for comprehensive threat detection and compliance monitoring. This solution leverages AWS native services including:

- **Amazon Kinesis Data Streams** - Scalable log ingestion
- **Amazon Kinesis Firehose** - Automated log delivery to S3
- **Amazon S3** - Centralized log storage
- **AWS KMS** - End-to-end encryption with automatic key rotation
- **AWS Lambda** - Automated log group subscription
- **Amazon EventBridge** - Event-driven automation
- **AWS CloudFormation** - Infrastructure as Code deployment

AWS Support provides guidance throughout the implementation process, from initial architecture design to ongoing operational support, ensuring customers can effectively monitor their security posture across their entire AWS environment.

---

## Solution Overview

### Core Components

The solution consists of two main CloudFormation templates:

#### 1. Central Logging Account Infrastructure
**Template**: `HuntersAI_logging_account_stack.yml`

**Deploys**:
- S3 bucket with encryption and lifecycle policies
- Kinesis Data Stream for log ingestion
- Kinesis Firehose for log delivery
- KMS encryption key with automatic rotation
- IAM roles for Hunters AI access
- CloudWatch alarms for monitoring

#### 2. Member Account Log Forwarding
**Template**: `HuntersAI_member_account_stack.yaml`

**Deploys**:
- Lambda functions for automatic log subscription
- EventBridge rules for new log group detection
- IAM roles and policies for cross-account access

### Key Features

✅ **Automated log collection** from CloudWatch, CloudTrail, GuardDuty, and VPC Flow Logs  
✅ **End-to-end encryption** using AWS KMS with automatic key rotation  
✅ **Scalable ingestion** via Kinesis Data Streams and Firehose  
✅ **Secure access** for Hunters AI via IAM roles with external ID validation  
✅ **Automatic subscription** of new CloudWatch Log Groups via EventBridge  
✅ **Cost optimization** through intelligent buffering and compression  
✅ **Enhanced security** with comprehensive bucket policies and encryption enforcement  
✅ **Monitoring and alerting** for Kinesis shard usage and Lambda function health  

---

## Prerequisites

Before implementing this solution, ensure you have:

### Required Access & Permissions
- [ ] AWS Organization with multiple member accounts
- [ ] Administrative access to the central logging account
- [ ] AWS CLI configured with appropriate permissions
- [ ] Permission to create IAM roles and policies

### Required Information
- [ ] Hunters AI account credentials and external ID (required for secure role assumption)
- [ ] List of member account IDs for log forwarding
- [ ] GuardDuty administrator account ID and detector ID
- [ ] AWS Organization ID (format: `o-abc1234567`)
- [ ] Existing CloudTrail trail ARN for monitoring
- [ ] Hunters AI AWS account ID: **685648138888** (for IAM role trust policy)

### AWS Service Prerequisites
- [ ] CloudTrail enabled in the central logging account (for EventBridge triggers)
- [ ] GuardDuty enabled with administrator account configured
- [ ] VPCs identified for flow log enablement

---

## Architecture Design

The solution implements a hub-and-spoke architecture where:

1. **Member accounts** generate logs from various your source documents. **CloudWatch Log Groups** are automatically subscribed to a central destination
3. **Kinesis Data Stream** ingests logs with high throughput
4. **Kinesis Firehose** buffers, compresses, and delivers logs to S3
5. **S3 bucket** stores logs with encryption and lifecycle policies
6. **Hunters AI** accesses logs securely via IAM role assumption

### Data Flow

```
Member Accounts → CloudWatch Logs → Kinesis Stream → Kinesis Firehose → S3 Bucket → Hunters AI
     ↓
CloudTrail → S3 Bucket (Direct)
     ↓
GuardDuty → S3 Bucket (Direct)
     ↓
VPC Flow Logs → S3 Bucket (Direct)
```

### Log Organization

Logs are organized in S3 using prefixes:

- `logs/` - CloudWatch Logs
- `CloudTrail/` - CloudTrail logs
- `guardduty/` - GuardDuty findings
- `VPCFlowLogs/` - VPC Flow Logs

---

## Deployment Guide

### Operations Runbook for SIEM Implementation

This section provides the complete operational procedure for deploying the Hunters AI SIEM integration in AWS environments, specifically tailored for AWS Managed Services (AMS) customers.

---

## Step 1: Deploy Central Logging Infrastructure

### 1.1 Set Deployment Parameters

```bash
# Set deployment parameters
BUCKET_NAME="hunters-siem-logs-$(date +%s)"
LOGGING_ACCOUNT_ID="123456789012"  # Replace with your central logging account ID
MEMBER_ACCOUNTS="111111111111,222222222222,333333333333"  # Comma-separated list
GD_ADMIN_ACCOUNT="444444444444"  # GuardDuty administrator account ID
GD_DETECTOR_ID="abc123def456"  # GuardDuty detector ID
CLOUDTRAIL_ARN="arn:aws:cloudtrail:us-east-1:123456789012:trail/organization-trail"
ORG_ID="o-abc1234567"  # Your AWS Organization ID
REGION="us-east-1"  # Deployment region
EXTERNAL_ID="your-hunters-ai-external-id"  # Obtain from Hunters AI
```

### 1.2 Deploy CloudFormation Stack

```bash
# Deploy central logging stack
aws cloudformation create-stack \
  --stack-name hunters-central-logging \
  --template-body file://HuntersAI_logging_account_stack.yml \
  --parameters \
    ParameterKey=BucketName,ParameterValue=$BUCKET_NAME \
    ParameterKey=ExternalID,ParameterValue=$EXTERNAL_ID \
    ParameterKey=LogsPrefix,ParameterValue=logs/ \
    ParameterKey=MemberAccountIDs,ParameterValue="$MEMBER_ACCOUNTS" \
    ParameterKey=DetectorID,ParameterValue=$GD_DETECTOR_ID \
    ParameterKey=GDPrimaryRegion,ParameterValue=$REGION \
    ParameterKey=GDAdministratorAccount,ParameterValue=$GD_ADMIN_ACCOUNT \
    ParameterKey=OrgId,ParameterValue=$ORG_ID \
    ParameterKey=CreateNewBucket,ParameterValue=true \
    ParameterKey=MonitoredCloudTrailARN,ParameterValue=$CLOUDTRAIL_ARN \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

# Wait for deployment completion
aws cloudformation wait stack-create-complete \
  --stack-name hunters-central-logging \
  --region $REGION

# Verify stack outputs
aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs' \
  --region $REGION
```

### 1.3 Create S3 Folder Structure

After stack deployment, create folders for different log sources:

```bash
# Get the S3 bucket name from stack outputs
S3_BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
  --output text \
  --region $REGION)

# Create folder structure (folders are created automatically on first object write)
# Verify bucket exists
aws s3 ls s3://${S3_BUCKET_NAME}/ --region $REGION
```

> **Note**: The logs from each data source need to be segregated separately using different prefixes:
> - `logs/` - CloudWatch Logs (via Kinesis)
> - `CloudTrail/` - CloudTrail logs
> - `guardduty/` - GuardDuty findings
> - `VPCFlowLogs/` - VPC Flow Logs

---

## Step 2: Update Kinesis Firehose Settings

### 2.1 Configure via AWS Console

Navigate to **Amazon Kinesis** → **Delivery streams** → Select `CWLogsToS3`

**Under "Transform and convert records":**
1. Configure **Decompress source records from Amazon CloudWatch Logs**: **On**

**Under "Destination settings":**
1. **Amazon S3 destination** → **New line delimiter**: **Enable**
2. **Compression, file extension, and encryption**:
   - **File extension format**: `.json`
3. **S3 bucket prefix**: `logs/` (or your required prefix)

### 2.2 Configure via AWS CLI (Alternative)

```bash
# Get Firehose delivery stream name
FIREHOSE_NAME=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`FirehoseDeliveryStreamName`].OutputValue' \
  --output text \
  --region $REGION)

# Note: Firehose configuration updates require recreating the delivery stream
# or using update-destination command with full configuration
# Recommend using AWS Console for these specific settings
```

---

## Step 3: Configure CloudTrail

Ensure that CloudTrail writes logs to the S3 bucket created by the Hunters AI Log Collector Template.

### 3.1 Update CloudTrail Configuration

```bash
# Get KMS Key ARN
KMS_KEY_ARN=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`KMSKeyArn`].OutputValue' \
  --output text \
  --region $REGION)

# Update CloudTrail to use the central S3 bucket
TRAIL_NAME="organization-trail"  # Replace with your trail name

aws cloudtrail update-trail \
  --name $TRAIL_NAME \
  --s3-bucket-name $S3_BUCKET_NAME \
  --s3-key-prefix "CloudTrail/" \
  --kms-key-id $KMS_KEY_ARN \
  --region $REGION

# Verify CloudTrail configuration
aws cloudtrail get-trail-status \
  --name $TRAIL_NAME \
  --region $REGION
```

> **Important**: CloudTrail logs must be written within the `CloudTrail/` prefix.

---

## Step 4: Deploy Member Account Log Forwarding

Deploy the log forwarding infrastructure in each member account.

### 4.1 Get Central Logging Destination ARN

```bash
# Get the CloudWatch Logs destination ARN from central logging stack
CW_DESTINATION_ARN=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`CWLogDestinationArn`].OutputValue' \
  --output text \
  --region $REGION)

echo "CloudWatch Destination ARN: $CW_DESTINATION_ARN"
```

### 4.2 Deploy Stack in Each Member Account

**Run this command in each member account** (requires switching AWS CLI profile or assuming role):

```bash
# Deploy in each member account
aws cloudformation create-stack \
  --stack-name hunters-log-forwarding \
  --template-body file://HuntersAI_member_account_stack.yaml \
  --parameters \
    ParameterKey=CentralLoggingAccountId,ParameterValue=$LOGGING_ACCOUNT_ID \
    ParameterKey=CWLDestinationARN,ParameterValue=$CW_DESTINATION_ARN \
    ParameterKey=SubscribeExistingLogGroups,ParameterValue=yes \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

# Wait for deployment completion
aws cloudformation wait stack-create-complete \
  --stack-name hunters-log-forwarding \
  --region $REGION

# Verify Lambda function was created
aws lambda list-functions \
  --query 'Functions[?starts_with(FunctionName, `hunters-log-forwarding`)].[FunctionName, Runtime, LastModified]' \
  --output table \
  --region $REGION
```

### 4.3 Automate Multi-Account Deployment (Optional)

For deploying across multiple accounts, use AWS Organizations or a script:

```bash
#!/bin/bash
# deploy-member-accounts.sh

MEMBER_ACCOUNT_IDS=("111111111111" "222222222222" "333333333333")

for ACCOUNT_ID in "${MEMBER_ACCOUNT_IDS[@]}"; do
  echo "Deploying to account: $ACCOUNT_ID"
  
  # Assume role in member account
  CREDENTIALS=$(aws sts assume-role \
    --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/OrganizationAccountAccessRole" \
    --role-session-name "HuntersDeployment" \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
    --output text)
  
  export AWS_ACCESS_KEY_ID=$(echo $CREDENTIALS | awk '{print $1}')
  export AWS_SECRET_ACCESS_KEY=$(echo $CREDENTIALS | awk '{print $2}')
  export AWS_SESSION_TOKEN=$(echo $CREDENTIALS | awk '{print $3}')
  
  # Deploy stack
  aws cloudformation create-stack \
    --stack-name hunters-log-forwarding \
    --template-body file://HuntersAI_member_account_stack.yaml \
    --parameters \
      ParameterKey=CentralLoggingAccountId,ParameterValue=$LOGGING_ACCOUNT_ID \
      ParameterKey=CWLDestinationARN,ParameterValue=$CW_DESTINATION_ARN \
      ParameterKey=SubscribeExistingLogGroups,ParameterValue=yes \
    --capabilities CAPABILITY_NAMED_IAM \
    --region $REGION
  
  # Unset credentials
  unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
  
  echo "Deployment initiated for account: $ACCOUNT_ID"
done
```

---

## Step 5: Configure GuardDuty Export

Set up GuardDuty to export findings to the central S3 bucket.

### 5.1 Via AWS Console

1. Navigate to **GuardDuty** in the administrator account
2. Go to **Settings** → **Findings export options** → **S3 bucket**
3. Add the following values:
   - **S3 bucket ARN**: `arn:aws:s3:::siem-logging-bucket-<region>`
   - **KMS Key ARN**: `arn:aws:kms:<region>:<account-id>:key/<key-id>`

### 5.2 Via AWS CLI

```bash
# Get KMS Key ARN and S3 bucket name from central logging stack
KMS_KEY_ARN=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`KMSKeyArn`].OutputValue' \
  --output text \
  --region $REGION)

S3_BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
  --output text \
  --region $REGION)

# Configure GuardDuty export (run in GuardDuty administrator account)
aws guardduty create-publishing-destination \
  --detector-id $GD_DETECTOR_ID \
  --destination-type S3 \
  --destination-properties \
    DestinationArn=arn:aws:s3:::${S3_BUCKET_NAME}/guardduty,KmsKeyArn=${KMS_KEY_ARN} \
  --region $REGION

# Verify publishing destination
aws guardduty list-publishing-destinations \
  --detector-id $GD_DETECTOR_ID \
  --region $REGION
```

---

## Step 6: Configure VPC Flow Logs

Enable VPC Flow Logs for each VPC in all member accounts to forward to the central S3 bucket.

### 6.1 Via AWS Console

For each VPC:
1. Navigate to **VPC** → **Your VPCs**
2. Select VPC → **Actions** → **Create flow log**
3. Configure:
   - **Flow Log Name**: `SIEM-Log-Forwarding`
   - **Destination**: **Send to an Amazon S3 bucket**
   - **S3 bucket ARN**: `arn:aws:s3:::siem-logging-bucket-<region>`
   - **Log record format**: AWS default format
   - **Log file format**: Parquet (recommended) or Text

### 6.2 Via AWS CLI

```bash
# For each VPC in each member account
VPC_ID="vpc-12345678"
ACCOUNT_ID="111111111111"

aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::${S3_BUCKET_NAME}/VPCFlowLogs/${ACCOUNT_ID}/ \
  --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=SIEM-Log-Forwarding}]' \
  --region $REGION

# Verify flow log creation
aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=$VPC_ID" \
  --region $REGION
```

### 6.3 Bulk Enable Flow Logs for All VPCs

```bash
#!/bin/bash
# enable-vpc-flow-logs.sh

# Get all VPCs in current account
VPC_IDS=$(aws ec2 describe-vpcs \
  --query 'Vpcs[*].VpcId' \
  --output text \
  --region $REGION)

for VPC_ID in $VPC_IDS; do
  echo "Enabling flow logs for VPC: $VPC_ID"
  
  aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::${S3_BUCKET_NAME}/VPCFlowLogs/${AWS_ACCOUNT_ID}/ \
    --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=SIEM-Log-Forwarding}]' \
    --region $REGION
  
  echo "Flow log enabled for VPC: $VPC_ID"
done
```

> **Important**: VPC Flow Logs must be written within the `VPCFlowLogs/` prefix.

---

## Testing and Validation

### 1. Verify Infrastructure Deployment

```bash
# Check all CloudFormation stacks are deployed successfully
aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].StackStatus' \
  --region $REGION

# Verify S3 bucket creation and initial log delivery
aws s3 ls s3://${S3_BUCKET_NAME}/logs/ --recursive --human-readable | head -10

# Check Kinesis stream health and shard usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name IncomingBytes \
  --dimensions Name=StreamName,Value=CWLogCollectorDataStream \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region $REGION

# Verify KMS key encryption is working
aws kms describe-key \
  --key-id $KMS_KEY_ARN \
  --region $REGION
```

### 2. Monitor Kinesis Shard Usage

The solution includes automated monitoring for Kinesis shard usage to prevent throttling:

```bash
# Check current shard usage alarm status
aws cloudwatch describe-alarms \
  --alarm-names "CWLogCollectorDataStream-ShardUsageAlarm" \
  --region $REGION

# View recent alarm history
aws cloudwatch describe-alarm-history \
  --alarm-name "CWLogCollectorDataStream-ShardUsageAlarm" \
  --max-records 5 \
  --region $REGION
```

### 3. Test Hunters AI Access

```bash
# Verify the IAM role configuration
HUNTERS_ROLE_ARN=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`HuntersAIReadOnlyRoleName`].OutputValue' \
  --output text \
  --region $REGION)

echo "Hunters AI Role ARN: arn:aws:iam::${LOGGING_ACCOUNT_ID}:role/${HUNTERS_ROLE_ARN}"

# Verify KMS key access for Hunters AI role
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${LOGGING_ACCOUNT_ID}:role/${HUNTERS_ROLE_ARN} \
  --action-names kms:Decrypt \
  --resource-arns $KMS_KEY_ARN \
  --region $REGION
```

### 4. Validate Automatic Log Group Subscription

```bash
# Check Lambda function logs in member accounts
aws logs tail /aws/lambda/hunters-log-forwarding-LogGroupSubscribeLambda-* \
  --follow --region $REGION

# Monitor Lambda function health via CloudWatch alarms
aws cloudwatch describe-alarms \
  --alarm-name-prefix "hunters-log-forwarding" \
  --region $REGION

# Create a test log group to verify automatic subscription
aws logs create-log-group \
  --log-group-name /test/hunters-integration \
  --region $REGION

# Wait a few seconds for EventBridge and Lambda to trigger

# Verify subscription was created automatically
aws logs describe-subscription-filters \
  --log-group-name /test/hunters-integration \
  --region $REGION

# Clean up test log group
aws logs delete-log-group \
  --log-group-name /test/hunters-integration \
  --region $REGION
```

---

## Troubleshooting and Best Practices

### Common Issues and Solutions

#### Issue 1: Kinesis Shard Throttling

**Symptoms**:
- High `WriteProvisionedThroughputExceeded` errors
- Increased latency in log delivery
- CloudWatch alarm triggered

**Solution**:
```bash
# Check current shard utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kinesis \
  --metric-name IncomingBytes \
  --dimensions Name=StreamName,Value=CWLogCollectorDataStream \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum \
  --region $REGION

# Scale up shards if needed (update stack parameter)
aws cloudformation update-stack \
  --stack-name hunters-central-logging \
  --use-previous-template \
  --parameters ParameterKey=KinesisShardCount,ParameterValue=2 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION
```

**Best Practice**: Monitor shard utilization proactively and set up auto-scaling based on incoming byte metrics.

---

#### Issue 2: Lambda Function Failures

**Symptoms**:
- Log groups not automatically subscribed
- Errors in Lambda CloudWatch Logs
- CloudWatch alarm triggered

**Solution**:
```bash
# Check Lambda function error rates
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=hunters-log-forwarding-LogGroupSubscribeLambda-* \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region $REGION

# View recent Lambda logs for troubleshooting
aws logs filter-log-events \
  --log-group-name /aws/lambda/hunters-log-forwarding-LogGroupSubscribeLambda-* \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern "ERROR" \
  --region $REGION
```

**Common causes**:
- Insufficient IAM permissions
- Destination ARN misconfigured
- Log group already has 2 subscription filters (AWS limit)

---

#### Issue 3: S3 Access Denied Errors

**Symptoms**:
- Logs not appearing in S3
- Access denied errors in Kinesis Firehose logs

**Solution**:
```bash
# Verify bucket policy allows Firehose access
aws s3api get-bucket-policy \
  --bucket $S3_BUCKET_NAME \
  --query Policy \
  --output text | jq '.Statement[] | select(.Sid == "FirehoseAccess")'

# Check Firehose IAM role permissions
FIREHOSE_ROLE=$(aws cloudformation describe-stacks \
  --stack-name hunters-central-logging \
  --query 'Stacks[0].Outputs[?OutputKey==`FirehoseDeliveryRoleArn`].OutputValue' \
  --output text)

aws iam get-role-policy \
  --role-name $(echo $FIREHOSE_ROLE | cut -d'/' -f2) \
  --policy-name FirehoseS3Access \
  --region $REGION
```

---

#### Issue 4: KMS Encryption Errors

**Symptoms**:
- "Access Denied" when writing to S3
- GuardDuty export failures
- CloudTrail log delivery errors

**Solution**:
```bash
# Verify KMS key policy includes all required services
aws kms get-key-policy \
  --key-id $KMS_KEY_ARN \
  --policy-name default \
  --query Policy \
  --output text | jq '.Statement'

# Test KMS key access
aws kms encrypt \
  --key-id $KMS_KEY_ARN \
  --plaintext "test" \
  --query CiphertextBlob \
  --output text \
  --region $REGION
```

---

### Best Practices

#### 1. Cost Optimization

- **Set appropriate S3 lifecycle policies** to transition older logs to cheaper storage classes
- **Monitor Kinesis shard usage** and right-size based on actual throughput
- **Use Firehose buffering** to reduce the number of S3 PutObject requests
- **Enable S3 Intelligent-Tiering** for automatic cost optimization

```bash
# Update lifecycle rule
aws s3api put-bucket-lifecycle-configuration \
  --bucket $S3_BUCKET_NAME \
  --lifecycle-configuration file://lifecycle-policy.json
```

**lifecycle-policy.json**:
```json
{
  "Rules": [
    {
      "Id": "MoveToIA",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 180,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

---

#### 2. Security Best Practices

- **Enable MFA Delete** on the S3 bucket for additional protection
- **Use VPC endpoints** for Kinesis and S3 to keep traffic within AWS network
- **Regularly rotate the Hunters AI external ID**
- **Enable AWS CloudTrail** for API activity monitoring
- **Use SCPs** (Service Control Policies) to prevent accidental deletion of logging resources

```bash
# Enable MFA Delete (requires root account credentials)
aws s3api put-bucket-versioning \
  --bucket $S3_BUCKET_NAME \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::ACCOUNT-ID:mfa/root-account-mfa-device TOKENCODE"
```

---

#### 3. Monitoring and Alerting

Set up comprehensive monitoring:

```bash
# Create dashboard for key metrics
aws cloudwatch put-dashboard \
  --dashboard-name HuntersAI-SIEM-Monitoring \
  --dashboard-body file://dashboard-config.json
```

**Key metrics to monitor**:
- Kinesis `IncomingBytes` and `WriteProvisionedThroughputExceeded`
- Firehose `DeliveryToS3.Success` and `DeliveryToS3.DataFreshness`
- Lambda `Errors`, `Throttles`, and `Duration`
- S3 bucket size and object count

---

#### 4. Disaster Recovery

- **Enable Cross-Region Replication** for S3 bucket
- **Back up CloudFormation templates** to version control
- **Document external dependencies** (Hunters AI external ID, account IDs)
- **Test recovery procedures** regularly

```bash
# Enable S3 replication
aws s3api put-bucket-replication \
  --bucket $S3_BUCKET_NAME \
  --replication-configuration file://replication-config.json
```

---

## CloudFormation Template Parameters Reference

### HuntersAI_logging_account_stack Parameters

| Parameter | Type | Description | Default | Required |
|-----------|------|-------------|---------|----------|
| `BucketName` | String | Name of the S3 bucket to store logs | - | Yes |
| `LogsPrefix` | String | Prefix for organizing logs in S3 | `logs/` | No |
| `S3DataExpirationInDays` | String | Number of days before log deletion | `365` | No |
| `CreateNewBucket` | String | Create new bucket or use existing | `true` | No |
| `MemberAccountIDs` | CommaDelimitedList | List of member account IDs | - | Yes |
| `OrgId` | String | AWS Organization ID | - | Yes |
| `DetectorID` | String | GuardDuty detector ID | - | Yes |
| `GDPrimaryRegion` | String | GuardDuty primary region | `us-east-1` | No |
| `GDAdministratorAccount` | String | GuardDuty administrator account ID | - | Yes |
| `MonitoredCloudTrailARN` | String | ARN of CloudTrail trail | - | Yes |
| `KinesisStreamName` | String | Name of Kinesis Data Stream | `CWLogCollectorDataStream` | No |
