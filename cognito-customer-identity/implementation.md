# Implementation Guide: Customer Identity with Amazon Cognito

## Overview

This guide provides step-by-step instructions for implementing customer identity management using Amazon Cognito User Pools and Identity Pools. The implementation demonstrates secure, scalable authentication for customer-facing applications with automatic data isolation and tiered service levels.

---

## Prerequisites

### Required Infrastructure
- AWS Organization with prod-account for application resources
- Admin access to prod-account (`prod-admin` IAM user)
- AWS CLI configured with appropriate credentials
- Basic understanding of JWT tokens and IAM policies

### Account Structure
| Account | Purpose | Access Method |
|---------|---------|---------------|
| prod-account | Customer application resources, Cognito pools | prod-admin IAM user |

---

## Phase 1: User Pool Setup (Customer Authentication)

### Step 1.1: Create User Pool

**Navigate to Amazon Cognito → User pools → Create user pool**

**Configure sign-in experience**:
- **Application type**: Traditional web application
- **Authentication providers**: Cognito user pool
- **Sign-in options**: ✅ Email

**Configure security requirements**:
- **Password policy**: Custom
  - Minimum length: 8 characters
  - ✅ Uppercase letters
  - ✅ Lowercase letters
  - ✅ Numbers
  - ❌ Special characters (for testing simplicity)
- **Multi-factor authentication**: No MFA (lab purposes)

**Configure sign-up experience**:
- **Self-service sign-up**: ❌ Disable (manual user creation)
- **Attribute verification**: Email verification

**Integrate your app**:
- **User pool name**: `CustomerFileAccess`
- **App client name**: `CustomerFileApp`
- **Generate client secret**: ❌ Disable (simplifies CLI testing)

### Step 1.2: Create User Groups

**User pools → CustomerFileAccess → Groups tab**

**Premium Users Group**:
- **Group name**: `premium-users`
- **Description**: `Premium customers with enhanced file access`
- **Precedence**: `1`

**Basic Users Group**:
- **Group name**: `basic-users`
- **Description**: `Standard customers with read-only access`
- **Precedence**: `2`

### Step 1.3: Create Test Users

**Users tab → Create user**

**Premium Customer**:
- **Username**: `alice-premium`
- **Email**: `alice@example.com`
- **Temporary password**: `TempPass123!`
- **Send invitation**: ❌ Disable
- **Add to group**: `premium-users`

**Basic Customer**:
- **Username**: `bob-basic`
- **Email**: `bob@example.com`
- **Temporary password**: `TempPass123!`
- **Send invitation**: ❌ Disable
- **Add to group**: `basic-users`

### Step 1.4: Configure App Client

**App integration tab → App clients → Edit app client**

**Enable required authentication flows**:
- ✅ **ALLOW_ADMIN_USER_PASSWORD_AUTH**
- ✅ **ALLOW_USER_PASSWORD_AUTH**
- ✅ **ALLOW_REFRESH_TOKEN_AUTH**

**Note User Pool and Client IDs** for later use:
- **User Pool ID**: `us-east-1_XXXXXXXXX`
- **App Client ID**: `xxxxxxxxxxxxxxxxxxxxxxxxxx`

---

## Phase 2: IAM Roles for Customer Authorization

### Step 2.1: Create Premium User Role

**IAM → Roles → Create role**

**Trust policy configuration**:
- **Trusted entity**: Web identity
- **Identity provider**: Cognito
- **Audience**: (Leave blank - will update after Identity Pool creation)
- **Role name**: `CognitoPremiumUserRole`

**Permissions policy** (inline policy):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/${cognito-identity.amazonaws.com:sub}/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::BUCKET_NAME",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "${cognito-identity.amazonaws.com:sub}/*"
                }
            }
        }
    ]
}
```

### Step 2.2: Create Basic User Role

**IAM → Roles → Create role**

**Same trust policy setup** as premium role.

**Role name**: `CognitoBasicUserRole`

**Permissions policy** (read-only):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKET_NAME/${cognito-identity.amazonaws.com:sub}/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::BUCKET_NAME",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "${cognito-identity.amazonaws.com:sub}/*"
                }
            }
        }
    ]
}
```

---

## Phase 3: Identity Pool Setup (AWS Authorization)

### Step 3.1: Create Identity Pool

**Amazon Cognito → Identity pools → Create identity pool**

**Configure identity pool trust**:
- **Identity pool name**: `CustomerFileAccessIdentityPool`
- **Authenticated access**: ✅ Grant access
- **Unauthenticated access**: ❌ Deny access

**Connect identity providers**:
- **Identity provider**: Cognito user pool
- **User pool ID**: `us-east-1_XXXXXXXXX` (from Phase 1)
- **App client ID**: `xxxxxxxxxxxxxxxxxxxxxxxxxx` (from Phase 1)

**Configure permissions**:
- **Authenticated role**: `CognitoBasicUserRole` (default fallback)

### Step 3.2: Configure Group-to-Role Mapping

**Identity pools → CustomerFileAccessIdentityPool → User access tab**

**Identity providers → Edit Cognito User Pool provider**:
- **Role selection**: Choose role with rules
- **Claims for role mapping**: 

**Rule 1 - Premium Users**:
- **Claim**: `cognito:groups`
- **Operator**: Equals
- **Value**: `premium-users`
- **Role**: `CognitoPremiumUserRole`

**Rule 2 - Basic Users**:
- **Claim**: `cognito:groups`
- **Operator**: Equals
- **Value**: `basic-users`
- **Role**: `CognitoBasicUserRole`

**Role resolution**: Use default authenticated role (for users without group assignment)

### Step 3.3: Update IAM Trust Policies

**Copy Identity Pool ID** from overview page: `us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

**Update both IAM roles** with corrected trust policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "cognito-identity.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "cognito-identity.amazonaws.com:aud": "IDENTITY_POOL_ID"
                },
                "ForAnyValue:StringLike": {
                    "cognito-identity.amazonaws.com:amr": "authenticated"
                }
            }
        }
    ]
}
```

---

## Phase 4: S3 Bucket and Testing

### Step 4.1: Create Customer Data Bucket

```bash
# Create bucket with unique name
aws s3 mb s3://customer-files-cognito-lab-$(date +%s)
```

**Note the exact bucket name** for policy updates.

### Step 4.2: Update IAM Policies with Bucket Name

**Replace `BUCKET_NAME`** in both IAM role policies with actual bucket name.

### Step 4.3: Set Permanent User Passwords

```bash
# Set permanent passwords for testing
aws cognito-idp admin-set-user-password \
    --user-pool-id us-east-1_XXXXXXXXX \
    --username alice-premium \
    --password NewPass123! \
    --permanent

aws cognito-idp admin-set-user-password \
    --user-pool-id us-east-1_XXXXXXXXX \
    --username bob-basic \
    --password NewPass123! \
    --permanent
```

---

## Phase 5: End-to-End Testing

### Step 5.1: Handle Client Secret (If Required)

**If authentication fails with SECRET_HASH error**, calculate hash:

```python
import hmac
import hashlib
import base64

def get_secret_hash(username, client_id, client_secret):
    message = username + client_id
    secret_hash = base64.b64encode(
        hmac.new(
            client_secret.encode(),
            message.encode(),
            digestmod=hashlib.sha256
        ).digest()
    ).decode()
    return secret_hash

# Calculate for actual usernames (UUIDs from console)
username = "e4b8c4f8-f081-70f9-c7b8-0d36cde85647"  # Alice's UUID
client_id = "11us8ekienpdlrd1um01fkgg7n"
client_secret = "your-client-secret-here"

print(get_secret_hash(username, client_id, client_secret))
```

### Step 5.2: Test Authentication Flow

```bash
# Authenticate premium user
aws cognito-idp admin-initiate-auth \
    --user-pool-id us-east-1_XXXXXXXXX \
    --client-id YYYYYYYYYYYYYYYYYYYY \
    --auth-flow ADMIN_NO_SRP_AUTH \
    --auth-parameters 'USERNAME=alice-premium-uuid,PASSWORD=NewPass123!,SECRET_HASH=calculated-hash'
```

**Copy ID Token** from response.

### Step 5.3: Get AWS Credentials

```bash
# Get Identity ID
aws cognito-identity get-id \
    --identity-pool-id us-east-1:ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ \
    --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX=ID_TOKEN

# Get temporary AWS credentials
aws cognito-identity get-credentials-for-identity \
    --identity-id us-east-1:IDENTITY-ID-FROM-PREVIOUS-STEP \
    --logins cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX=ID_TOKEN
```

### Step 5.4: Test S3 Access

```bash
# Export temporary credentials
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Verify assumed role
aws sts get-caller-identity

# Test premium user permissions
echo "Premium user test file" | aws s3 cp - s3://bucket-name/IDENTITY-ID/test-file.txt

# List personal folder
aws s3 ls s3://bucket-name/IDENTITY-ID/

# Test isolation (should fail)
aws s3 ls s3://bucket-name/different-identity-id/
```

---

## Troubleshooting Guide

### Common Issues

**SECRET_HASH Errors**:
- **Cause**: App client has secret enabled but hash not provided
- **Solution**: Calculate SECRET_HASH or disable client secret
- **Verification**: Check app client configuration in console

**Authentication Flow Errors**:
- **Cause**: Required auth flows not enabled on app client
- **Solution**: Enable ALLOW_ADMIN_USER_PASSWORD_AUTH and ALLOW_USER_PASSWORD_AUTH
- **Verification**: Check app client authentication flows

**Role Mapping Failures**:
- **Cause**: Trust policy audience mismatch or rule configuration errors
- **Solution**: Verify Identity Pool ID in trust policies and rule syntax
- **Verification**: Check IAM role trust relationships and Identity Pool rules

**S3 Access Denied**:
- **Cause**: Bucket name mismatch in policies or incorrect resource ARNs
- **Solution**: Update IAM policies with exact bucket name
- **Verification**: Compare policy ARNs with actual bucket name

### Validation Commands

```bash
# Check User Pool configuration
aws cognito-idp describe-user-pool --user-pool-id us-east-1_XXXXXXXXX

# Verify group membership
aws cognito-idp admin-list-groups-for-user \
    --user-pool-id us-east-1_XXXXXXXXX \
    --username alice-premium

# Check IAM role trust policy
aws iam get-role --role-name CognitoPremiumUserRole

# Verify Identity Pool configuration
aws cognito-identity describe-identity-pool \
    --identity-pool-id us-east-1:ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ
```

---

## Security Best Practices

### Authentication Security
- **Password policies** enforced at User Pool level
- **Account lockout** mechanisms for brute force protection
- **Email verification** required for account activation
- **Secure app client** configuration without unnecessary permissions

### Authorization Security  
- **Principle of least privilege** with role-specific permissions
- **Customer data isolation** using identity-based resource policies
- **Temporary credentials** with limited lifetime (1 hour default)
- **Cross-customer access prevention** through IAM policy conditions

### Operational Security
- **Token validation** in application code before trusting claims
- **Session management** with proper token refresh handling
- **Audit logging** through CloudTrail for authentication events
- **Monitoring** for suspicious access patterns and failed authentication attempts

---

## Success Validation

✅ **User Pool created** with groups and test users configured  
✅ **Identity Pool established** with proper group-to-role mapping  
✅ **IAM roles configured** with customer data isolation policies  
✅ **S3 bucket created** and integrated with identity-based access  
✅ **End-to-end flow tested** from authentication through data access  
✅ **Premium vs basic permissions** validated with different user types  
✅ **Customer isolation confirmed** by testing cross-customer access denial  

This implementation provides a production-ready foundation for customer identity management with Amazon Cognito, demonstrating enterprise-grade security patterns suitable for real-world applications.
