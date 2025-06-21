# Implementation Guide: Cross-Account Role Chain

## Overview

This guide provides step-by-step instructions for implementing a 3-tier cross-account role assumption chain using IAM Identity Center federation, intermediate roles, and External ID security.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with multiple member accounts
- IAM Identity Center enabled at organization level
- Federated user `devuser` configured with existing permission set
- Admin access to prod-account and security-account

### Account Structure
| Account | Purpose | Admin User |
|---------|---------|------------|
| management | Organization root, Identity Center | N/A |
| prod-account | Intermediate role hosting | prod-admin |
| security-account | Target resources, logging | security-admin |

---

## Step 1: Create Intermediate Role in prod-account

### 1.1 Access prod-account
- Log in as `prod-admin` IAM user with MFA
- Navigate to **IAM** → **Roles** → **Create role**

### 1.2 Configure Trust Policy
- Select **Custom trust policy**
- Use this template (replace PROD-ACCOUNT-ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PROD-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringLike": {
          "aws:userid": "AROA*:devuser"
        }
      }
    }
  ]
}
```

### 1.3 Role Configuration
- **Role name**: `CrossAccountIntermediateRole`
- **Description**: `Intermediate role for cross-account access to security logs`

### 1.4 Attach Permissions Policy
Create inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Resource": "arn:aws:iam::SECURITY-ACCOUNT-ID:role/LogAnalystRole"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

**Policy name**: `AssumeTargetRolePolicy`

---

## Step 2: Create Target Role in security-account

### 2.1 Access security-account
- Log in as `security-admin` IAM user with MFA
- Navigate to **IAM** → **Roles** → **Create role**

### 2.2 Configure Trust Policy with External ID
- Select **Custom trust policy**
- Use this template (replace PROD-ACCOUNT-ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PROD-ACCOUNT-ID:role/CrossAccountIntermediateRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        }
      }
    }
  ]
}
```

### 2.3 Role Configuration
- **Role name**: `LogAnalystRole`
- **Description**: `Target role for accessing CloudTrail logs and security resources`

### 2.4 Attach Permissions Policy
Create inline policy for security resource access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::org-cloudtrail-logs-sec123",
        "arn:aws:s3:::org-cloudtrail-logs-sec123/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "events:ListRules",
        "events:DescribeRule"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

**Policy name**: `SecurityLogsAccessPolicy`

---

## Step 3: Update IAM Identity Center Permission Set

### 3.1 Access Management Account
- Log in to management account
- Navigate to **IAM Identity Center**

### 3.2 Update DevAccessProd Permission Set
- **Permission sets** → `DevAccessProd` → **Edit**
- **Permissions** → **Inline policy** → **Edit**
- Replace existing policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Resource": "arn:aws:iam::PROD-ACCOUNT-ID:role/CrossAccountIntermediateRole"
    }
  ]
}
```

### 3.3 Provision Changes
- **Save changes**
- **Provision to accounts** (wait for completion)

---

## Step 4: Configure CLI Access for devuser

### 4.1 Setup SSO Profile
```bash
aws configure sso --profile devuser
```

**Configuration values:**
- **SSO session name**: `my-sso`
- **SSO start URL**: Your Identity Center portal URL
- **SSO region**: Your Identity Center region (e.g., us-east-1)
- **SSO registration scopes**: `sso:account:access`

### 4.2 Complete Browser Authentication
- Browser will open automatically
- Log in as `devuser`
- Select prod-account and `DevAccessProd` role
- Complete CLI configuration

---

## Step 5: Test the Role Chain

### 5.1 Verify Initial Identity
```bash
aws sts get-caller-identity --profile devuser
```

**Expected output format:**
```json
{
    "UserId": "AROA...:devuser",
    "Account": "PROD-ACCOUNT-ID",
    "Arn": "arn:aws:sts::PROD-ACCOUNT-ID:assumed-role/AWSReservedSSO_DevAccessProd_.../devuser"
}
```

### 5.2 Assume Intermediate Role
```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::PROD-ACCOUNT-ID:role/CrossAccountIntermediateRole" \
  --role-session-name "test-intermediate-session" \
  --profile devuser
```

### 5.3 Export Intermediate Credentials
```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
```

### 5.4 Assume Target Role with External ID
```bash
aws sts assume-role \
  --role-arn "arn:aws:iam::SECURITY-ACCOUNT-ID:role/LogAnalystRole" \
  --role-session-name "log-analyst-session" \
  --external-id "unique-external-id-12345"
```

### 5.5 Export Target Credentials and Test Access
```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Verify final identity
aws sts get-caller-identity

# Test security resource access
aws s3 ls s3://org-cloudtrail-logs-sec123
aws events list-rules
```

---

## Troubleshooting Guide

### Common Issues

**Trust Policy Condition Failures**
- **Error**: `AccessDenied` when assuming intermediate role
- **Cause**: Incorrect `aws:userid` condition format
- **Solution**: Check actual userid with `aws sts get-caller-identity --profile devuser`

**Missing External ID**
- **Error**: `AccessDenied` when assuming target role
- **Cause**: External ID not provided or incorrect
- **Solution**: Verify External ID matches trust policy exactly

**Permission Boundary Issues**
- **Error**: `AccessDenied` for specific actions
- **Cause**: Insufficient permissions in role policies
- **Solution**: Review and expand inline policies

**Session Token Expiration**
- **Error**: `InvalidToken` errors
- **Cause**: Temporary credentials expired
- **Solution**: Re-assume roles to refresh tokens

### Validation Commands

```bash
# Check role trust relationships
aws iam get-role --role-name CrossAccountIntermediateRole

# Verify permission set provisioning status
aws sso-admin list-account-assignments --instance-arn YOUR-INSTANCE-ARN

# Review CloudTrail for assumption events
aws logs filter-log-events \
  --log-group-name CloudTrail/OrgTrail \
  --filter-pattern "AssumeRole"
```

---

## Security Considerations

### Best Practices Implemented
- **Principle of Least Privilege**: Each role has minimal required permissions
- **Defense in Depth**: Multiple security layers with different conditions
- **Audit Trail**: Complete logging of all role assumptions
- **External ID Security**: Protection against confused deputy attacks

### Security Monitoring
- Monitor CloudTrail for unexpected `AssumeRole` events
- Set up EventBridge rules for cross-account role assumptions
- Review External ID usage patterns regularly
- Implement session duration monitoring

---

## Success Validation

✅ **Intermediate role created** with correct trust policy  
✅ **Target role created** with External ID requirement  
✅ **Permission set updated** with assume role permissions  
✅ **CLI configured** for federated user access  
✅ **End-to-end test successful** with resource access confirmed  
✅ **CloudTrail logging verified** for all assumption events  

This implementation provides a production-ready foundation for complex cross-account access patterns while maintaining security best practices and comprehensive audit capabilities.