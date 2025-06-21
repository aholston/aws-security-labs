# Advanced Cross-Account Role Chaining with External ID

This lab extends the foundational IAM Identity Center federation lab by implementing a sophisticated 3-tier cross-account role assumption chain. It demonstrates enterprise-grade access patterns and directly targets Domain 4 (Identity and Access Management) scenarios from the AWS Certified Security – Specialty exam.

The lab emphasizes:
- Multi-hop role assumption chains across AWS accounts
- External ID implementation for third-party access security
- Conditional trust policies and federated user identification
- Real-world privilege escalation patterns with audit trails

---

## Lab Architecture

```
devuser (IAM Identity Center)
    ↓ (federated authentication)
DevAccessProd Permission Set (prod-account)
    ↓ (assumes with userid condition)
CrossAccountIntermediateRole (prod-account)
    ↓ (assumes with External ID)
LogAnalystRole (security-account)
    ↓ (accesses)
CloudTrail S3 Bucket + EventBridge Rules
```

## Prerequisites

This lab builds upon the existing IAM Identity Center federation setup:
- AWS Organization with management, security-account, prod-account
- IAM Identity Center enabled at organization level
- Federated user `devuser` with `DevAccessProd` permission set
- CloudTrail logging to security-account S3 bucket

---

## Phase 1: Cross-Account Role Chain Implementation

### 🎯 Learning Objectives
- Understand federated user identification in trust policies
- Implement External ID for confused deputy attack prevention
- Master multi-hop role assumption patterns
- Practice cross-account security resource access

### ⚡ Key Exam Topics Covered
- Cross-account role assumption (heavily tested in Domain 4)
- Trust policy conditions and troubleshooting
- External ID security patterns
- STS service mechanics and token flows

---

## Implementation Summary

### Step 1: Intermediate Role Creation
- **Account**: prod-account
- **Role**: `CrossAccountIntermediateRole`
- **Trust Policy**: Federated user condition with `aws:userid`
- **Permissions**: STS AssumeRole for target role

### Step 2: Target Role Creation  
- **Account**: security-account
- **Role**: `LogAnalystRole`
- **Trust Policy**: Cross-account with External ID requirement
- **Permissions**: CloudTrail S3 and EventBridge access

### Step 3: Permission Set Update
- **Component**: `DevAccessProd` permission set
- **Addition**: STS AssumeRole for intermediate role
- **Security**: Least privilege access to specific role ARN

### Step 4: End-to-End Testing
- Federated authentication via Identity Center portal
- CLI role assumption chain validation
- Security resource access verification

---

## Security Design Principles

### Defense in Depth
Each role assumption layer enforces additional security controls:
- **Layer 1**: Federated authentication with Identity Center
- **Layer 2**: Specific user identification via `aws:userid` condition
- **Layer 3**: External ID requirement preventing confused deputy attacks

### Audit Trail Completeness
CloudTrail captures separate `AssumeRole` events for each hop:
1. `AssumeRoleWithSAML` - Initial federated login
2. `AssumeRole` - Intermediate role assumption
3. `AssumeRole` - Target role assumption with External ID

### Session Duration Control
Progressive session timeout enforcement:
- Identity Center session: 8 hours (configurable)
- Intermediate role: 1 hour (default)
- Target role: 1 hour (can be shortened for sensitive access)

---

## Key Implementation Details

### Federated User Identification
```json
"Condition": {
  "StringLike": {
    "aws:userid": "AROA*:devuser"
  }
}
```

### External ID Security
```json
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "unique-external-id-12345"
  }
}
```

### Cross-Account Resource ARN
```
arn:aws:iam::SECURITY-ACCOUNT-ID:role/LogAnalystRole
```

---

## Exam Preparation Value

### Common Exam Scenarios Addressed
- "A federated user needs access to resources in another account..."
- "How do you prevent the confused deputy attack in cross-account access?"
- "What's the most secure way to implement vendor access to your AWS resources?"
- "Why is role assumption failing between accounts?"

### Troubleshooting Skills Developed
- Trust policy condition syntax for federated users
- External ID requirement validation
- Cross-account permission boundary analysis
- STS error message interpretation

---

## Next Phase Opportunities

### Advanced Conditional Access
- MFA requirements for role assumption
- Time-based access windows
- Source IP restrictions
- Geographic limitations

### Break and Fix Scenarios
- Intentionally misconfigured trust policies
- Missing External ID conditions
- Circular trust relationships
- Expired session handling

### Automation and Monitoring
- EventBridge detection of role assumption chains
- CloudWatch metrics for access patterns
- Automated External ID rotation
- Compliance reporting automation

---

## Success Metrics

✅ Successfully completed 3-tier role assumption chain  
✅ Verified External ID security requirement  
✅ Accessed cross-account security resources  
✅ Documented complete audit trail in CloudTrail  
✅ Demonstrated federated user trust policy conditions  

This lab provides hands-on experience with the most complex cross-account access patterns tested in the AWS Security Specialty exam while building production-ready security controls.