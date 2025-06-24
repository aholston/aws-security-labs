# Permission Boundaries with IAM Identity Center

This lab implements enterprise-grade permission boundaries using AWS IAM Identity Center to demonstrate secure delegated administration patterns. It focuses on the most advanced and frequently tested permission boundary concepts in Domain 4 (Identity and Access Management) of the AWS Certified Security – Specialty exam.

The lab emphasizes:
- Permission boundary policy evaluation precedence and intersection logic
- Cross-account policy management patterns with Identity Center
- Real-world enterprise access control with security guardrails
- Systematic testing methodology for complex policy scenarios

---

## Lab Architecture

```
boundaryuser (IAM Identity Center)
    ↓ (federated authentication)
SimpleBoundedDeveloper Permission Set (security-account)
    ↓ (cross-account provisioning)
Bounded Role in prod-account
    ↓ (permission boundary enforcement)
SimpleBoundedPermissionBoundary Policy
    ↓ (restricts maximum permissions)
Allowed: EC2 t2.micro only, S3 dev-* buckets, describe operations
Denied: IAM operations, large instances, production resources
```

## Prerequisites

This lab builds upon the existing multi-account AWS environment:
- AWS Organization with management, security-account, prod-account
- IAM Identity Center enabled with delegated administration to security-account
- Existing federated users and groups (`Developers` group)
- CloudTrail logging for comprehensive audit trails

---

## Phase 1: Permission Boundary Policy Implementation

### 🎯 Learning Objectives
- Understand permission boundaries as security ceilings that restrict maximum permissions
- Master policy evaluation logic: effective permissions = identity policy ∩ boundary policy
- Implement cross-account policy management patterns with Identity Center

### ⚡ Key Exam Topics Covered
- Permission boundary implementation and troubleshooting (heavily tested)
- Policy evaluation precedence with multiple policy types
- Cross-account federated access patterns
- Enterprise delegated administration with security controls

---

## Implementation Summary

### Step 1: Cross-Account Policy Architecture
- **Challenge Discovered**: Identity Center cannot reference permission boundary policies across accounts via ARN
- **Solution Implemented**: Policy name matching pattern - identical policies in both accounts
- **Boundary Policy**: Created `SimpleBoundedPermissionBoundary` in both security-account and prod-account

### Step 2: Federated User Creation
- **User**: `boundaryuser` created via Identity Center with organizational attributes
- **Group Assignment**: Added to existing `Developers` group for access patterns
- **Identity Source**: Internal Identity Center directory

### Step 3: Permission Set with Broad Identity Policy
- **Permission Set**: `SimpleBoundedDeveloper` with intentionally generous permissions
- **Identity Policy**: Grants `ec2:*`, `s3:*`, and limited IAM permissions
- **Boundary Application**: Applied via policy name reference to enforce restrictions

### Step 4: Systematic Testing and Validation
- **Test Scenarios**: 4 comprehensive scenarios testing policy intersection logic
- **Results**: All scenarios validated theoretical understanding through real AWS API responses
- **Learning**: Boundaries successfully restricted broad identity policy permissions

---

## Policy Architecture Details

### Permission Boundary Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBasicEC2Operations",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus", 
        "ec2:RunInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:InstanceType": ["t2.micro"]
        }
      }
    },
    {
      "Sid": "AllowBasicS3Operations",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::dev-*",
        "arn:aws:s3:::dev-*/*"
      ]
    },
    {
      "Sid": "DenyIAMOperations",
      "Effect": "Deny",
      "Action": ["iam:*"],
      "Resource": "*"
    }
  ]
}
```

### Identity Policy (Broad Permissions)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": "*"
    },
    {
      "Effect": "Allow", 
      "Action": ["s3:*"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:ListRoles",
        "iam:GetRole",
        "iam:CreateRole"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Validation Results

### Policy Evaluation Testing Matrix

| Test Scenario | Identity Policy | Boundary Policy | Expected Result | Actual Result |
|---------------|----------------|-----------------|-----------------|---------------|
| `aws iam create-role` | ✅ Allow (`iam:CreateRole`) | ❌ Deny (`iam:*`) | ❌ AccessDenied | ✅ Confirmed |
| `aws s3 ls s3://production-data` | ✅ Allow (`s3:*`) | ❌ No matching resource | ❌ AccessDenied | ✅ Confirmed |
| `aws ec2 run-instances --instance-type t2.micro` | ✅ Allow (`ec2:*`) | ✅ Allow (condition met) | ✅ Success | ✅ Confirmed |
| `aws ec2 run-instances --instance-type m5.large` | ✅ Allow (`ec2:*`) | ❌ Deny (condition failed) | ❌ AccessDenied | ✅ Confirmed |

### Key Architectural Discovery

**Cross-Account Permission Boundary Pattern:**
- Identity Center provisions roles in target accounts (prod-account)
- Permission boundary policies only need to exist in the target account where roles are created
- Identity Center references policies by name string during provisioning
- Provisioning fails fast if referenced boundary policy doesn't exist in target account
- Policy enforcement happens entirely within the target account during API calls
- Role naming follows pattern: `AWSReservedSSO_[PermissionSetName]_[UniqueRandomString]`

---

## Security Design Principles

### Policy Intersection Logic
The lab demonstrates the fundamental security principle:
**Effective Permissions = Identity Policy ∩ Permission Boundary Policy**

### Defense in Depth
Multiple security layers implemented:
- **Identity Center authentication**: Federated access with organizational attributes
- **Permission boundary restrictions**: Maximum permission ceiling regardless of identity grants
- **Explicit deny statements**: Absolute restrictions that cannot be overridden
- **Resource-level controls**: Granular access based on naming patterns and resource types

### Enterprise Scalability
- **Policy name matching**: Enables consistent cross-account policy management
- **Attribute-based compatibility**: Works alongside existing ABAC implementations
- **Organizational structure**: Aligns with delegated administration patterns

---

## Exam Preparation Value

### Frequently Tested Scenarios Mastered
- "User has broad permissions but gets AccessDenied - troubleshoot the issue"
- "How do you implement secure delegated administration in large organizations?"
- "What's the difference between permission boundaries and SCPs?"
- "Why might cross-account federated access fail with permission boundaries?"

### Common Troubleshooting Patterns
- **Provisioning failures**: Verify all referenced policies exist in target account before assignment
- **Policy evaluation order**: Understanding precedence of different policy types
- **Role naming patterns**: Identity Center uses predictable naming with random suffixes
- **Policy lifecycle management**: Updates must happen in target account, not Identity Center account
- **Error message interpretation**: AWS provides clear policy existence validation during provisioning

### Real-World Implementation Skills
- **Enterprise access control**: Balancing developer autonomy with security constraints
- **Cross-account architecture**: Managing policies in federated multi-account environments
- **Systematic testing**: Validating complex policy logic through methodical verification
- **Operational excellence**: Designing maintainable and scalable permission models

---

## Advanced Extensions

### Additional Boundary Scenarios
- **Graduated permissions**: Different boundary policies for junior vs senior developers
- **Project-based restrictions**: Cost center and resource tagging enforcement
- **Time-based access**: Conditional boundaries with temporal restrictions

### Break-Glass Emergency Access
- **Time-limited elevation**: Temporary administrative access outside normal boundaries
- **Automated monitoring**: EventBridge and Lambda integration for emergency access detection
- **Compliance integration**: Audit trails and automated revocation patterns

### Multi-Policy Evaluation
- **Service Control Policies**: Organization-level restrictions combined with boundaries
- **Resource-based policies**: Complex interactions between identity, boundary, and resource policies
- **Session policies**: Additional restrictions for temporary credentials

---

## Success Criteria

✅ **Permission boundary successfully restricts developer access** despite broad identity policy permissions  
✅ **Cross-account policy management mastered** - discovered and solved ARN reference limitations  
✅ **Policy name matching pattern implemented** for Identity Center cross-account provisioning  
✅ **Policy evaluation precedence validated** through systematic testing of all 4 scenarios  
✅ **Real-world enterprise access patterns demonstrated** with federated users and boundaries  
✅ **Critical exam concepts mastered** through hands-on validation and troubleshooting  

This permission boundary implementation provides comprehensive hands-on experience with the most sophisticated IAM concepts tested in Domain 4, demonstrating production-ready enterprise security patterns while building deep understanding of AWS policy evaluation logic.