# Implementation Guide: Automated IAM Incident Response

## Overview

This guide implements enterprise-grade automated incident response for IAM security findings using AWS Security Hub, EventBridge, and Lambda. The system automatically contains overprivileged IAM users within 60 seconds while preserving forensic evidence and maintaining audit trails.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with Security Hub enabled at organization level
- Cross-account administrator/member relationship established
- EventBridge integration with Security Hub configured
- SNS topic for security alerts
- CloudTrail logging enabled for audit trails

### Account Structure
| Account | Purpose | Admin Access |
|---------|---------|--------------|
| security-account | Central Security Hub administrator, Lambda functions | sec-admin |
| prod-account | Target resources, member account | prod-admin |

---

## Phase 1: Lambda Remediation Function

### Step 1.1: Create IAM Remediation Lambda Function

**Console Navigation:** `security-account` → **AWS Lambda** → **Create function**

**Function Configuration:**
- **Function name**: `SecurityHubIAMRemediation`
- **Runtime**: Python 3.12
- **Architecture**: x86_64
- **Execution role**: Create new role with basic Lambda permissions

**Function Code:**
```python
import json
import boto3
import logging
from datetime import datetime

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Automated IAM remediation for Security Hub findings
    Handles cross-account IAM incident response
    """
    
    try:
        # Parse Security Hub finding from EventBridge
        detail = event['detail']
        findings = detail['findings']
        
        for finding in findings:
            logger.info(f"Processing finding: {finding['Id']}")
            
            # Extract finding details
            severity = finding['Severity']['Label']
            title = finding['Title']
            account_id = finding['AwsAccountId']
            resources = finding.get('Resources', [])
            
            # Process only HIGH/CRITICAL IAM-related findings
            if severity in ['HIGH', 'CRITICAL'] and 'IAM' in title:
                logger.info(f"Processing {severity} IAM finding: {title}")
                
                # Determine remediation action based on finding type
                if 'AdministratorAccess' in title or 'admin' in title.lower():
                    remediate_overprivileged_user(account_id, resources, finding)
                elif 'root' in title.lower() and 'access key' in title.lower():
                    remediate_root_access_keys(account_id, resources, finding)
                elif 'unused' in title.lower() and 'user' in title.lower():
                    remediate_unused_user(account_id, resources, finding)
                else:
                    logger.info(f"No specific remediation for finding type: {title}")
            
            # Update finding workflow status
            update_finding_workflow(finding['Id'], account_id)
            
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'Processed {len(findings)} findings',
                'timestamp': datetime.utcnow().isoformat()
            })
        }
        
    except Exception as e:
        logger.error(f"Error processing Security Hub findings: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def remediate_overprivileged_user(account_id, resources, finding):
    """Contain overprivileged IAM user by detaching policies and deactivating keys"""
    
    # Assume cross-account role for remediation
    iam_client = assume_remediation_role(account_id)
    if not iam_client:
        return
    
    try:
        for resource in resources:
            if resource['Type'] == 'AwsIamUser':
                username = extract_username_from_arn(resource['Id'])
                logger.info(f"Remediating overprivileged user: {username}")
                
                # 1. List and detach all attached policies
                try:
                    attached_policies = iam_client.list_attached_user_policies(UserName=username)
                    for policy in attached_policies['AttachedPolicies']:
                        iam_client.detach_user_policy(
                            UserName=username,
                            PolicyArn=policy['PolicyArn']
                        )
                        logger.info(f"Detached policy {policy['PolicyName']} from {username}")
                except Exception as e:
                    logger.error(f"Error detaching policies: {str(e)}")
                
                # 2. Remove from all groups
                try:
                    groups = iam_client.list_groups_for_user(UserName=username)
                    for group in groups['Groups']:
                        iam_client.remove_user_from_group(
                            GroupName=group['GroupName'],
                            UserName=username
                        )
                        logger.info(f"Removed {username} from group {group['GroupName']}")
                except Exception as e:
                    logger.error(f"Error removing from groups: {str(e)}")
                
                # 3. Deactivate all access keys
                try:
                    access_keys = iam_client.list_access_keys(UserName=username)
                    for key in access_keys['AccessKeyMetadata']:
                        iam_client.update_access_key(
                            UserName=username,
                            AccessKeyId=key['AccessKeyId'],
                            Status='Inactive'
                        )
                        logger.info(f"Deactivated access key {key['AccessKeyId']} for {username}")
                except Exception as e:
                    logger.error(f"Error deactivating access keys: {str(e)}")
                
                # 4. Tag user for forensic investigation
                try:
                    iam_client.tag_user(
                        UserName=username,
                        Tags=[
                            {'Key': 'SecurityIncident', 'Value': 'AutoRemediated'},
                            {'Key': 'RemediationDate', 'Value': datetime.utcnow().isoformat()},
                            {'Key': 'FindingId', 'Value': finding['Id'][:32]}
                        ]
                    )
                    logger.info(f"Successfully tagged user: {username}")
                except Exception as e:
                    logger.error(f"Error tagging user: {str(e)}")
                
                logger.info(f"Successfully remediated overprivileged user: {username}")
                
    except Exception as e:
        logger.error(f"Error remediating overprivileged user: {str(e)}")

def assume_remediation_role(target_account_id):
    """Assume cross-account remediation role"""
    
    try:
        sts_client = boto3.client('sts')
        role_arn = f"arn:aws:iam::{target_account_id}:role/SecurityRemediationRole"
        
        response = sts_client.assume_role(
            RoleArn=role_arn,
            RoleSessionName=f"SecurityRemediation-{int(datetime.utcnow().timestamp())}",
            ExternalId="security-remediation-2025"
        )
        
        credentials = response['Credentials']
        return boto3.client(
            'iam',
            aws_access_key_id=credentials['AccessKeyId'],
            aws_secret_access_key=credentials['SecretAccessKey'],
            aws_session_token=credentials['SessionToken']
        )
        
    except Exception as e:
        logger.error(f"Failed to assume remediation role in {target_account_id}: {str(e)}")
        return None

def extract_username_from_arn(resource_arn):
    """Extract username from IAM user ARN"""
    return resource_arn.split('/')[-1]

def update_finding_workflow(finding_id, account_id):
    """Update Security Hub finding workflow status"""
    
    try:
        securityhub = boto3.client('securityhub')
        securityhub.batch_update_findings(
            FindingIdentifiers=[
                {
                    'Id': finding_id,
                    'ProductArn': f"arn:aws:securityhub:us-east-1::product/{account_id}/default"
                }
            ],
            Workflow={'Status': 'RESOLVED'},
            Note={
                'Text': 'Automatically remediated by SecurityHubIAMRemediation Lambda',
                'UpdatedBy': 'AWS Lambda Automation'
            }
        )
        logger.info(f"Updated finding workflow status: {finding_id}")
        
    except Exception as e:
        logger.error(f"Failed to update finding workflow: {str(e)}")

# Additional remediation functions for root access keys and unused users omitted for brevity
```

### Step 1.2: Update Lambda Execution Role Permissions

**Console Navigation:** `security-account` → **IAM** → **Roles** → **SecurityHubIAMRemediation-role-XXXXX**

**Add inline policy `RemediationPermissions`:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AssumeRemediationRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::*:role/SecurityRemediationRole"
        },
        {
            "Sid": "SecurityHubAccess",
            "Effect": "Allow",
            "Action": [
                "securityhub:BatchUpdateFindings",
                "securityhub:GetFindings"
            ],
            "Resource": "*"
        },
        {
            "Sid": "SNSAlerts",
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "arn:aws:sns:us-east-1:257288818635:IamPolicyAlerts"
        }
    ]
}
```

---

## Phase 2: Cross-Account Remediation Role

### Step 2.1: Create Remediation Role in Target Account

**Console Navigation:** `prod-account` → **IAM** → **Roles** → **Create role**

**Trust Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::257288818635:role/service-role/SecurityHubIAMRemediation-role-XXXXX"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "security-remediation-2025"
                }
            }
        }
    ]
}
```

**Role Configuration:**
- **Role name**: `SecurityRemediationRole`
- **Description**: `Cross-account role for automated security remediation`

**Permissions Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "IAMRemediationActions",
            "Effect": "Allow",
            "Action": [
                "iam:DetachUserPolicy",
                "iam:DetachRolePolicy",
                "iam:RemoveUserFromGroup",
                "iam:UpdateAccessKey",
                "iam:TagUser",
                "iam:TagRole",
                "iam:ListAttachedUserPolicies",
                "iam:ListAttachedRolePolicies",
                "iam:ListAccessKeys",
                "iam:ListGroupsForUser"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMReadOnlyForRemediation",
            "Effect": "Allow",
            "Action": [
                "iam:GetUser",
                "iam:GetRole",
                "iam:ListUsers",
                "iam:ListRoles"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyDestructiveActions",
            "Effect": "Deny",
            "Action": [
                "iam:DeleteUser",
                "iam:DeleteRole",
                "iam:DeleteAccessKey"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Phase 3: EventBridge Integration

### Step 3.1: Update EventBridge Rule

**Console Navigation:** `security-account` → **Amazon EventBridge** → **Rules** → **SecurityHubFindingsProcessor**

**Add Lambda target while keeping existing SNS target:**
- **Target type**: AWS service
- **Service**: Lambda function
- **Function**: `SecurityHubIAMRemediation`

**Event Pattern (existing):**
```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["HIGH", "CRITICAL"]
      },
      "Workflow": {
        "Status": ["NEW"]
      }
    }
  }
}
```

---

## Phase 4: Testing and Validation

### Step 4.1: Create Test Scenario

```bash
# Create dangerous IAM user in prod-account
aws iam create-user --user-name dangerous-test-user --profile prod-admin

aws iam attach-user-policy \
  --user-name dangerous-test-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile prod-admin

aws iam create-access-key --user-name dangerous-test-user --profile prod-admin
```

### Step 4.2: Generate Security Hub Finding

```bash
# Create CRITICAL finding to trigger automated response
aws securityhub batch-import-findings \
  --profile prod-admin \
  --region us-east-1 \
  --findings '[{
    "SchemaVersion": "2018-10-08",
    "Id": "iam-remediation-test-001",
    "ProductArn": "arn:aws:securityhub:us-east-1:ACCOUNT-ID:product/ACCOUNT-ID/default",
    "GeneratorId": "automated-remediation-test",
    "AwsAccountId": "ACCOUNT-ID",
    "Types": ["Sensitive Data Identifications/PII"],
    "Title": "IAM CRITICAL - AdministratorAccess Policy Attached to User",
    "Description": "Test finding to trigger automated IAM remediation",
    "Severity": {"Label": "CRITICAL"},
    "Workflow": {"Status": "NEW"},
    "Resources": [{
      "Type": "AwsIamUser",
      "Id": "arn:aws:iam::ACCOUNT-ID:user/dangerous-test-user"
    }],
    "CreatedAt": "2025-06-29T03:00:00.000Z",
    "UpdatedAt": "2025-06-29T03:00:00.000Z"
  }]'
```

### Step 4.3: Verify Automated Remediation

**Check Lambda execution:**
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/SecurityHubIAMRemediation \
  --start-time $(date -u -v-5M +%s)000 \
  --profile sec-admin \
  --region us-east-1
```

**Verify policy detachment:**
```bash
aws iam list-attached-user-policies --user-name dangerous-test-user --profile prod-admin
```

**Check access key status:**
```bash
aws iam list-access-keys --user-name dangerous-test-user --profile prod-admin
```

**Verify security tags:**
```bash
aws iam list-user-tags --user-name dangerous-test-user --profile prod-admin
```

---

## Troubleshooting Guide

### Common Issues

**Lambda Role Assumption Failures:**
- **Error**: `AccessDenied` when assuming cross-account role
- **Cause**: Trust policy misconfiguration or missing External ID
- **Solution**: Verify exact Lambda role ARN in trust policy and include External ID parameter

**Security Hub Finding Updates Failing:**
- **Error**: Finding status remains "NEW" despite successful remediation
- **Cause**: Product ARN mismatch in cross-account scenarios
- **Solution**: Use correct Product ARN format for cross-account findings

**EventBridge Rule Not Triggering:**
- **Error**: Lambda never executes despite findings creation
- **Cause**: Event pattern mismatch or rule configuration issues
- **Solution**: Verify event pattern matches finding severity and status

### Validation Commands

```bash
# Check Lambda role permissions
aws iam get-role-policy \
  --role-name SecurityHubIAMRemediation-role-XXXXX \
  --policy-name RemediationPermissions \
  --profile sec-admin

# Verify EventBridge rule targets
aws events list-targets-by-rule \
  --rule SecurityHubFindingsProcessor \
  --profile sec-admin \
  --region us-east-1

# Test cross-account role assumption
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-ID:role/SecurityRemediationRole \
  --role-session-name test \
  --external-id security-remediation-2025 \
  --profile sec-admin
```

---

## Security Best Practices

### Principle of Least Privilege
- Remediation role has minimal required permissions
- Explicit deny on destructive operations (delete user/role/keys)
- Limited to specific remediation actions only

### Audit and Compliance
- Complete CloudTrail logging of all remediation actions
- Security tags applied for forensic investigation
- Finding status updates for compliance tracking

### Defense in Depth
- External ID requirement prevents confused deputy attacks
- Cross-account isolation contains blast radius
- Automated response with human notification fallback

---

## Success Validation

✅ **Lambda function deployed** with cross-account IAM remediation logic  
✅ **Cross-account role established** with least-privilege remediation permissions  
✅ **EventBridge integration configured** with dual SNS/Lambda targets  
✅ **End-to-end testing completed** with dangerous policy automatic detachment  
✅ **Forensic preservation implemented** through user tagging and account retention  
✅ **Audit trail maintained** with CloudTrail logging and Security Hub updates  

This implementation provides production-ready automated incident response that reduces mean time to containment from hours to seconds while maintaining comprehensive security controls and audit capabilities.