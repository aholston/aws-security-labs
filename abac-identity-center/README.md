# Attribute-Based Access Control (ABAC) with IAM Identity Center

This lab implements enterprise-grade attribute-based access control using AWS IAM Identity Center session tags and EC2 resource tags. It demonstrates dynamic permissions that automatically adapt based on user attributes and resource characteristics, representing the most advanced IAM concepts tested in the AWS Certified Security – Specialty exam.

The lab emphasizes:
- Session tag propagation from IAM Identity Center to AWS sessions
- Dynamic resource access control using attribute matching
- Progressive permission models with role and environment restrictions
- Real-world organizational scaling patterns

---

## Lab Architecture

```
devuser (Department: Engineering, CostCenter: CC-1001, Title: Developer)
    ↓ (Identity Center session tags)
DevAccessProd Permission Set (ABAC policies)
    ↓ (dynamic access based on tag matching)
EC2 Instances (tagged with Department, CostCenter, Environment, Project)
    ↓ (actions allowed/denied based on attribute alignment)
Organizational Access Control (automatic scaling with new departments/users)
```

## Prerequisites

This lab builds upon the foundational IAM Identity Center federation setup:
- AWS Organization with management, security-account, prod-account
- IAM Identity Center enabled at organization level with internal directory
- Federated user `devuser` configured in `Developers` group
- Basic permission set `DevAccessProd` assigned to prod-account

---

## Phase 1: Identity Center ABAC Configuration

### 🎯 Learning Objectives
- Configure attribute mapping from identity source to session tags
- Understand session tag propagation through federated authentication
- Master the relationship between user attributes and AWS permissions

### ⚡ Key Exam Topics Covered
- Attributes for access control configuration (heavily tested)
- Session tag mechanics with IAM Identity Center
- Identity source attribute mapping and troubleshooting

---

## Implementation Summary

### Step 1: Enable Attributes for Access Control
- **Location**: Management account → IAM Identity Center → Settings
- **Configuration**: Attributes for access control → Manage attributes
- **Mappings**: User attributes to session tag keys

### Step 2: User Attribute Configuration
- **User**: devuser configured with Department, CostCenter, Title
- **Values**: Engineering, CC-1001, Developer
- **Source**: IAM Identity Center internal directory

### Step 3: EC2 Resource Creation and Tagging
- **Engineering Development**: Department=Engineering, Environment=Development
- **Engineering Production**: Department=Engineering, Environment=Production  
- **Finance Production**: Department=Finance, CostCenter=CC-2002

### Step 4: ABAC Policy Implementation
- **Discovery permissions**: Unconditional describe operations
- **Department isolation**: Access based on Department + CostCenter matching
- **Role-based restrictions**: Developer write access limited to Development
- **Explicit deny**: Production termination blocked for developers

---

## ABAC Policy Architecture

### Progressive Permission Model

**Level 1: Discovery (Unconditional)**
```json
"Action": [
  "ec2:DescribeInstances",
  "ec2:DescribeInstanceStatus"
],
"Resource": "*"
```

**Level 2: Department Management (Department + CostCenter)**
```json
"Condition": {
  "StringEquals": {
    "ec2:ResourceTag/Department": "${aws:PrincipalTag/Department}",
    "ec2:ResourceTag/CostCenter": "${aws:PrincipalTag/CostCenter}"
  }
}
```

**Level 3: Role + Environment Restrictions (Developer + Development)**
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalTag/Title": "Developer",
    "ec2:ResourceTag/Environment": "Development"
  }
}
```

**Level 4: Explicit Protection (Production Denial)**
```json
"Effect": "Deny",
"Condition": {
  "StringEquals": {
    "aws:PrincipalTag/Title": "Developer",
    "ec2:ResourceTag/Environment": "Production"
  }
}
```

---

## Validation Results

### Department-Based Access Control

| Resource | User Access | Expected Result | Actual Result |
|----------|-------------|----------------|---------------|
| Engineering Dev Instance | Department + CostCenter Match | ✅ Full Access | ✅ Confirmed |
| Engineering Prod Instance | Department + CostCenter Match | ✅ Read/Manage, ❌ Terminate | ✅ Confirmed |
| Finance Prod Instance | Department + CostCenter Mismatch | ❌ Access Denied | ✅ Confirmed |

### Role and Environment Restrictions

| Action | Engineering Dev | Engineering Prod | Finance Prod | Validation |
|--------|----------------|------------------|--------------|------------|
| View Details | ✅ | ✅ | ✅ | ✅ Discovery permissions work |
| Start/Stop | ✅ | ✅ | ❌ | ✅ Department isolation works |
| Terminate | ✅ | ❌ | ❌ | ✅ Environment + role restrictions work |

---

## Key ABAC Concepts Mastered

### 1. Session Tag Flow
- **Identity Center attributes** → **Session tags** → **Policy variables**
- **Attribute mapping**: `${path:enterprise.department}` → `aws:PrincipalTag/Department`
- **Dynamic evaluation**: Real-time attribute comparison during API calls

### 2. Resource Tag Matching
- **EC2 condition keys**: `ec2:ResourceTag/Department` references actual instance tags
- **Session tag variables**: `${aws:PrincipalTag/Department}` references user session
- **Exact matching**: Case-sensitive string comparison for access decisions

### 3. Progressive Access Control
- **Layered permissions**: Discovery → Department → Role → Environment
- **Explicit deny precedence**: Deny statements override any allow
- **Organizational scaling**: New departments/users work without policy changes

### 4. Enterprise Access Patterns
- **Cost center isolation**: Users can only affect their budget allocations
- **Environment protection**: Production resources require elevated permissions
- **Role-based restrictions**: Different capabilities based on job function

---

## Exam Preparation Value

### Frequently Tested Scenarios
- "How do you implement department-based resource isolation?"
- "What's the difference between static roles and dynamic ABAC?"
- "How do session tags propagate through role assumption?"
- "Why might ABAC access be denied despite correct user attributes?"

### Common Troubleshooting Patterns
- **Session tags not flowing**: Attributes for access control not enabled
- **Access denied with matching tags**: Case sensitivity or typos in tag values
- **Partial access working**: Missing conditions or explicit deny conflicts
- **Policy syntax errors**: Invalid condition keys for specific services

### Real-World Implementation
- **Automatic scaling**: New organizational units work without policy updates
- **Compliance alignment**: Clear audit trail of attribute-based decisions
- **Operational efficiency**: Reduced IAM administrative overhead
- **Security consistency**: Uniform access patterns across AWS services

---

## Advanced Extensions

### Multi-Service ABAC
- **S3 limitations**: Bucket tags not supported in conditions (object tags only)
- **RDS support**: Database instance tag-based access control
- **Lambda integration**: Function tag matching for execution permissions

### Attribute Sources
- **External IdP integration**: SAML assertion attributes from Active Directory
- **Custom attributes**: Additional metadata for fine-grained control
- **Dynamic attributes**: Time-based, location-based access restrictions

### Policy Sophistication
- **Conditional logic**: ForAllValues, ForAnyValue operators
- **Complex conditions**: Multiple attribute combinations
- **Emergency access**: Break-glass procedures with elevated permissions

---

## Success Criteria

✅ **Session tags configured** and flowing from Identity Center to AWS sessions  
✅ **Department isolation** implemented with automatic organizational scaling  
✅ **Role-based restrictions** enforcing job function limitations  
✅ **Environment protection** preventing unauthorized production access  
✅ **Explicit deny precedence** demonstrated with production termination blocks  
✅ **Real-world access patterns** validating enterprise-grade ABAC implementation  

This ABAC implementation provides hands-on experience with the most sophisticated IAM concepts in AWS, directly preparing for Security Specialty exam Domain 4 scenarios while demonstrating production-ready access control architecture.