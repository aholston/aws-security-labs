# Implementation Guide: ABAC with IAM Identity Center and EC2

## Overview

This guide provides step-by-step instructions for implementing attribute-based access control (ABAC) using IAM Identity Center session tags and EC2 resource tags. Unlike traditional role-based access control, ABAC provides dynamic permissions that automatically adapt based on user attributes and resource characteristics.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with IAM Identity Center enabled at organization level
- IAM Identity Center internal directory configured
- Federated user `devuser` in `Developers` group
- Basic permission set `DevAccessProd` assigned to prod-account
- Admin access to management account and prod-account

### Account Structure
| Account | Purpose | Admin Access |
|---------|---------|--------------|
| management | IAM Identity Center administration | Organization admin |
| prod-account | EC2 resources and ABAC testing | prod-admin |
| security-account | Centralized logging (optional) | security-admin |

---

## Step 1: Configure Attributes for Access Control

### 1.1 Access Management Account
- Log in to **management account** with organization admin credentials
- Navigate to **IAM Identity Center** service

### 1.2 Enable ABAC Configuration
- **Settings** → **Attributes for access control** tab
- If disabled, click **Enable attributes for access control**
- Click **Manage attributes**

### 1.3 Configure Attribute Mappings
Add the following attribute mappings:

**Attribute 1: Department**
- **Key**: `Department`
- **Value**: `${path:enterprise.department}`

**Attribute 2: Cost Center**
- **Key**: `CostCenter`
- **Value**: `${path:enterprise.costCenter}`

**Attribute 3: Job Title**
- **Key**: `Title`
- **Value**: `${path:enterprise.title}`

### 1.4 Save Configuration
- Click **Save changes**
- Note: Changes take effect immediately for new sessions

---

## Step 2: Configure User Attributes

### 2.1 Access User Management
- **Users** → Select `devuser` → **Edit**

### 2.2 Set Required Attributes
Configure the following user attributes:

```
Title: Developer
Cost Center: CC-1001
Department: Engineering
```

### 2.3 Verify Attribute Mapping
Ensure attribute field names match the enterprise mapping:
- **Department** field → `${path:enterprise.department}`
- **Cost Center** field → `${path:enterprise.costCenter}`
- **Title** field → `${path:enterprise.title}`

---

## Step 3: Create Tagged EC2 Instances

### 3.1 Access prod-account
- Log in as `prod-admin` IAM user with MFA
- Navigate to **EC2** service → **Instances**

### 3.2 Create Engineering Development Instance

**Launch Instance**:
- **Name**: `engineering-dev-web-server`
- **AMI**: Amazon Linux 2023 (free tier)
- **Instance type**: t2.micro
- **Key pair**: Create new or select existing
- **Security group**: Default VPC security group
- **Storage**: Default (8 GB gp3)

**Advanced Details → User Data** (optional):
```bash
#!/bin/bash
yum update -y
echo "Engineering Development Server" > /var/www/html/index.html
```

**Tags**:
```
Name: engineering-dev-web-server
Department: Engineering
CostCenter: CC-1001
Environment: Development
Project: WebApp
Owner: DevTeam
```

### 3.3 Create Engineering Production Instance

**Launch Instance**:
- **Name**: `engineering-prod-web-server`
- **Configuration**: Same as development instance

**Tags**:
```
Name: engineering-prod-web-server
Department: Engineering
CostCenter: CC-1001
Environment: Production
Project: WebApp
Owner: DevTeam
```

### 3.4 Create Finance Production Instance

**Launch Instance**:
- **Name**: `finance-prod-billing-server`
- **Configuration**: Same as previous instances

**Tags**:
```
Name: finance-prod-billing-server
Department: Finance
CostCenter: CC-2002
Environment: Production
Project: Billing
Owner: FinanceTeam
```

### 3.5 Verify Instance Creation
- Confirm all 3 instances are running
- Verify tags are correctly applied to each instance
- Note instance IDs for testing reference

---

## Step 4: Implement ABAC Policy

### 4.1 Access Permission Set Configuration
- **Management account** → **IAM Identity Center**
- **Permission sets** → **DevAccessProd** → **Edit**

### 4.2 Update Inline Policy
- **Permissions** → **Inline policy** → **Edit**
- Replace existing policy with ABAC policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2DescribeAll",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInstanceAttribute",
        "ec2:DescribeRegions",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DepartmentBasedInstanceAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Department": "${aws:PrincipalTag/Department}",
          "ec2:ResourceTag/CostCenter": "${aws:PrincipalTag/CostCenter}"
        }
      }
    },
    {
      "Sid": "DeveloperEnvironmentRestrictions",
      "Effect": "Allow",
      "Action": [
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Title": "Developer",
          "ec2:ResourceTag/Department": "${aws:PrincipalTag/Department}",
          "ec2:ResourceTag/Environment": "Development"
        }
      }
    },
    {
      "Sid": "DenyProductionModification",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Title": "Developer",
          "ec2:ResourceTag/Environment": "Production"
        }
      }
    }
  ]
}
```

### 4.3 Provision Changes
- **Save changes**
- **Provision to accounts** → Select prod-account
- Wait for provisioning to complete (green checkmark)

---

## Step 5: Test ABAC Implementation

### 5.1 Federated Authentication
- Open **IAM Identity Center access portal** URL
- Sign in as `devuser` with configured password
- Select **prod-account**
- Choose **DevAccessProd** role
- Click **Management console**

### 5.2 Verify Session Identity
- Navigate to **EC2** service
- **Optional**: Open CloudShell and run:
```bash
aws sts get-caller-identity
```

Expected output shows federated session:
```json
{
    "UserId": "AROA...:devuser",
    "Account": "PROD-ACCOUNT-ID",
    "Arn": "arn:aws:sts::ACCOUNT:assumed-role/AWSReservedSSO_DevAccessProd_.../devuser"
}
```

### 5.3 Test Discovery Permissions
- **EC2** → **Instances**
- **✅ Expected**: See all 3 instances with full details
- **Validation**: Statement 1 (EC2DescribeAll) working correctly

### 5.4 Test Department-Based Access Control

#### Engineering Development Instance
- **Right-click** instance → **Instance state** → **Stop**
- **✅ Expected**: Action succeeds
- **Reason**: Department=Engineering + CostCenter=CC-1001 match session tags

#### Engineering Production Instance
- **Right-click** instance → **Instance state** → **Stop**
- **✅ Expected**: Action succeeds
- **Reason**: Department + CostCenter match (environment irrelevant for stop)

#### Finance Production Instance
- **Right-click** instance → **Instance state** → **Stop**
- **❌ Expected**: AccessDenied error
- **Reason**: Department=Finance ≠ Engineering, CostCenter mismatch

### 5.5 Test Environment-Based Termination

#### Engineering Development Instance
- **Right-click** instance → **Instance state** → **Terminate**
- **✅ Expected**: Action succeeds
- **Reason**: Title=Developer + Department=Engineering + Environment=Development

#### Engineering Production Instance
- **Right-click** instance → **Instance state** → **Terminate**
- **❌ Expected**: AccessDenied error
- **Reason**: Statement 4 explicit deny for Developer + Production

#### Finance Production Instance
- **Right-click** instance → **Instance state** → **Terminate**
- **❌ Expected**: AccessDenied error
- **Reason**: Department mismatch prevents Statement 3 conditions

---

## Step 6: Advanced Testing Scenarios

### 6.1 Create Additional Test User

**In Management Account**:
- **Users** → **Create user**
- **Username**: `financeuser`
- **Attributes**:
  ```
  Title: Analyst
  Cost Center: CC-2002
  Department: Finance
  ```
- **Groups**: Add to `Developers` group
- **Account assignment**: Assign `DevAccessProd` to prod-account

### 6.2 Test Cross-Department Access
- Login as `financeuser`
- Attempt to manage Engineering instances
- **Expected**: Access denied due to department mismatch

### 6.3 Test Role-Based Restrictions
- Create user with `Title: Manager`
- Test if manager can terminate production instances
- **Current policy**: Manager would also be denied (only Developer title allowed)

---

## Troubleshooting Guide

### Common Issues

**Session Tags Not Flowing**
- **Symptoms**: Access denied to all resources despite correct tags
- **Causes**: Attributes for access control not enabled
- **Solutions**: 
  - Verify ABAC is enabled in Settings
  - Check attribute mappings are exact
  - Re-provision permission set

**Partial Access Working**
- **Symptoms**: Some operations work, others don't
- **Causes**: Missing conditions or explicit deny conflicts
- **Solutions**:
  - Review each policy statement independently
  - Check for typos in condition keys
  - Verify resource tags are applied correctly

**Case Sensitivity Issues**
- **Symptoms**: Access denied despite matching attributes
- **Causes**: Tag values don't match exactly
- **Solutions**:
  - Verify exact case: "Engineering" ≠ "engineering"
  - Check for extra spaces in tag values
  - Use StringLike instead of StringEquals for flexibility

### Validation Commands

**Check EC2 instance tags**:
```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Department`].Value|[0],Tags[?Key==`Environment`].Value|[0]]'
```

**Verify session tags** (requires CloudTrail):
- Search for `AssumeRoleWithSAML` events
- Examine `principalTags` in event details

**Test specific conditions**:
```bash
# This should work for Engineering instances
aws ec2 describe-instances --filters "Name=tag:Department,Values=Engineering"

# This should fail if session tags aren't matching
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
```

---

## Security Best Practices

### Attribute Management
- **Consistent naming**: Use standardized tag keys across organization
- **Case sensitivity**: Establish and enforce case conventions
- **Value validation**: Implement allowed values for critical attributes
- **Regular auditing**: Review attribute assignments periodically

### Policy Design
- **Least privilege**: Start with minimal permissions, expand as needed
- **Explicit deny**: Use deny statements for critical protection
- **Resource specificity**: Avoid wildcards where possible
- **Condition complexity**: Balance security with maintainability

### Operational Considerations
- **Testing procedures**: Validate ABAC changes in non-production first
- **Emergency access**: Maintain break-glass procedures
- **Documentation**: Keep attribute mappings and policies documented
- **Monitoring**: Set up CloudTrail alerts for ABAC policy violations

---

## Success Validation

✅ **Attributes for access control enabled** and properly configured  
✅ **Session tags flowing** from Identity Center to AWS sessions  
✅ **Department isolation working** (Engineering ≠ Finance access)  
✅ **Cost center restrictions enforced** (CC-1001 ≠ CC-2002)  
✅ **Environment protection active** (Development ≠ Production for terminate)  
✅ **Role-based limitations applied** (Developer restrictions working)  
✅ **Explicit deny precedence confirmed** (Production termination blocked)  

This implementation demonstrates enterprise-grade ABAC that automatically scales with organizational growth while maintaining security boundaries and operational efficiency.