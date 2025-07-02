# Implementation Guide: GuardDuty Automated Threat Response

## Overview

This guide implements automated EC2 threat isolation triggered by GuardDuty network-based findings. The system detects cryptocurrency mining, DNS exfiltration, and malware communication, then automatically quarantines compromised instances while preserving forensic evidence.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with GuardDuty delegated administration
- Security Hub cross-account aggregation established
- EventBridge integration with Security Hub configured
- Existing automated incident response pipeline (from Security Hub labs)

### Account Structure
| Account | Purpose | Access Method |
|---------|---------|---------------|
| security-account | GuardDuty administrator, Security Hub aggregation, Lambda automation | sec-admin |
| prod-account | Target resources, GuardDuty member, remediation role | prod-admin |

---

## Phase 1: Multi-Account GuardDuty Setup

### Step 1.1: Enable GuardDuty with Delegated Administration

**Management Account Setup:**
```bash
# Enable trusted access from AWS Organizations → Services → GuardDuty
# Delegate administrator to security-account (111111111111)
```

**Security Account Configuration:**
- **GuardDuty**: Enable with all data sources (VPC Flow Logs, DNS, S3, Malware Protection)
- **Member Accounts**: Add prod-account via organization integration
- **Security Hub**: Enable integration for automatic finding forwarding

### Step 1.2: Verify Cross-Account Integration

```bash
# Check member account status
aws guardduty list-members \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --profile sec-admin \
  --region us-east-1

# Test finding flow with sample finding
aws guardduty create-sample-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --finding-types 'CryptoCurrency:EC2/BitcoinTool.B!DNS' \
  --profile prod-admin \
  --region us-east-1
```

---

## Phase 2: Cross-Account Remediation Role

### Step 2.1: Create Remediation Role in prod-account

**Trust Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111111111111:role/service-role/GuardDutyThreatResponse-role-XXXXX"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "guardduty-threat-response-2025"
                }
            }
        }
    ]
}
```

**Permissions Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EC2IsolationActions",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeVolumes",
                "ec2:ModifyInstanceAttribute",
                "ec2:CreateSecurityGroup",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:CreateSnapshot",
                "ec2:CreateTags"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyDestructiveActions",
            "Effect": "Deny",
            "Action": [
                "ec2:TerminateInstances",
                "ec2:StopInstances",
                "ec2:DeleteVolume"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Phase 3: Automated Threat Response Lambda

### Step 3.1: Create Lambda Function

**Console Navigation:** `security-account` → **AWS Lambda** → **Create function**

**Function Configuration:**
- **Name**: `GuardDutyThreatResponse`
- **Runtime**: Python 3.12
- **Timeout**: 60 seconds
- **Memory**: 256 MB

**Key Lambda Functions:**
- **Threat Detection**: Identifies network-based threats (crypto mining, DNS exfiltration, backdoors)
- **Cross-Account Access**: Assumes remediation role with External ID security
- **Instance Isolation**: Replaces security groups with quarantine security group
- **Forensic Preservation**: Creates EBS snapshots and tags instances with incident metadata

### Step 3.2: Update Lambda Execution Role

**Add cross-account permissions:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AssumeGuardDutyRemediationRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::*:role/GuardDutyRemediationRole"
        },
        {
            "Sid": "SecurityHubUpdateFindings",
            "Effect": "Allow",
            "Action": [
                "securityhub:BatchUpdateFindings",
                "securityhub:GetFindings"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Phase 4: EventBridge Integration

### Step 4.1: Create Threat Response EventBridge Rule

**Event Pattern:**
```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "ProductName": ["GuardDuty"],
      "Severity": {
        "Label": ["HIGH", "CRITICAL"]
      },
      "Types": [
        {
          "wildcard": "*CryptoCurrency:EC2*"
        },
        {
          "wildcard": "*Trojan:EC2*"
        },
        {
          "wildcard": "*Backdoor:EC2*"
        }
      ]
    }
  }
}
```

**Targets:**
- **SNS Topic**: `IamPolicyAlerts` (human notification)
- **Lambda Function**: `GuardDutyThreatResponse` (automated response)

### Step 4.2: Grant EventBridge Permissions

```bash
aws lambda add-permission \
  --function-name GuardDutyThreatResponse \
  --statement-id allow-eventbridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:us-east-1:111111111111:rule/GuardDutyThreatResponse" \
  --profile sec-admin \
  --region us-east-1
```

---

## Phase 5: Testing and Validation

### Step 5.1: Direct Lambda Testing

**Create test event:**
```bash
echo '{
  "source": "aws.securityhub",
  "detail-type": "Security Hub Findings - Imported",
  "detail": {
    "findings": [{
      "Id": "test-finding-123",
      "Types": ["TTPs/Command and Control/Trojan:EC2-DNSDataExfiltration"],
      "ProductName": "GuardDuty",
      "Severity": {"Label": "HIGH"},
      "AwsAccountId": "222222222222",
      "Resources": [{
        "Type": "AwsEc2Instance",
        "Id": "arn:aws:ec2:us-east-1:222222222222:instance/i-99999999"
      }]
    }]
  }
}' > test-event.json

# Test Lambda function
aws lambda invoke \
  --function-name GuardDutyThreatResponse \
  --payload file://test-event.json \
  --cli-binary-format raw-in-base64-out \
  --profile sec-admin \
  --region us-east-1 \
  response.json
```

### Step 5.2: End-to-End Automation Testing

```bash
# Generate GuardDuty finding to trigger full pipeline
aws guardduty create-sample-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --finding-types 'CryptoCurrency:EC2/BitcoinTool.B!DNS' \
  --profile prod-admin \
  --region us-east-1

# Monitor Lambda execution logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/GuardDutyThreatResponse \
  --start-time $(date -v-10M +%s)000 \
  --profile sec-admin \
  --region us-east-1
```

---

## Troubleshooting Guide

### Common Issues

**EventBridge Not Triggering Lambda:**
- **Cause**: Event pattern doesn't match actual Security Hub finding format
- **Solution**: Use wildcards in Types array to match GuardDuty finding variations
- **Validation**: Check Security Hub findings for exact Types format

**Cross-Account Role Assumption Failures:**
- **Cause**: Trust policy principal mismatch or missing External ID
- **Solution**: Get actual Lambda role ARN and update trust policy
- **Validation**: Test role assumption manually with correct External ID

**Instance Isolation Failures:**
- **Cause**: Insufficient EC2 permissions or missing security group
- **Solution**: Verify remediation role has ModifyInstanceAttribute permissions
- **Validation**: Check Lambda logs for specific EC2 API errors

### Validation Commands

```bash
# Check Lambda role ARN
aws lambda get-function \
  --function-name GuardDutyThreatResponse \
  --profile sec-admin \
  --query 'Configuration.Role'

# Verify EventBridge rule pattern
aws events describe-rule \
  --name GuardDutyThreatResponse \
  --profile sec-admin

# Test cross-account role assumption
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-ID:role/GuardDutyRemediationRole \
  --role-session-name test \
  --external-id guardduty-threat-response-2025 \
  --profile sec-admin
```

---

## Security Best Practices

### Automated Containment
- **Non-destructive isolation**: Instances preserved for investigation
- **Forensic snapshots**: EBS volumes captured before remediation
- **Security tagging**: Complete incident metadata for tracking

### Cross-Account Security
- **External ID requirement**: Prevents confused deputy attacks
- **Least-privilege permissions**: Minimal required actions only
- **Explicit deny statements**: Protection against destructive operations

### Audit and Compliance
- **Complete CloudTrail logging**: All remediation actions recorded
- **Security Hub integration**: Finding status updates for compliance
- **Comprehensive alerting**: Immediate human notification with automated response

---

## Success Validation

✅ **GuardDuty multi-account setup** with delegated administration  
✅ **Cross-account remediation role** with External ID security  
✅ **Lambda threat response** with intelligent finding processing  
✅ **EventBridge automation** triggering on HIGH/CRITICAL network threats  
✅ **End-to-end isolation** within 60 seconds of finding creation  
✅ **Forensic preservation** with comprehensive incident documentation  

This implementation provides production-ready automated threat response that reduces mean time to containment from hours to seconds while maintaining enterprise security controls and compliance requirements.